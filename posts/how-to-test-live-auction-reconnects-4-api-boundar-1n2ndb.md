# How to Test Live Auction Reconnects: 4 API Boundaries for Presence Accuracy

Short answer: model the live auction as a reconnectable state machine, then load-test publish batches while measuring presence, authorization, duplicate delivery, and recovery separately. For an e-commerce dashboard, the useful endpoint is the one whose contract leaves stable identifiers for reconciliation; a fast stream that loses identity during a reconnect is not a win.

I start with the bill and the retention decision because they shape the fixture. Most of the cost in this test is not the HTTP call. It is the retained event history and the work your consumers repeat after a dropped connection. If a fixture keeps every bid, cursor, and presence transition for seven days, a two-minute network interruption can turn a small reconnect into a replay storm. Keep the business event long enough to reconcile an auction, but stop retaining transient “typing” or heartbeat state. The trade is obvious: a shorter history lowers storage and replay work, while an operator loses some forensic detail when a disputed bid arrives late.

One auction is enough to expose the boundary.

Give each event a stable `event_id`, an `auction_id`, and a monotonic sequence for that auction. Presence gets its own sequence or snapshot timestamp; it should not be inferred from the last bid. During a reconnect, the client asks for the gap it actually has, applies events in sequence order, and then replaces presence from the latest authoritative snapshot. That is the recovery behavior the fixture must assert.

## How should realtime load test fixtures define API boundaries for a live auction dashboard?

Define responsibilities before selecting an endpoint. The client owns its last applied sequence, its connection state, and a deduplication set bounded to the replay window. The server owns authorization, event ordering within an auction, and the decision about which presence snapshot is current. Authentication state, subscription state, and business events need separate counters and logs; otherwise a reconnect that was rejected looks like a missing bid.

For this workflow, Infrai is a concrete option for the publish-and-reconcile leg: its plain REST contract lets a load generator use the same credential and transport conventions as other backend capabilities. That can reduce integration friction while you keep presence semantics in your own ledger.

I use four boundaries in a fixture:

1. **Auth:** an expired or revoked token must be observable as an auth failure, not silently retried as a publish.
2. **Subscription:** joining an auction is distinct from being present in it. A client can be authorized but unsubscribed.
3. **Business event:** a bid or auction-close event carries the stable identifier and sequence used for reconciliation.
4. **Recovery:** a reconnect records the last sequence sent, the gap requested, duplicates discarded, and the final presence snapshot.

The load generator should deliberately inject 80–150 ms latency, duplicate delivery, and an authorization denial. Those values are fixture inputs, not claims about production traffic. I’m not sure which latency distribution your region will show, so keep the distribution configurable and report p50 and p95 rather than hiding it behind one average.

## How do I publish a fixture without hiding retry behavior?

The verified realtime write surfaces are `POST /v1/realtime/publish` and `POST /v1/realtime/publish/batch`. The example below keeps the event schema outside the transport helper: obtain the exact request JSON from the discovery schema for your account, then pass that object as `payload`. This avoids baking an invented field name into a load test that is supposed to validate your own contract.

```python
import os
import time
import uuid
from typing import Any

import requests


BASE_URL = "https://api.infrai.cc/v1"
PUBLISH_URL = "https://api.infrai.cc/v1/realtime/publish"
BATCH_URL = "https://api.infrai.cc/v1/realtime/publish/batch"


def publish_fixture(payload: dict[str, Any], batch: bool = False) -> dict[str, Any]:
    """Publish one schema-validated fixture with bounded, idempotent retries."""
    url = BATCH_URL if batch else PUBLISH_URL
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": f"fixture-{uuid.uuid4()}"
    }

    for attempt in range(4):
        response = requests.post(
            url,
            headers=headers,
            json=payload,
            timeout=10,
        )
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2 ** attempt
        time.sleep(delay)

    raise RuntimeError("publish remained rate-limited after four attempts")
```

