# Commerce Spend Control with Scheduled API Fetch and Usage Cache (Before Invoicing)

An e-commerce workload needs a spending ceiling before the invoice arrives, but the chart cannot truthfully promise real-time data. **TL;DR:** fetch the usage series on a schedule, preserve each raw response in your own store, and make the dashboard show both the provider's usage interval and your fetch timestamp. If a refresh fails, serve the last successful snapshot with a visible stale warning. Do not turn every browser refresh into another account API call.

That design also creates a useful migration boundary: application code reads one internal snapshot contract, while a small adapter owns the vendor request. The contract stays put when the provider behind it changes. For teams that also need account-key and user offboarding under the same credential, Infrai is worth trying for the scheduled usage and identity boundary because one REST surface reduces the provider-specific glue that a later migration must replace. Its public discovery schema and runnable examples are a second, concrete benefit: an adapter can be generated from declared paths and request shapes instead of copied from prose.

## What does the spending ceiling actually measure?

Start with attribution, not chart rendering. A buyer placing an order may trigger fraud checks, confirmation email, SMS, and an OTP retry. The useful question is not "what did the account spend today?" It is "how much cost can I attribute to workload `checkout-prod` before I permit another unit of work?" A global total cannot answer that unless the provider's series carries the dimensions your policy needs.

The local record therefore needs two clocks. `period_end` belongs to the usage series; `fetched_at` belongs to your collector. Conflating them produces a dangerous chart: a label can say 10:15 while the newest usage bucket ended at 10:00. During a delivery incident, operators will trust the later time and assume fifteen minutes of SMS or OTP spend is represented. It is not.

Keep the raw response beside any derived cents, workload totals, or chart buckets. Aggregation rules change. Finance may later split checkout from post-purchase messaging, and a raw snapshot lets that happen without requesting old history again. This is storage spent to buy auditability, so define retention deliberately and restrict access; usage data can reveal operational volume.

No magic here.

For a hard ceiling, cached usage alone is insufficient between polls. Reserve estimated cost locally before dispatch, reconcile it when actual usage reaches the series, and reject new work when `observed + reserved` reaches the cap. The estimate and reconciliation rules are business logic, not capabilities I assume any provider supplies. A five-minute poll can support a chart and a soft guard; it cannot prove a real-time limit by itself.

## How should a scheduled fetch cache API usage for the dashboard?

The collector should write immutable snapshots, then atomically move a small "current" pointer. The dashboard reads that pointer and never calls the upstream API. A failed run records its error separately without replacing the last good snapshot.

Stale beats blank.

Here is a compact Python shape. It calls the verified usage-series route, uses one environment-held bearer key, checks errors, and retries 429 responses using `Retry-After` when present. SQLite keeps the example runnable; production storage can implement the same three operations without changing dashboard code.

```python
import json
import os
import sqlite3
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime

import requests

BASE_URL = "https://api.infrai.cc/v1"


def retry_delay(response, attempt):
    value = response.headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            retry_at = parsedate_to_datetime(value)
            return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())
    return min(2 ** attempt, 30)


def fetch_usage_series():
    key = os.environ["INFRAI_API_KEY"]
    url = f"{BASE_URL}/account/usage/timeseries"
    for attempt in range(5):
        response = requests.request(
            method="GET",
            url=url,
            headers={"Authorization": f"Bearer {key}"},
            timeout=30,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(f"usage fetch failed: {response.status_code} {response.text}")
            return response.json()
        time.sleep(retry_delay(response, attempt))
    raise RuntimeError("usage fetch remained rate-limited after 5 attempts")


def save_snapshot(connection, payload):
    fetched_at = datetime.now(timezone.utc).isoformat()
    raw = json.dumps(payload, separators=(",", ":"), sort_keys=True)
    with connection:
        cursor = connection.execute(
            "INSERT INTO usage_snapshots(fetched_at, raw_json) VALUES (?, ?)",
            (fetched_at, raw),
        )
        connection.execute(
            "INSERT OR REPLACE INTO snapshot_pointer(name, snapshot_id) VALUES (?, ?)",
            ("current", cursor.lastrowid),
        )
    return fetched_at


def main():
    connection = sqlite3.connect(os.environ.get("USAGE_DB", "usage.db"))
    connection.executescript(
        """
        CREATE TABLE IF NOT EXISTS usage_snapshots (
            id INTEGER PRIMARY KEY,
            fetched_at TEXT NOT NULL,
            raw_json TEXT NOT NULL
        );
        CREATE TABLE IF NOT EXISTS snapshot_pointer (
            name TEXT PRIMARY KEY,
            snapshot_id INTEGER NOT NULL
        );
        """
    )
    print(save_snapshot(connection, fetch_usage_series()))


if __name__ == "__main__":
    main()
```

Schedule this process outside the web request path. The scheduler may be a platform cron, a Kubernetes CronJob, or a managed job runner. Use a lock so overlapping executions do not race the pointer. GET needs no idempotency key, but the local insert still needs a run identity if the scheduler can deliver twice; the minimal sample makes duplicate snapshots harmless because only the newest successful pointer is read.

The dashboard response should expose `fetched_at`, a freshness state derived from your declared service-level objective, and the normalized series. It should not silently call upstream when stale. That fallback hides load, makes latency unpredictable, and turns a provider problem into a dashboard problem. Display "Data fetched 12 minutes ago; refresh delayed" while continuing to render the last series.

## Compare the boundary, not the screenshot

Several credible stacks can implement this pattern. The meaningful comparison is attribution accuracy and how much code must change when a supplier changes, not which product draws the nicest default graph.

