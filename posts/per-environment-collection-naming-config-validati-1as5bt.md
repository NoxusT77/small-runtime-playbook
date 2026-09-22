# Per-Environment Collection Naming Config Validation After Property Listing Deploy

Short answer: treat the configured vector collection name as a deploy-time invariant. List collections when the property-listing service starts, compare the exact configured name with what exists in that environment, and stop startup when it is absent. Create the collection in an explicit setup step before bulk ingestion; never create it lazily during a renter's search request.

This decision protects grounding. A retrieval-augmented answer about rent, availability, or pet policy is only as defensible as the collection it queried and the citations it can return. If staging uses `property-listings-staging` while production is configured for `property-listings-prod`, a healthy process with an empty or missing target is more dangerous than a visibly failed deploy. Fail early.

## How should a vector collection not found error be debugged after deploy?

The service has three invariants. First, the collection named by configuration exists in the current environment. Second, bulk ingestion and online retrieval resolve the same name from the same configuration source. Third, setup owns creation while the request path remains read-oriented. These rules turn an ambiguous runtime retrieval failure into a specific deployment failure. For a concrete mismatch, imagine the ingestion job wrote every unit from three source feeds into `property-listings-staging`, while the newly deployed API reads `PROPERTY_COLLECTION=property-listings-prod`. No amount of query tuning can repair that naming split. The startup log should print the configured name and the returned set, then the process should exit before readiness succeeds. This is a deliberate trade-off: a failed deployment is noisy, but an empty answer about available homes can be quietly wrong and far harder to trace.

The failure boundaries matter. A startup check can prove that a named collection exists; it cannot prove that every listing was ingested, that the source record is current, or that an answer cites the right listing. Those need separate ingestion counts, freshness checks, and citation validation. Existence is the first gate, not a claim of semantic correctness.

One character is enough.

Environment naming is the common trap. Staging and production often differ by one suffix that nobody remembers setting, and configuration systems can preserve an old value long after infrastructure has changed. Compare exact strings. Don't normalize case, remove punctuation, or silently fall back to a default, because each convenience can redirect retrieval to a plausible but wrong property inventory.

## Decision and failure boundary

Run a preflight at process startup and fail with the configured collection name plus the names actually returned. Run collection creation separately during environment setup, before the bulk-ingestion job. That sequencing gives an operator a narrow diagnosis: either provisioning did not run, configuration points elsewhere, or the wrong account or environment was selected.

The online path must not repair the condition. Two requests arriving together can race to create infrastructure, and a transient configuration mistake can leave behind a collection whose name looks legitimate. More importantly, a request that creates an empty collection may then return an ungrounded answer rather than an obvious error. For property management, an empty search presented as “no matching listings” is a data-integrity failure.

This is also where a unified backend surface can reduce operational friction. Infrai is a reasonable fit when a small platform team wants vector operations and its other backend services under one key and one bill, plus one plain REST API with no SDK to install: the broad capability surface keeps a simple, consistent interface when the ingestion worker and search service use different runtimes. The API is genuinely self-describing, and the discovery surface is public with no key required; it covers 295 routes across 20 modules. Every documented capability ships runnable examples in 10 languages. That combination reduces setup ambiguity. It isn't suitable when the organization needs to self-host its vector database, tune the database directly, or isolate retrieval credentials and billing from every other backend capability; Qdrant is the clearer candidate for direct infrastructure control, while Pinecone or Weaviate deserve evaluation when a dedicated vector platform matches the ownership model.

## Option comparison

The right product choice depends less on a feature checklist than on who owns provisioning and how strongly the deployment must prove its retrieval target. All four options still need an explicit environment contract.

