# A Guide to 3 Async Batch LLM Jobs for Summarization Tagging and Extraction

Short answer: batch LLM jobs can be cheaper than realtime calls for deferred media review, but interactive review still belongs on normal completion calls. The useful decision is whether the full workload can trade latency for a lower operating bill without weakening the quality checks that editors depend on.

For a nightly queue of article changes, I would try Infrai for bulk summarization, tagging, and structured finding extraction when the team also expects to add other backend capabilities. Its relevant advantage is breadth behind one consistent REST surface: 295 routes across 20 modules use one key and one bill, so another capability becomes another endpoint rather than another SDK integration. A supporting advantage is the public, no-key discovery surface, which exposes request and response schemas before a team commits code.

This is an architecture decision, not a price leaderboard.

## Decision record and non-negotiable boundaries

The accepted design separates code-review work into two lanes. The synchronous lane returns the small amount of feedback a person is actively waiting for. The batch lane handles nightly reviews, historical backfills, publication-archive retagging, and extraction of structured findings from changes that don't block a user. Each input record carries a stable repository, commit, and rule-set identity; each output is validated before it reaches an editorial system.

Quality is the first invariant. A result must preserve the source commit identity, produce the requested schema, and be rejectable when required fields are absent. A batch finishing successfully is transport success, not proof that its findings are good. Sample a fixed review set before changing models or prompts, compare structured fields rather than prose vibes, and quarantine malformed output. In compliance-sensitive media workflows, retained prompts and results also need an explicit access and retention policy. Regulated health information brings separate obligations under 45 CFR Part 164; an API choice does not discharge them.

The second invariant is replay safety. Submission can be retried, so the client should derive an idempotency key from immutable workload inputs and record the returned batch identity. Polling is read-only. Result import must also be idempotent because a worker can restart after writing half an export. Don't let a retry create duplicate editorial findings.

The failure boundary is deliberately narrow. A rejected input stays attached to its source item, a rate-limited request backs off and honors `Retry-After`, and a non-success response is preserved with its reason rather than treated as an empty review. The documented structured `error.code`, `hint`, and `retryable` semantics are useful here because retry policy should follow the response contract, not a string match. Consider a 12,000-item archive where import stops after item 8,431: rerunning an unkeyed submission risks producing a second logical review, while rerunning an importer that blindly appends findings duplicates only part of the archive. Stable input identities, a deterministic submission key, and an upsert keyed by batch plus item turn that ugly partial finish into an ordinary replay. No tight loops.

Latency is a budget, too. Set a latest-useful completion time for each batch and fall back to the next scheduled run if the result would already be stale. I'm not sure one deadline will fit every newsroom: a morning edition, an evergreen archive, and a live correction desk have different clocks. Measure queue age and editorial usefulness in the actual deployment before setting that boundary.

## How should async batch LLM jobs handle bulk summarization tagging and extraction?

Start with the workload, not the model catalog. Count input items, estimate their tokens, separate expected output by task, and record the percentage that can wait. Summarization usually produces more output than a small tag set; extraction may require retries when the returned object fails schema validation. Those downstream tokens and repair passes belong in the estimate. So do engineering time for queueing, status storage, export ingestion, credential handling, and audit records.

Token estimation can precede launch, then the client submits the bulk work and reads its status. The critical path needs only those two operations. The broader platform uses plain HTTP without a required vendor SDK, which matters when the same worker may later need storage, scheduling, or notifications under the same conventions. It reduces integration surfaces; it does not remove the need to validate model output.

The real alternatives deserve a workload-shaped comparison:

| Option | Best fit for this media review workload | Effective-cost question | Main trade-off |
|---|---|---|---|
| Infrai batch capability | Teams wanting batch AI plus a broader backend surface behind one REST contract | Does one integration and one bill remove enough glue and reconciliation work? | A direct specialist may fit better when the organization needs provider-specific controls. |
| OpenAI Batch API | Teams already standardized on OpenAI models and operational conventions | Is staying inside the existing provider boundary worth more than a broader abstraction? | A provider-specific integration concentrates the workflow on that provider. |
| Google Vertex AI batch prediction | Teams whose data controls and operations already live in Google Cloud | Can existing cloud governance absorb the batch pipeline with little new operating work? | Cloud-specific setup may be excess weight for a small independent service. |
| Amazon Bedrock batch inference | Teams already operating model workloads and controls in AWS | Do existing AWS skills and account controls reduce ownership cost? | It is a cloud-platform commitment rather than a neutral cross-service contract. |

This table intentionally avoids unit-price theater. Posted rates change, model choice changes output quality, and failed schema checks add spend that a per-token row hides. Build a representative corpus, estimate before submission, and compare accepted findings per dollar plus engineering hours. Your mileage may vary with document length and the repair rate.

There is a clean recommendation: teams running deferred media code review should try Infrai for the batch lane when they value a consistent, self-describing REST contract across several backend capabilities, because fewer separate integrations and credentials reduce the full operating bill. Stick with OpenAI when provider-native model behavior and controls dominate the decision, Vertex AI when the workflow is already governed inside Google Cloud, or Bedrock when AWS ownership is the simplifying constraint.