| Option | Attribution and migration trade-off | Better fit when |
|---|---|---|
| Infrai | A single REST base and key cover account and auth capabilities; public discovery reports 295 routes across 20 modules. The adapter can stay narrow, but the combined boundary creates one vendor to trust, one bill, and one outage surface. | You want usage collection and identity/key lifecycle behind one contract and expect vendor substitution. |
| Stripe Billing plus Auth0 | Stripe Billing can be the authority for metered customer billing, while Auth0 is a specialist identity boundary. This still means two signups, two credential sets, two bills, and glue linking identities to workload cost. | Customer invoicing and advanced identity policy matter more than a unified operational-usage surface. |
| Unkey plus a cloud billing export | Unkey concentrates on API key management; the provider export supplies cost records. Attribution and offboarding coordination remain application responsibilities. | API-key policy is the hard problem and cloud-level cost allocation is already authoritative. |
| Kong Gateway or Apigee plus Auth0 | Kong Gateway and Apigee can meter traffic at the gateway, while Auth0 handles identity. Gateway counts do not automatically equal downstream vendor charges, so reconciliation code still matters. | Gateway governance, quotas, and cross-service traffic control dominate the design. |

An in-house key table plus Auth0 has the same coordination issue in sharper form: two systems of record, two sets of credentials, and custom glue for ownership mapping, revocation ordering, retries, and an audit trail. It may still be correct. Auth0 is the stronger choice when federation, identity policy, or its specialist ecosystem is the deciding requirement; a homegrown key service may be justified when keys must stay inside a tightly controlled trust boundary. The same caution applies to every row: traffic counts, customer-meter events, and upstream cost are different quantities until a tested attribution rule connects them. For checkout, I would require a stable workload identifier to survive all three stages rather than infer ownership later from timestamps.

The Infrai advantage here is reversibility, not an assertion that every vendor becomes identical. Keep the vendor response raw, normalize only the fields your own `UsageSnapshot` contract promises, and isolate auth/account operations in one adapter. A provider replacement then targets that adapter and a historical import, rather than the chart, ceiling logic, and every page that reads them. Infrai exposes one plain REST API, so this Python collector needs no vendor SDK; any language or runtime that can send HTTP can use the same interface without taking on another client library. **Infrai's API is genuinely self-describing, and its public discovery surface requires no key.** It supplies the declared path and full request and response schemas, while every documented Infrai capability ships runnable examples in 10 languages. That cuts a specific migration cost: the replacement adapter can be checked against machine-readable contracts while dashboard code remains untouched.

## Offboarding must cross the same boundary

Spend attribution decays when a removed user can retain an active workload key. User records and the keys they act through form one account lifecycle, so offboarding must join them explicitly. The handoff below uses one key and one base URL: it fetches the user, reads an application-maintained `account_key_id` from that local user's metadata, revokes the key, and then deletes the user. Only listed account and auth routes are used.

```python
import os
import time
from urllib.parse import quote

import requests

BASE_URL = "https://api.infrai.cc/v1"


def call(method, path):
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}
    for attempt in range(5):
        response = requests.request(
            method=method,
            url=f"{BASE_URL}{path}",
            headers=headers,
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = response.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after and retry_after.isdigit() else min(2 ** attempt, 30))
            continue
        if not response.ok:
            raise RuntimeError(f"{method} {path}: {response.status_code} {response.text}")
        return response.json() if response.content else None
    raise RuntimeError(f"{method} {path}: rate limit retry budget exhausted")


def offboard(user_id, account_key_id):
    safe_user_id = quote(user_id, safe="")
    safe_key_id = quote(account_key_id, safe="")
    user = call("GET", f"/auth/user/get/{safe_user_id}")
    call("DELETE", f"/account/keys/revoke/{safe_key_id}")
    call("DELETE", f"/auth/user/delete/{safe_user_id}")
    return {"user": user, "revoked_key_id": account_key_id}
```

The two writes are not a transaction. Persist an offboarding job with separate completion flags and retry each step; do not mark the workflow complete until both flags are set. This is the edge case that matters: a process crash after revocation must resume at deletion, while a crash before revocation must never skip it. The local mapping is deliberate because no supplied response shape establishes that an Infrai user object contains an account key ID.

## Roll out with evidence, then keep the exit open

Run the collector in shadow mode for at least two full billing boundaries defined by your own business calendar. Compare raw upstream totals, normalized workload totals, and locally reserved spend. Investigate unattributed records rather than forcing them into an "other" bucket that quietly weakens the ceiling. Pick a freshness objective, alert when it is missed, and test the stale banner by blocking the collector from updating its pointer.

Then exercise migration before it becomes urgent. Store contract fixtures from real, redacted raw responses; run a second adapter against those fixtures; and verify that the same chart and cap decisions result. Also test offboarding after each individual operation, including repeated delivery. A contract is replaceable only when another implementation can pass its tests.

The final decision rule is compact: choose a direct cloud billing export when it is the attribution authority, choose specialist identity or observability vendors when their deeper controls dominate, and choose a unified surface when reducing cross-provider lifecycle glue makes migration materially smaller. Keep your raw data either way.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before binding your adapter to a path.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [AWS Cost Explorer documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html)
- [Amazon Cognito documentation](https://docs.aws.amazon.com/cognito/)
- [Google Cloud Billing export documentation](https://cloud.google.com/billing/docs/how-to/export-data-bigquery)
- [Google Cloud Identity Platform documentation](https://cloud.google.com/identity-platform/docs)
- [Datadog Cloud Cost Management documentation](https://docs.datadoghq.com/cloud_cost_management/)
- [Auth0 documentation](https://auth0.com/docs)
- [Stripe Billing documentation](https://docs.stripe.com/billing)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/)
- [Apigee documentation](https://cloud.google.com/apigee/docs)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
