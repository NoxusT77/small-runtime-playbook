# Raising a Launch-Day API Spend Cap — Credential Alerts and Rollback Boundaries

TL;DR: On launch day, raise the API spend cap for the developer-tool workload, keep a lower alert threshold, and bind both controls to one dedicated credential with an automatic expiry. The deciding constraint is blast radius: if that credential leaks, loops, or receives unexpected traffic, its maximum authorized spend must still be tolerable before the invoice arrives.

This is an architecture decision record for a growth spike, not a request to disable cost controls. The change should be reversible without rotating a shared secret, and an operator should be able to tell whether the workload is healthy before allowing more exposure. Treat the alert as an early signal, the ceiling as containment, and rollback as a tested state transition. They solve different problems.

## How should we raise the launch-day API spend cap?

Four invariants define the decision. First, the launch workload uses a credential that no unrelated workload uses. Second, the hard ceiling always remains finite. Third, crossing the warning threshold does not silently raise that ceiling. Fourth, the temporary policy has an owner, an expiry time, and a known prior version.

The first invariant matters most. A budget attached to an account can limit the invoice but cannot identify which workload consumed it. A credential scoped to one launch worker makes the failure boundary legible: revoking it stops that worker without also stopping documentation builds, internal automation, or other customers' traffic. OWASP's secrets guidance recommends limiting scope and access, automating rotation where possible, and maintaining audit information around secret use.

The operating numbers should come from exposure the team is willing to accept, not from a hoped-for traffic curve. For a concrete decision record, suppose the normal daily ceiling is 400 budget units, the launch ceiling is 1,200, and the first alert stays at 600. Those are example policy units, not vendor prices. The 600-unit gap between warning and containment is deliberate: it gives humans time to inspect request rate, error distribution, and business events while the hard stop still bounds damage.

No infinite mode.

That is the first trade-off: more headroom buys continuity, while a finite stop preserves containment.

Before approval, record the credential identifier, policy version, change owner, start and expiry timestamps, alert recipients, and the previous ceiling. Do not place the secret value in the record, logs, tickets, or alert payloads. The record needs enough identity to investigate and revoke; it does not need secret material.

## Failure boundaries and alert behavior

A spend warning and a hard ceiling belong on separate paths. The warning path may be delayed, deduplicated, or temporarily unavailable. The enforcement decision therefore cannot depend on a chat message, mailbox, or human acknowledgment. It must read authoritative usage and policy state, reject work beyond the ceiling, and emit an audit event.

Delivery deserves suspicion. Anyone who has operated OTP or transactional messaging knows that an accepted notification is not the same as a read notification. Spam filtering, provider throttling, stale routing rules, and a sleeping on-call device can all erase the practical value of a single channel. Send the same threshold event to the incident system and the budget owner, deduplicate it by credential and policy version, and escalate when it remains unacknowledged. The ceiling still stands if every notification disappears.

Use at least two signals when deciding whether to continue the temporary allowance: budget consumption rate and useful work completed. A request counter alone cannot distinguish customer adoption from a retry loop. Compare consumption with successful jobs, active tenants, queue age, and error classes already available in the workload's telemetry. A sharp rise in spend with flat successful work is a rollback signal even when the absolute ceiling is distant.

The containment response should be explicit. Requests at or above the hard ceiling fail closed for that credential, with a stable machine-readable reason; unrelated credentials continue. Alert processing must never log authorization headers or secret values. Rotation remains available for suspected disclosure, but ordinary rollback should restore the prior policy rather than force emergency secret distribution.

Test the pager.

## Choosing the control boundary

The following options are not vendor rankings. They compare where the enforceable boundary sits.

| Boundary | Maximum failure scope | Launch-day isolation | Rollback behavior | Decision |
|---|---|---|---|---|
| Shared account ceiling | Every workload under the account | Poor; one workload can consume another's allowance | Lowering the ceiling affects all workloads | Reject for this launch |
| Per-project ceiling | Workloads grouped in one project | Useful only if the project contains the launch workload alone | Restore the project policy | Accept when project isolation is enforced |
| Dedicated credential ceiling | One credential and its workload | Direct and independently revocable | Restore one credential policy | Preferred boundary |
| Application-only counter | Whatever the application counter covers | Flexible but bypassable by unmetered call paths | Deploy or reconfigure application state | Defense in depth only |

