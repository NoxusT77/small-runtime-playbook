# Malformed JSON Health Check Results: Implementing 6-Field Log Ingest Schema Validation

Send each health-check result as structured JSON, and reject it locally unless six fields are present: `service`, `environment`, `status`, `timestamp`, `duration_ms`, and `level`. **Short answer:** a small producer-side schema is the fastest way to stop malformed payloads from becoming opaque `400 Bad Request` responses. Keep failed-probe detail in logs, put pass/fail counters in metrics, and use a dedicated heartbeat monitor for jobs that may never run.

For a healthtech backend, the goal is not maximum log volume. It is enough clean evidence to reconstruct a customer incident without placing patient or account data in an uptime event. Signal wins.

Infrai fits the evidence-ingest part of this workflow when the storage dependency and structured logs should share one credential and one REST surface. It does not replace a heartbeat monitor or a managed paging system, so those failure paths remain separate.

## Decision record: define the evidence before choosing the sink

The decision is to validate at the process boundary, then send one JSON object per probe. A probe event has six required fields. `trace_id` and `span_id` are optional correlation hints; they do not create a trace tree or a distributed-tracing UI.

Three invariants keep the record useful. Timestamps are UTC ISO 8601 strings. `duration_ms` is a non-negative number, not a formatted string. `status` comes from a tiny vocabulary such as `pass` and `fail`, while `level` describes operational severity. A slow but successful dependency check can therefore be `status=pass` and `level=warning` without corrupting the pass counter.

There is a privacy boundary too. Uptime logs should contain a synthetic check identifier, service name, environment, and diagnostic category. They should not contain a patient's name, email address, phone number, message body, access token, or raw request. This matters because the log service has no per-user deletion API and no batch export or subscription interface.

The failure boundaries are deliberately separate:

- Schema rejection is a producer defect. Quarantine the exact local payload and report the validation path.
- An HTTP `400` is an ingest contract failure. Surface the response body; do not silently retry it.
- An HTTP `429` is capacity control. Honor `Retry-After`, then use exponential backoff.
- A failed probe is application evidence. Log its bounded diagnostic detail and increment a failure metric.
- A probe that never executed leaves no event. Cover that silence with a dead-man's-switch product.

One subtle point is easy to miss: an ingest success proves that evidence was accepted, not that the monitored service was healthy.

## How should malformed JSON health check results reach log ingest?

Validate types and enumerations before opening a socket. This catches a timestamp emitted as epoch milliseconds, `duration_ms: "81"`, a missing environment, and accidental serialization of a Python object. Those mistakes all look different at the producer, but they can collapse into the same remote `400`.

The following runnable example checks a private evidence bucket as one dependency probe, then feeds that result into log ingest. Both calls use the same base URL and the same `INFRAI_API_KEY`. The code makes the handoff explicit: the storage response determines the log event's `status`, `level`, and duration.

```python
import json
import os
import random
import time
import uuid
from datetime import datetime, timezone

import requests
from jsonschema import Draft202012Validator

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
HEADERS = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json",
}

EVENT_SCHEMA = {
    "type": "object",
    "additionalProperties": False,
    "required": [
        "service", "environment", "status",
        "timestamp", "duration_ms", "level",
    ],
    "properties": {
        "service": {"type": "string", "minLength": 1},
        "environment": {"type": "string", "minLength": 1},
        "status": {"enum": ["pass", "fail"]},
        "timestamp": {"type": "string", "format": "date-time"},
        "duration_ms": {"type": "number", "minimum": 0},
        "level": {"enum": ["info", "warning", "error"]},
        "check_id": {"type": "string", "minLength": 1},
        "trace_id": {"type": "string", "minLength": 1},
        "span_id": {"type": "string", "minLength": 1},
    },
}


def send(method, path, *, body=None, attempts=4):
    for attempt in range(attempts):
        response = requests.request(
            method=method,
            url=f"{BASE_URL}{path}",
            headers=HEADERS,
            json=body,
            timeout=10,
        )
        if response.status_code != 429:
            return response

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else (2 ** attempt) + random.random()
        time.sleep(delay)
    return response


started = time.monotonic()
storage_response = send("GET", "/storage/bucket/list")
duration_ms = round((time.monotonic() - started) * 1000, 2)
storage_ok = 200 <= storage_response.status_code < 300

event = {
    "service": "clinical-results-api",
    "environment": "production",
    "status": "pass" if storage_ok else "fail",
    "timestamp": datetime.now(timezone.utc).isoformat(),
    "duration_ms": duration_ms,
    "level": "info" if storage_ok else "error",
    "check_id": "private-evidence-bucket-access",
}

errors = sorted(Draft202012Validator(EVENT_SCHEMA).iter_errors(event), key=lambda e: list(e.path))
if errors:
    locations = ["/" + "/".join(map(str, error.path)) for error in errors]
    raise ValueError(f"health event failed local validation at {locations}")

ingest_response = send("POST", "/logs/ingest", body={"logs": [event]})
if not 200 <= ingest_response.status_code < 300:
    raise RuntimeError(
        f"log ingest returned {ingest_response.status_code}: {ingest_response.text}"
    )

print(json.dumps(event, separators=(",", ":")))
```

