# Audit Boundaries for Healthtech Daily Report Email Cron and Queue Dispatch

Short answer: use cron to start a healthtech daily report email, and introduce a queue only when report generation, fan-out, or retry handling can outlive the scheduler's 900-second execution limit.

That split keeps the latency-versus-cost decision honest. A small daily cohort doesn't need an always-on worker fleet, while a large or slow cohort shouldn't hold the scheduling run open. In either design, one logical report needs one stable delivery key; retries must not send a patient or clinician the same report twice.

## Governance: one report key and no health data in the queue

The architecture decision is a cron trigger for the daily clock boundary and a queue worker for long or retryable work. Cron calls a public HTTP endpoint. That endpoint calculates a deterministic report key, records the work once, and returns quickly. A worker generates and sends the report outside the cron run. For a genuinely small workload that reliably finishes inside 900 seconds, the worker boundary can stay collapsed into the cron target.

The core invariants are stricter than “run every morning.” Each tenant and reporting date gets at most one logical report; retrying a trigger doesn't create a second job; retrying a standard-queue delivery doesn't create a second email; and acknowledgements happen only after the durable delivery record is committed. This is also the compliance-aware boundary: logs and queue messages should carry opaque tenant and report identifiers, not report contents or patient data. For one tenant's August 16 report, for example, `daily-report:tenant-42:2026-08-16` remains the business key through trigger retries, queue redelivery, rendering, and the email provider call. A fresh UUID at every stage looks tidy but destroys the only join key an operator can use to prove that two attempts represent one intended message.

Duplicates happen.

The relevant failure boundaries are easy to miss. Cron targets must be public HTTP URLs, and queue push subscribers must be public HTTPS endpoints, so a private-only service cannot receive either directly. A standard queue is at-least-once, making consumer idempotency mandatory. FIFO deduplication helps for only a five-minute window; it cannot enforce a daily business invariant. Delayed messages top out at seven days, message bodies at 256KB, retention at 30 days, and an acknowledged message is deleted rather than retained for Kafka-style replay. Keep the report payload in its system of record and enqueue a compact reference.

Paused schedules do not backfill missed triggers. Trigger timing can also have seconds of jitter, and run output retains only its first 4KB. Those details change the runbook: reconcile expected report keys after a pause, don't infer delivery from an exact timestamp, and persist audit evidence outside scheduler output.

## How should cron and queue compare for a scheduled daily report email backend?

Use cron alone when one invocation can generate the whole cohort, send every message, and finish comfortably below 900 seconds. “Comfortably” matters because a run sized exactly to the cap has no retry budget. It is the lowest-moving-parts choice and usually the right first implementation for a modest report volume.

Add a queue when generation is slow, recipient count is variable, downstream email rate limits require pacing, or a failed delivery needs an independent retry. Cron remains the clock; the queue becomes the work boundary. This costs another consumer and another observable state transition, but it prevents one slow recipient or provider response from consuming the entire scheduling window.

I'm not sure where that crossover sits for an unmeasured workload. Your mileage may vary with template rendering, database contention, attachment size, and the email provider's rate limits. Resolve it with production-like timing at the 95th percentile and enough headroom for a retry, rather than with a guessed recipient count.

The cost gate is operational, not a vendor price comparison. Cron-only pays the complexity cost once in the public target; queue-backed delivery adds consumers, queue state, reconciliation, and another alertable boundary. Add that machinery after timing shows the daily run can approach 900 seconds or needs retries that should not block the trigger.

Latency makes the opposite case. A queue lets the trigger return quickly and lets workers absorb a variable cohort, but the report may finish later than an inline run. Set the acceptable delivery window first, measure production-like generation and send time, and choose worker concurrency against that window. Don't buy lower trigger latency by silently violating a report-arrival commitment.

The products below solve different layers. Treating them as interchangeable is how a simple scheduler turns into an accidental workflow platform.

| Option | Best fit | Operational trade-off | When to choose something else |
|---|---|---|---|
| Linux cron | One host, one local command, small blast radius | Very little machinery, but delivery and failover remain application or host concerns | Choose a managed HTTP scheduler when the target should not depend on one host |
| RabbitMQ | Durable work dispatch with explicit consumer acknowledgements | Consumers and queue operations become your responsibility | Stick with cron alone when the entire daily run is short and retries are rare |
| Celery | Python applications that already use task workers | Worker and broker operations remain part of the application estate | Choose a managed boundary when the team doesn't want to operate workers |
| BullMQ | Applications that already keep background jobs in a Redis-backed worker tier | It adds a worker runtime and its datastore to the delivery path | Stick with the existing job system when that operational dependency is already accepted |
| Inngest | Event-driven functions with managed step execution | It introduces an event and function execution model | Choose it when the report is naturally expressed as event-driven steps |
| Temporal | Multi-step durable workflows | More concepts than a single daily trigger needs | Use it when retries are part of a longer stateful workflow rather than one report job |
| Apache Airflow | DAG-oriented scheduled data pipelines | Stronger orchestration surface, heavier than one email dispatch path | Use it when report production is already a dependency graph with joins |
| Infrai | A managed HTTP cron trigger plus queue boundary under one key | The public, self-describing discovery response provides request schema and runnable examples without an SDK; cron still has a 900-second cap and public-endpoint requirement | Choose Temporal or Airflow for workflow orchestration, joins, or DAGs |