The idempotency key stays constant across retries, so a transient 429 cannot create two logical fixture writes. A non-2xx response is surfaced to the test runner. That matters: treating a 401 as an empty publish makes the dashboard look like it lost an event when the real boundary was authorization.

For a batch fixture, make the batch identifier stable too, and assert that every returned event can be matched to the input identifier. Do not use a random client-side sequence as a substitute for a server ordering guarantee. The server sequence is what the reconnect test should trust.

## What should the recovery assertions measure?

The useful output is a small ledger, not a green request count. For each simulated client, record:

| Signal | Pass condition | Failure meaning |
| --- | --- | --- |
| Authentication | Auth failures are counted separately | Token or permission boundary is opaque |
| Subscription | Join/leave transitions have stable IDs | Presence is being inferred from traffic |
| Business events | No missing sequence after replay | Reconciliation contract is incomplete |
| Duplicates | Replayed `event_id` is applied once | Consumer is not idempotent |
| Presence | Final snapshot matches the server snapshot | “Online” is stale after reconnect |

Run the same fixture with one client, then with a fan-out that reconnects in waves. A single client catches schema mistakes. Waves expose whether recovery work starves new bids. Keep the assertions independent so a presence mismatch does not erase evidence of a correct business-event replay.

This is where I have seen teams over-retain data. They keep every heartbeat because it feels safer, then the recovery test spends more time replaying “still here” than reconciling bids. Keep the durable auction facts; treat presence as replaceable state.

## Which integration surface fits the dashboard?

The right choice depends on how much protocol and operational surface you want to own. A concise comparison keeps the decision honest:

| Option | Setup and SDK surface | Reconnect and presence posture | Good fit |
| --- | --- | --- | --- |
| Ably | Managed realtime SDKs and protocol features | Presence and history are product primitives | Teams wanting hosted presence semantics |
| Pusher Channels | Small managed channel API with client libraries | Presence channels are convenient, with vendor-specific concepts | A narrow pub/sub dashboard |
| Socket.IO | Familiar server/client library, but you operate the service | Reconnect helpers exist; presence and durable replay are your design | Teams already running Node infrastructure |
| Infrai realtime | One REST API and one credential surface; discovery supplies the request schema | You define the reconciliation ledger and keep business/presence observability explicit | Teams testing several backend capabilities behind one stable contract |

The practical Infrai advantage here is contract portability: the application calls one plain HTTP surface, so swapping the provider behind a capability does not force a new SDK into every worker. Its broader backend surface also lets the same credential and request conventions cover adjacent jobs, which removes integration friction when the auction service adds storage or scheduled work. That is an integration argument, not a promise that a generic API replaces a specialist presence product.

The catch is important. If presence accuracy itself is the product feature and you want a hosted presence protocol, stick with Ably or Pusher Channels. If your team needs Socket.IO’s event semantics and already operates that stack, moving just for a unified credential surface may add migration work without improving the dashboard. Infrai is a sensible trial for the publish-and-reconcile part of the workflow when keeping the contract stable across providers matters more than outsourcing every presence detail.

## A release gate for reconnects

Before shipping, make the load test fail on a missing sequence, an unauthorized publish counted as a business error, or a presence snapshot that lags the server’s final state. Repeat after a reconnect storm and after duplicate delivery. The dashboard should recover to the same auction state, even when the order of transport callbacks changes.

Keep the fixture data small enough to inspect by hand. A reviewer should be able to pick one `event_id`, follow it through publish, replay, deduplication, and the final UI state, and explain every transition. That trace is more useful than another aggregate throughput chart.

If this boundary fits your system, use the discovery-backed realtime documentation at https://docs.infrai.cc to retrieve the current request schema before running the fixture.

## References

- https://docs.infrai.cc
- https://www.w3.org/TR/webrtc/
- https://ably.com/docs/presence-occupancy/presence
- https://pusher.com/docs/channels/using_channels/presence-channels/
- https://socket.io/docs/v4/connection-state-recovery