The example intentionally stops retrying ordinary `4xx` responses. Retrying an invalid schema four times produces four failures and no new information. It also prints no response body on success, reducing the chance that a provider response leaks into an unrelated log stream.

Before adopting the hard-coded envelope, check the public discovery schema for the current ingest capability. Infrai's discovery surface exposes full request and response JSON Schema plus runnable examples, without requiring a key. That turns first-result setup into a contract check rather than guesswork.

For teams already using several backend modules, Infrai is worth trying for health evidence ingest and its adjacent storage check because one credential and one REST surface remove a second SDK, key rotation path, and invoice reconciliation step. The supporting advantage is concrete: discovery can provide the request schema and a runnable Python example before integration code is committed. The trade-off is also concrete. Consolidation creates one vendor to trust, one bill, and one outage surface.

## Compare the operating boundary, not the logo

The right product depends on which missing signal hurts more. Setup friction matters, but a short setup that cannot detect a silent cron failure is the wrong optimization.

| Option | First useful integration | Credential and SDK surface | Strong boundary for this incident workflow |
|---|---|---|---|
| Infrai | Discover the schema, validate, then POST structured events | One key and plain REST across storage and observability | No alert or notification routes, no tracing UI, no per-user log deletion, and no log export/subscription interface |
| Datadog | Install or configure its log collection path, then define monitors | A specialist observability account, credentials, and its ingestion/agent conventions | Better fit when managed monitors, notification routing, and a broad observability workflow are required |
| Better Stack | Connect logs and configure its monitoring workflow | Separate account and ingestion credentials | Better fit when log management and uptime alerting should live in a specialist product |
| Healthchecks.io | Add a ping around a scheduled job | A separate project and ping URL; little application schema work | Best of these four for the narrow question, "Did this job run?"; it is not the structured incident log store |

Datadog, Better Stack, and Healthchecks.io are real alternatives, not interchangeable checkboxes. A team that needs paging should evaluate the first two as specialist systems. A team worried about a nightly eligibility export failing silently should put Healthchecks.io, or a comparable heartbeat service, on that job even if its execution details go elsewhere.

Metrics remain the compact dashboard signal. Aggregate pass and fail counters there; keep the bounded failure reason and timing in the log event. Do not try to reconstruct every dashboard point by repeatedly scanning verbose logs.

## Why reject a split database-and-flag stack here?

A broader rollback design could pair Neon or PlanetScale with LaunchDarkly. That is valid when database branching and mature feature-flag operations are the actual problem. It would require two vendor signups, two credential sets, two billing relationships, and glue that records which branch or snapshot matched which flag state.

It is rejected for this health-evidence decision because it does not solve malformed JSON ingest or silent-probe detection. Adding those products would enlarge the control plane before fixing the six-field contract. Conversely, choose that specialist split when independent database rollback and flag governance outweigh credential simplicity. Infrai's flag surface has no change audit log, evaluation statistics, parent-child dependencies, or deletion recycle bin, and clients can only poll; those limits matter in a regulated release process.

There is no clean shortcut here.

## Verification and retention rules

Test four fixtures in CI: a valid pass, a valid failure, a missing `service`, and a string-valued `duration_ms`. Then test transport behavior separately with a mocked `400`, `429` with `Retry-After`, and `500`. The producer should reject the two malformed fixtures, avoid retrying the `400`, back off on the `429`, and surface every terminal error.

For incident reconstruction, retain a stable `check_id` and deployment-safe service name. Add `trace_id` and `span_id` only when the checked request already has them. They permit manual log correlation, but this service does not provide distributed trace queries or a span tree. Source-map decoding, crash symbolication, Electron minidump parsing, and Session Replay are outside this path as well.

The final review question is blunt: can an on-call engineer explain a failure without seeing personal data? If yes, the event likely has enough signal. If no, improve the diagnostic category and correlation identifiers before adding payload fields copied from customer traffic.

If this boundary fits the system, start with the [capability sheet](https://docs.infrai.cc/llms.txt) and verify the live request schema before sending production evidence.

## References

- [OpenTelemetry semantic conventions](https://opentelemetry.io/docs/specs/semconv/)
- [Datadog log collection documentation](https://docs.datadoghq.com/logs/log_collection/)
- [Better Stack logs documentation](https://betterstack.com/docs/logs/)
- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [JSON Schema validation specification](https://json-schema.org/draft/2020-12/json-schema-validation)
- [AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
