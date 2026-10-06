# Checkout Failures from a Missed Cron Job Run Without a Heartbeat

Short answer: when a scheduled cron job has a missed run and no heartbeat, alert on the checkout task's expired completion deadline, not on every scheduler event. Treat each expected execution as a named obligation, accept one terminal outcome, and page only when neither success nor an explicit failure arrives before the deadline. That monitoring rule catches a skipped launch as well as a crashed worker while keeping routine telemetry out of the incident channel.

The key decision is semantic: the monitor should answer "did the required checkout work finish for this region and window?" A pulse that says a process existed cannot answer it. In a B2B SaaS checkout flow, duplicate alerts are more than an annoyance; they train responders to skim the channel where a real payment or entitlement failure needs attention.

## Decision record and invariants

The chosen design is an obligation ledger plus terminal events. Before an EU or US execution window opens, a control-plane process records an expected run with a stable key such as `(job, region, scheduled_window)`. The worker later records exactly one successful or failed terminal state. A separate evaluator alerts after the obligation's deadline if no terminal state exists.

This establishes four invariants. Each business window has one stable identity. Retries reuse that identity instead of creating new expected runs. Success means the checkout reconciliation reached its defined commit point, rather than merely starting. An explicit failure and an expired obligation enter the same incident pipeline but retain different reasons. Keep the failure boundaries visible. The regional scheduler may fail before launching a worker. The worker may start and stop before reporting. Event delivery may be delayed or repeated. The evaluator may run twice. None of those boundaries should produce two pages for one checkout obligation, so the incident key must derive from the stable run key, not from an event identifier.

One obligation. One incident.

This is a signal-quality decision. It does not require every telemetry record to be durable forever, but the expected obligation and its terminal state must survive the exact components whose health they judge. Putting the evaluator only inside the scheduled worker defeats that boundary.

## How should monitoring detect a missed cron job run without a heartbeat?

A missed execution is an expected regional obligation that has no accepted terminal outcome after its deadline. It is not simply an absent periodic ping. Define the deadline from the scheduled window, the maximum legitimate start delay, and the job's allowed completion time. Those values are policy inputs, so keep them beside the job definition and review them when workload behavior changes.

Late success needs an explicit rule. I prefer to preserve the incident as "late, then succeeded" rather than erase it, because erasure hides a service-level miss. The notification layer can resolve the page once success arrives, while the ledger retains both the deadline breach and eventual outcome.

Short delay. Long memory.

Clock and region handling are easy places to create noise. Persist window boundaries as absolute instants and keep the business timezone as metadata. Generate the run key from the declared window, not from a worker's local clock. For EU and US execution, region is part of the identity; otherwise one region's success can accidentally satisfy another region's obligation.

## Options compared by signal quality

| Option | Detects skipped launch | Confirms business completion | Typical noise risk | Useful boundary |
|---|---:|---:|---|---|
| Process pulse | Sometimes | No | Pulses continue while work is stuck | Long-lived worker liveness |
| Failure-only exception alert | No | No | Retries can emit repeated failures | Fast diagnosis after code throws |
| External deadline check | Yes | Only if completion is modeled | Late delivery can look missed | Simple, independent supervision |
| Obligation plus terminal outcome | Yes | Yes | Bad deadlines create premature alerts | Business-critical scheduled work |

The last option carries more state, and that is the price of better semantics. The table also explains why an exception alert remains useful: it supplies diagnostic evidence quickly, but it cannot prove that an execution was ever attempted. Pairing it with deadline evaluation closes that blind spot without paging on healthy starts. I would keep raw starts, attempts, and retry errors searchable, then route only state transitions into the alert channel. This resembles deliverability work: a provider accepting a message is not the same event as the recipient getting the OTP. Likewise, a scheduler accepting a trigger is not checkout completion. Compliance reviews also benefit from preserving that distinction because operators can explain what was expected, what occurred, and when the monitoring decision changed. NIST SP 800-66r2 is a useful control-oriented reference when the checkout system handles regulated health information; it frames implementation as risk management rather than prescribing this particular design.

## Critical path in code

The evaluator can be small. Storage must enforce uniqueness for the run key and perform compare-and-set transitions atomically; the Python below shows the decision logic around that interface, not a database implementation.

```python
from dataclasses import dataclass
from datetime import datetime
from enum import Enum


class State(str, Enum):
    EXPECTED = "expected"
    SUCCEEDED = "succeeded"
    FAILED = "failed"
    EXPIRED = "expired"


@dataclass(frozen=True)
class Obligation:
    run_key: str
    deadline: datetime
    state: State


def evaluate(obligation: Obligation, now: datetime, store, incidents) -> None:
    if obligation.state is not State.EXPECTED or now < obligation.deadline:
        return

    # Only one evaluator may win this transition.
    expired = store.compare_and_set(
        obligation.run_key,
        expected=State.EXPECTED,
        replacement=State.EXPIRED,
    )
    if not expired:
        return

    incidents.open_once(
        key=f"scheduled-checkout:{obligation.run_key}",
        reason="completion deadline expired",
    )
```

The worker should write `SUCCEEDED` only after the defined checkout commit point. On a known terminal error, it writes `FAILED` and opens the same incident key with a different reason. A retry reads the existing obligation and attempts a legal state transition; it does not mint another key. This makes duplicate event delivery boring, which is exactly what an on-call path needs.

Deployment deserves a failure test, not just a unit test. In a staging environment, create an obligation and suppress worker launch; verify one expiry incident. Then deliver the success event twice and verify one state transition. Delay success until after expiry and verify that the history records both facts. Finally, run two evaluators concurrently. These tests target boundaries, not framework syntax.

For dashboards, aggregate outcomes without hiding tails. Core Web Vitals uses a 75th-percentile assessment to represent user experience while retaining a defined threshold model; that is a useful reminder that an aggregate needs an explicit population and decision rule. Scheduled checkout monitoring needs its own domain-specific measures, such as outcome counts by region and lateness distribution, rather than borrowing web-performance thresholds.

## Rejected option and where it still fits

The rejected design is a single heartbeat emitted at task start. It is attractive because it is nearly stateless and proves the scheduler reached the worker. It fails this checkout requirement: a worker can emit the heartbeat and stall before the reconciliation commit, while a scheduler that never launches emits nothing without leaving a durable record of what was expected.

The chosen design has a real limitation: it is not suitable when no independent component can create durable obligations before execution. In that environment, use an external deadline check around a start pulse, accept that it proves launch rather than completion, and keep the job low impact. The obligation ledger also adds storage, clock policy, retention decisions, and a state reconciler. A team that cannot operate those parts should choose the simpler pulse until the business consequence justifies the extra machinery.

A start heartbeat still has a valid use case. For low-impact housekeeping where launch itself is the only meaningful contract, an external deadline around that pulse may be enough. It can also remain as diagnostic telemetry in the richer design. Do not promote it to business completion.

The operational rule is therefore narrow: page from expired or explicitly failed obligations, group by stable regional run key, and keep attempts as context. Review deadlines from observed distributions, but require an intentional policy change before altering alert semantics. This produces fewer notifications because fewer events qualify, not because failures are filtered after the fact.

## References

- https://web.dev/articles/vitals
- https://csrc.nist.gov/pubs/sp/800/66/r2/final