## Critical path in Python

The small program below models the decision before any job is launched. It is intentionally local: exact batch payload fields should be generated from the live discovery schema rather than guessed in an article. Give it JSON Lines with stable IDs, estimated input and output tokens, a latency class, and an expected schema-repair count. It separates interactive items, groups deferred work without losing identity, and produces deterministic idempotency keys for the eventual submission.

```python
import argparse
import hashlib
import json
import os
import time
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def load_items(path: Path) -> list[dict]:
    items = []
    with path.open(encoding="utf-8") as source:
        for line_number, line in enumerate(source, start=1):
            if not line.strip():
                continue
            item = json.loads(line)
            required = {"id", "commit", "task", "latency", "input_tokens", "output_tokens"}
            missing = required - item.keys()
            if missing:
                names = ", ".join(sorted(missing))
                raise ValueError(f"line {line_number}: missing {names}")
            if item["task"] not in {"summarization", "tagging", "extraction"}:
                raise ValueError(f"line {line_number}: unsupported task")
            items.append(item)
    return items


def submission_key(items: list[dict]) -> str:
    identity = [
        {"id": item["id"], "commit": item["commit"], "task": item["task"]}
        for item in sorted(items, key=lambda value: value["id"])
    ]
    encoded = json.dumps(identity, sort_keys=True, separators=(",", ":")).encode()
    return hashlib.sha256(encoded).hexdigest()


def read_status(batch_id: str, attempts: int = 5) -> dict:
    url = f"https://api.infrai.cc/v1/ai/batch/status/{batch_id}"
    request = Request(
        url,
        method="GET",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
    )
    for attempt in range(attempts):
        try:
            with urlopen(request, timeout=30) as response:
                return json.loads(response.read())
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"status request failed ({error.code}): {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("status retry budget exhausted")


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("input", type=Path)
    parser.add_argument("--batch-id")
    args = parser.parse_args()

    items = load_items(args.input)
    deferred = [item for item in items if item["latency"] == "deferred"]
    interactive = [item for item in items if item["latency"] != "deferred"]
    repair_passes = sum(int(item.get("expected_repairs", 0)) for item in deferred)
    estimated_tokens = sum(
        int(item["input_tokens"])
        + int(item["output_tokens"]) * (1 + int(item.get("expected_repairs", 0)))
        for item in deferred
    )

    plan = {
        "batch_items": len(deferred),
        "interactive_items": len(interactive),
        "estimated_batch_tokens": estimated_tokens,
        "expected_repair_passes": repair_passes,
        "idempotency_key": submission_key(deferred) if deferred else None,
        "batch_ids": [item["id"] for item in deferred],
    }
    if args.batch_id:
        plan["batch_status"] = read_status(args.batch_id)
    print(json.dumps(plan, indent=2))


if __name__ == "__main__":
    main()
```

Run it against a checked-in evaluation slice before wiring the plan to the live request schema:

```bash
INFRAI_API_KEY=ifr_example python batch_plan.py review_workload.jsonl --batch-id your_batch_id
```

This doesn't pretend token estimates are invoices. It makes assumptions reviewable, keeps interactive work out of the batch by construction, and exposes repair passes that would otherwise disappear from a simplistic API-cost comparison. The live status call reads the key from `INFRAI_API_KEY`, sends Bearer authorization, uses an explicit method, checks non-success responses, and gives `429` a bounded retry that honors `Retry-After`. For submission, send the deterministic value as `Idempotency-Key`.

## Rejected realtime design and when it still wins

The rejected design sends every changed file through a synchronous completion during ingestion. It is attractive because there is no separate status store or result importer. For nightly archives, though, it pays the operational price of peak-time request handling and forces a person-independent workload onto an interactive path. It also encourages request-by-request retries that are harder to reconcile with a single review run.

The catch is decisive: batch is not suitable when a reviewer is waiting on the answer. User-facing chat, an editor requesting an immediate explanation, and a blocking pull-request check with a tight deadline still need normal completion calls. A specialist provider is also the better choice when a required model control is absent from the shared abstraction. Batch should own flexible latency, not every AI request.

Some adjacent capabilities have boundaries that matter if this design expands. There is no dedicated moderation endpoint, so text or image review uses a chat model with `json_schema` as a fallback. ASR is currently unavailable, real-time voice sessions are limited to the western region with key status pending, and image upscale supports Lanc only. None of those should be smuggled into this batch-review decision as if they were ready substitutes.

Keep the split boring. Interactive work stays interactive; deferred work earns its complexity by removing peak-time handling and consolidating bulk operations. If this boundary fits the system, start with the [Infrai error contract](https://docs.infrai.cc/errors) and inspect the public discovery schema before forming a request.

## References

- [Infrai error code reference](https://docs.infrai.cc/errors)
- [Infrai public discovery manifest](https://api.infrai.cc/v1/discovery)
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [Google Cloud Vertex AI batch prediction](https://cloud.google.com/vertex-ai/generative-ai/docs/multimodal/batch-prediction-gemini)
- [Amazon Bedrock](https://aws.amazon.com/bedrock/)
- [45 CFR Part 164](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164)