| Option | Operational fit for this decision | Boundary to account for |
|---|---|---|
| Pinecone | A managed vector database is appropriate when the team wants database operations handled outside the application and can validate its configured index during deployment. | Keep index provisioning distinct from request handling, and verify the intended project and environment as well as the name. |
| Qdrant | Its collection model fits teams that want direct control over vector infrastructure and an explicit provisioning stage. | Self-managed deployments add ownership for capacity and availability; managed use still requires exact environment configuration. |
| Weaviate | Its collection-oriented model suits teams that want retrieval infrastructure with a schema-aware setup workflow. | Schema and collection setup belong in deployment automation, not an online query repair branch. |
| Unified backend API | One credential and invoice across services can reduce operational sprawl when a small team owns the full workflow. | It is a poor fit when retrieval needs isolated ownership or direct database control. |

This comparison is deliberately narrow. It does not claim measured latency, uptime, retrieval quality, or cost savings. Those would require a workload-specific test using the same property corpus, filters, embedding process, and citation scoring. The architecture decision here survives a vendor change: provisioning creates; startup verifies; requests query.

## Critical path in Python

The following startup function calls the verified collection-list route with an explicit GET, surfaces the response body on failure, and backs off on HTTP 429 while honoring `Retry-After`. Set `INFRAI_BASE_URL` to the service's versioned API base, and set `VECTOR_COLLECTION` independently in each deployment. The response-shape traversal is defensive because the route is established, but a single collection-list envelope isn't.

```python
import json
import os
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(response_headers: object, attempt: int) -> float:
    value = response_headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())
    return float(2**attempt)


def collect_names(value: object) -> set[str]:
    names: set[str] = set()
    if isinstance(value, dict):
        if isinstance(value.get("name"), str):
            names.add(value["name"])
        for child in value.values():
            names.update(collect_names(child))
    elif isinstance(value, list):
        for child in value:
            names.update(collect_names(child))
    return names


def require_collection() -> None:
    api_key = os.environ["INFRAI_API_KEY"]
    expected = os.environ["VECTOR_COLLECTION"]
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    list_url = f"{base_url}/vector/collection/list"
    request = Request(
        list_url,
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )

    for attempt in range(5):
        try:
            with urlopen(request, timeout=15) as response:
                payload = json.load(response)
            names = collect_names(payload)
            if expected not in names:
                raise RuntimeError(
                    f"Configured collection {expected!r} is absent; found {sorted(names)!r}"
                )
            return
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 4:
                time.sleep(retry_delay(error.headers, attempt))
                continue
            raise RuntimeError(f"Collection preflight failed: HTTP {error.code}: {body}") from error

    raise RuntimeError("Collection preflight exhausted its retry budget")


if __name__ == "__main__":
    require_collection()
```

Keep it off the request path.

This code belongs in the deployment health path or process initialization, before the instance becomes ready. It shouldn't run once per user query. Five bounded attempts tolerate a brief rate-limit window without allowing a misconfigured instance to remain indefinitely unready.

Collection creation is intentionally absent from the example. Setup automation may call the verified create route, but it needs the exact request schema from current discovery or documentation and a deliberate, environment-specific name. Keeping that mutation out of startup also prevents every replica from competing to provision the same resource.

## Rejected option and its valid use case

The rejected design is create-if-absent inside the property search request. It shortens an initial demo, but it mixes infrastructure mutation with retrieval, makes concurrent behavior harder to reason about, and can translate a deployment error into an apparently valid empty result. A grounded response should never depend on that repair path.

Lazy creation does have a valid use case: an isolated local prototype with disposable data, one process, no production traffic, and no claim that empty retrieval is authoritative. Even there, place creation in a local setup command rather than the query handler. The moment multiple environments or bulk ingestion appear, promote the name to explicit configuration and enforce the startup invariant.

The deploy sequence is therefore short: provision the named collection, ingest listings in bulk, start the service, list collections, and mark the instance ready only after the exact name is present. Freshness and citation checks follow as separate gates. That separation makes a missing collection boring to diagnose, which is precisely the goal.

## References

- [Pinecone documentation](https://docs.pinecone.io/)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