A dedicated credential is the narrowest practical unit in this scenario. It also produces a clean decision rule: raise the cap only if every call made with that credential belongs to the launch workload and every path using that workload is metered. If either condition is false, split the credential before changing the budget.

Project isolation can be equivalent when membership is enforced and reviewed. Labels are not isolation. A project called `launch` that still contains a nightly indexing job has already widened the blast radius, regardless of what the dashboard calls it.

The limitation of credential-level control is operational: the boundary is only as accurate as credential assignment. It is unsuitable when a shared gateway rewrites every downstream call onto one common secret, unless that gateway also enforces an equally narrow workload identity and budget. In that topology, isolate the project or gateway tenant first; raising a nominally per-credential cap would create false confidence.

## The critical policy transition

The control plane should accept a complete desired policy, use compare-and-swap semantics, and preserve the prior version for reversal. This focused Python example uses a generic policy store; it does not include a secret or depend on a commercial API. Times are timezone-aware, and the temporary grant expires even if the scheduled rollback job fails.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta


@dataclass(frozen=True)
class SpendPolicy:
    credential_id: str
    warn_at: int
    stop_at: int
    expires_at: datetime | None
    version: int


def prepare_launch_policy(current: SpendPolicy, now: datetime) -> SpendPolicy:
    if current.credential_id != "cred_launch_worker":
        raise ValueError("policy is not bound to the isolated launch credential")
    if current.stop_at != 400 or current.warn_at > 400:
        raise ValueError("unexpected baseline; require a fresh review")

    return SpendPolicy(
        credential_id=current.credential_id,
        warn_at=600,
        stop_at=1200,
        expires_at=now + timedelta(hours=18),
        version=current.version + 1,
    )


def activate(store, current: SpendPolicy, proposed: SpendPolicy) -> None:
    store.compare_and_swap(
        key=current.credential_id,
        expected_version=current.version,
        new_value=proposed,
    )
```

The exact storage API will differ, but three properties should survive translation. A stale operator cannot overwrite a newer policy version. The warning threshold remains explicit rather than being derived as a percentage of the ceiling. Expiry is policy data, so enforcement can fall back to the baseline even when the deployment pipeline or a human reminder fails.

Exercise both directions before launch. In a non-production environment, replay enough synthetic usage to cross 600 units, verify one deduplicated incident, continue to 1,200, and confirm only the launch credential is rejected. Then activate and roll back a fresh version while requests are in flight. The test is incomplete if it checks a dashboard but never checks enforcement.

During the live window, the operator's first intervention is rollback to the 400-unit baseline. Revoke and replace the credential when there is evidence of disclosure or use outside the intended workload. These actions are intentionally different: rollback narrows economic authority; rotation changes authentication material and has a wider deployment burden.

## Why not remove the ceiling temporarily?

Removing the ceiling was rejected because it converts uncertain demand into unbounded credential authority. An alert cannot substitute for containment: it may arrive late, reach nobody, or describe consumption that has already accelerated beyond the team's response time. Keeping a finite ceiling makes the worst case a policy decision instead of an incident-time guess.

There is a valid use case for operating without a workload-level ceiling: a separately isolated environment where an upstream contractual limit is already enforceable, the service must remain available despite cost, and accountable owners have explicitly accepted that exposure. That is a different risk decision. It should not be smuggled into a launch change as a temporary convenience.

The approval is therefore narrow: one credential, an alert at 600 units, a stop at 1,200, automatic expiry after 18 hours, and a rehearsed return to 400. Pause the increase if credential sharing is discovered, the alert route fails its preflight, usage cannot be reconciled with completed work, or the baseline policy version has changed. This keeps the growth experiment possible without letting one token decide the whole account's invoice.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