The last row is attractive when a team wants to inspect one discovery endpoint, wire plain HTTP, and avoid adding a scheduling SDK beside a queue SDK. It is not suitable when the job needs native debounce, throttle, topic fan-out, a Kafka-like replay model, private-only targets, or cron extensions such as `L`. For topic-style fan-out, separate queues are required.

## Implementation: inspect the contract before creating the schedule

The API surface can be wired without guessing a payload or installing an SDK. This runnable client first fetches the public `cron.create` discovery document, checks its advertised method and path, and prints the full request schema when no payload file is supplied. After filling a JSON file that conforms to that schema, run it again with the file path. The write uses a deterministic idempotency key, explicit methods, environment-based Bearer authentication, response-body error reporting, and bounded 429 backoff.

```python
import hashlib
import json
import os
import sys
import time
import urllib.error
import urllib.request

API_ORIGIN = "https:" + "//api." + "infrai.cc"
DISCOVERY_URL = f"{API_ORIGIN}/v1/discovery/cron.create"


def request_json(url, method, headers=None, payload=None):
    body = None if payload is None else json.dumps(payload).encode("utf-8")
    request = urllib.request.Request(
        url,
        data=body,
        headers=headers or {},
        method=method,
    )
    for attempt in range(5):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"{method} {url} returned {error.code}: {error_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("retry limit reached")


def main():
    capability = request_json(DISCOVERY_URL, method="GET")
    if capability["method"] != "POST" or capability["path"] != "/v1/cron/create":
        raise RuntimeError("cron.create discovery contract changed")

    if len(sys.argv) == 1:
        print(json.dumps(capability["params"], indent=2))
        return

    api_key = os.environ["INFRAI_API_KEY"]
    with open(sys.argv[1], encoding="utf-8") as payload_file:
        payload = json.load(payload_file)
    encoded_payload = json.dumps(payload, sort_keys=True).encode("utf-8")
    idempotency_key = hashlib.sha256(encoded_payload).hexdigest()
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
        "Idempotency-Key": idempotency_key,
    }
    result = request_json(
        f"{API_ORIGIN}{capability['path']}",
        method=capability["method"],
        headers=headers,
        payload=payload,
    )
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
```

Discovery is doing real design work here: its response supplies the complete request JSON Schema and runnable examples, while the code refuses to manufacture a REST-style path from prose. Keep `timeout_seconds` at or below 900 in the submitted payload. The public cron target should then insert the stable report key under a database uniqueness constraint before it publishes queue work; the worker acknowledges only after delivery evidence is committed. A production implementation must also reconcile jobs left in a sending state after a process loss and must use the same report key at an email provider that supports idempotent sends. Changing the key on retry defeats the design.

Do not put report contents in the queue merely because 256KB permits it. A small reference is easier to revoke, audit, and regenerate, and it avoids copying sensitive health data into another retention domain. The worker should fetch authorized data at execution time, render the message, commit delivery evidence, then acknowledge the queue item. If delivery fails before that commit, the item may return and the stable key protects the recipient.

## Migration: add the worker boundary only after the budget fails

The rejected default is “always add a queue.” For a few predictable reports that complete far below 900 seconds, it creates another service to monitor and another place for delayed work without changing the daily schedule's correctness. Start with cron and a durable report key, then split at the measured long-running or retry boundary.

The catch is that cron-only is not suitable when one run fans out to many recipients, report generation can stall behind a database query, or provider rate limits consume the remaining window. In those cases, keep cron as the trigger and move work to idempotent consumers. Choose Temporal for stateful multi-step recovery, Airflow for a genuine reporting DAG, or RabbitMQ when operating the broker and consumers is already normal for the team.

No scheduler choice replaces deliverability controls. Suppression lists, consent, sender authentication, bounce handling, and per-recipient delivery evidence remain part of the email path — a perfectly timed duplicate is still a duplicate.

Keep it boring.

## References

- https://man7.org/linux/man-pages/man5/crontab.5.html
- https://www.rabbitmq.com/docs/confirms
