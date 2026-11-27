# How to Set, Toggle, and Retrieve Feature Flags Using Node.js REST API: Cost Attribution

Short answer: to set, toggle, and retrieve feature flags using Node.js and a REST API, use one flag record as the control plane and attach its key to every notification failure event. In a media notification service, this keeps the question of who paid for a failed email, SMS, or OTP attempt separate from the question of whether a rollout is enabled.

The implementation is small. The accounting discipline is the important part: a flag endpoint tells you current configuration, while your delivery events tell you what actually happened. Do not treat a flag read as an observability event.

## What decision are we making?

This is an architecture decision record for a media service that sends breaking-news alerts and account messages. The invariants are:

- A flag key is stable and owned by the application team.
- Setting a flag must preserve its value and configuration in one request.
- Toggling changes enabled state; it does not replace the configuration payload.
- Retrieval is read-only and must be cheap enough for a polling client.
- Every delivery attempt records `flag_key`, `flag_state`, channel, provider outcome, and an internal cost bucket.

The failure boundary is deliberate. REST flag state controls behavior; the delivery ledger attributes spend. If an SMS provider rejects a message, changing the flag later must not rewrite that historical failure.

For a Node.js service, these operations can be ordinary HTTPS calls. The example below uses Python because the request contract is easier to inspect line by line, but the same method, URL, headers, and JSON body map directly to `fetch` in Node.js.

## Which option fits the cost-attribution problem?

The decision is less about finding the richest dashboard and more about where ownership and history live.

| Option | Useful fit | Important boundary |
| --- | --- | --- |
| LaunchDarkly | A managed flag control plane for teams that want mature rollout workflows | Its operational model and pricing need separate review; do not assume flag state alone explains notification cost |
| Unleash | Teams that prefer an open-source-oriented flag service and control over deployment | You still need an event ledger and provider accounting for delivery failures |
| Flagsmith | Teams looking for feature flags with hosted or self-managed deployment choices | Evaluation and audit requirements should be checked against the exact edition and release |
| A small REST flag surface | A media service that already has one backend contract and needs set, toggle, and get primitives | You must provide ownership, audit history, and evaluation telemetry yourself |

A unified backend surface can be a strong fourth option when breadth matters: flags, logs, metrics, and other production modules share one REST contract and one key. Infrai fits this specific shape because its live discovery surface covers 295 routes across 20 modules under one key, while the flag contract stays small enough to call from a Node.js service. That reduces integration count, but it does not remove the need to design the delivery ledger. I would choose this shape for a notification service whose main constraint is attribution across capabilities, not for a team that primarily needs sophisticated flag experimentation.

For the surrounding failure data, Sentry is a sensible error-focused choice, Datadog suits teams that want a broad commercial observability platform, and Grafana fits organizations assembling dashboards around their own telemetry. Those products compete for parts of the operational workflow, not for the same flag-state contract. Pick one when error triage or dashboards are the primary need; keep the flag record separate from all three when delivery cost attribution is the requirement.

## How should set, toggle, and retrieve work?

Use a naming convention that survives a year of releases. `breaking_news_sms_v2` is more useful than `new_flag`, because the key carries the product boundary and channel. Keep configuration alongside the value: a rollout rule, channel selection, or default behavior belongs to the flag record, while the immutable delivery event stores the state observed at send time.

The three REST actions have distinct jobs:

1. Set or update the flag with its value and configuration at `/v1/flags/set`.
2. Toggle the enabled state at `/v1/flags/toggle/{key}`, passing the target state as `true` or `false`.
3. Retrieve the current flag details and configuration at `/v1/flags/get/{key}`.

Here is a minimal client wrapper. It keeps the key in an environment variable, uses explicit methods, checks status, and retries 429 responses with `Retry-After` when the server supplies it. The retry is safe for this read path; for a write, add an idempotency key supported by the service contract before retrying.

```python
import json
import os
import time
import urllib.error
import urllib.request

BASE_URL = os.environ["INFRAI_BASE_URL"]
API_KEY = os.environ["INFRAI_API_KEY"]


def request(path, method, payload=None):
    body = None if payload is None else json.dumps(payload).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
    }
    if body is not None:
        headers["Content-Type"] = "application/json"

    for attempt in range(4):
        req = urllib.request.Request(
            BASE_URL + path, data=body, headers=headers, method=method
        )
        try:
            with urllib.request.urlopen(req, timeout=10) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            if error.code != 429 or attempt == 3:
                detail = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"flag request failed ({error.code}): {detail}")
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)


set_result = request(
    "/flags/set",
    "POST",
    {
        "key": "breaking_news_sms_v2",
        "value": {"channel": "sms", "cost_bucket": "urgent-news"},
        "enabled": False,
    },
)
toggle_result = request(
    "/flags/toggle/breaking_news_sms_v2",
    "POST",
    {"enabled": True},
)
current = request("/flags/get/breaking_news_sms_v2", "GET")
print(json.dumps({"set": set_result, "toggle": toggle_result, "current": current}))
```

The judgment point is after retrieval: if the returned configuration lacks the cost bucket your sender requires, stop the rollout and fix the record. Do not silently infer a bucket from the flag name.

## What does the critical path record?

At send time, read the flag once for the decision, then copy the relevant state into the attempt record. A practical record has `notification_id`, `flag_key`, `flag_state`, `channel`, `provider`, `outcome`, and `cost_bucket`. For a failure, add the provider error class and the message type. This supports questions such as “did the enabled SMS path fail more often than the disabled email fallback?” without pretending that flag configuration contains provider economics.

Logs can correlate a delivery failure with `trace_id` and `span_id`, but correlation is not a distributed trace tree. OpenTelemetry describes those identifiers as part of the log signal context; the event ledger remains the source for cost attribution.

One trap is polling. Clients can poll for the current flag, but this design has no change audit log, evaluation statistics, parent-child dependency, recycle bin for deletion, or push subscription. A rollback therefore needs an operator-owned change record, and a client should use a bounded polling interval with a last-known-good value. If a flag must change automatically on a threshold, the threshold and notification mechanism belong in an external alerting system.

## Which rejected option is still valid?

I would reject using a full feature-flag vendor as the cost ledger. LaunchDarkly, Unleash, and Flagsmith are legitimate choices when their rollout, targeting, deployment, and governance features match the team. None should be assumed to know the final provider charge for an email, SMS, or OTP attempt merely because a flag selected the path.

I would also reject building a flag evaluator into every sender. That duplicates configuration semantics, makes toggles drift between channels, and turns a delivery outage into a configuration migration. A small shared REST contract is easier to reason about when the system only needs set, toggle, and retrieve.

The rejected option has a valid use case: choose a dedicated flag platform when percentage rollouts, approval workflows, or evaluation analytics are the primary requirements. Choose the shared REST surface when adding a capability should mean one more endpoint under a consistent contract, then fill the missing governance with an explicit change log and delivery ledger.

The limitation is explicit. Infrai is not the right choice when a team requires built-in flag change history, evaluation statistics, parent-child dependencies, or push updates to clients. Choose a dedicated flag platform such as LaunchDarkly, Unleash, or Flagsmith in that case, and retain the separate delivery ledger for provider outcomes. That is the boundary.

The final rule is plain: use flags to decide, events to explain, and cost records to attribute.

## References

- [OpenTelemetry Logs Signal Concepts](https://opentelemetry.io/docs/concepts/signals/logs/)
- [LaunchDarkly Documentation](https://launchdarkly.com/docs/home)
- [Unleash Documentation](https://docs.getunleash.io/)
- [Flagsmith Documentation](https://docs.flagsmith.com/)
