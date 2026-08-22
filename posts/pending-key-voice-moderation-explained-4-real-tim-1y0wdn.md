# Pending-Key Voice Moderation Explained — 4 Real-Time Western Call Boundaries

Short answer: treat real-time voice moderation as a bounded safety pipeline, not a speech-to-text feature. For e-commerce support calls, a pending API key or unavailable Western region should put the system into an explicit degraded mode: preserve the call, limit automated actions, route the resulting ticket for review, and keep the transcription and moderation providers replaceable behind one internal contract.

The decision is less about which model produces the prettiest transcript and more about what the application does during silence, partial speech, delayed segments, ambiguous policy labels, and unavailable regional capacity. Four boundaries matter: admission, transcription, policy decision, and downstream action.

Keep those separate.

This is an architecture decision record for that design. The goal is not to block a customer because a classifier had a bad minute. It is to create a useful support ticket without pretending that missing evidence is safe evidence.

## How should real-time voice moderation handle pending keys in Western regions?

A pending key is an admission-state problem. The application should discover it before accepting a call into a mode that promises live moderation. If regional access is unavailable, the call can continue only under a clearly selected alternative, such as recording for later review or routing directly to a trained agent. It should not quietly swap “no decision” for “allow.”

The same rule applies after admission. Speech recognizers emit partial hypotheses that can change as more audio arrives; moderation decisions made from those fragments are provisional by design. A fragment such as “don't cancel” can briefly resemble “cancel,” and an order number can be mistaken for other digit-heavy content. Those are ordinary uncertainty boundaries, not exceptional events. The ticket model therefore needs separate fields for transcript state, policy state, and action state rather than a single `moderated: true` flag.

Silence is ambiguous.

For this e-commerce workflow, the safe result of ambiguity is usually “continue the call and constrain automation,” not “terminate the customer.” An agent may still see a warning, while irreversible actions such as account suspension remain unavailable until a final segment or human review supplies enough evidence. Consider a caller disputing an order after saying, “I didn't authorize that charge.” The live transcript might first contain only “I didn't authorize,” while the ticket has not yet received the object of the sentence, the final segment marker, or a policy result. The admission boundary says the call was accepted under live controls; the transcription boundary says evidence is incomplete; the policy boundary therefore returns `unknown`; and the action boundary can open a review task but cannot restrict the buyer. When the finalized segment arrives, the system evaluates it under a recorded policy version and updates that same task by idempotency key. No stage needs to guess what another stage meant. That choice reflects the asymmetric cost: a delayed ticket is inconvenient; an unjustified enforcement action can strand a legitimate buyer during a return, fraud report, or delivery dispute.

“Western region” also isn't a sufficient deployment requirement. Write down the actual processing location, storage location, failover location, retention window, and people allowed to replay audio. A provider's region label answers only part of that set. If a call can contain health information, the privacy and security obligations in 45 CFR Part 164 are a separate review from model accuracy; applicability depends on the organization and data flow, so counsel and the security owner must resolve it rather than an architecture diagram.

## Invariants and failure boundaries

The first invariant is blunt: **absence of a moderation result is never an allow result**. Represent `UNAVAILABLE`, `PENDING`, `PARTIAL`, `FINAL`, and `REVIEW_REQUIRED` as distinct states. Don't flatten them into booleans, and don't infer one from an empty string. Empty audio, a network timeout, and a clean transcript all produce different operational decisions.

The second invariant is replayability. Store a content-addressed reference to the permitted audio or transcript, the policy version, the normalized decision, and timestamps for each stage. Avoid putting raw call text in general application logs. The support ticket should carry the smallest excerpt needed by the reviewer, while access-controlled evidence lives under the retention policy. This is where compliance-aware design and debugging agree: structured metadata is more useful than a giant log line containing a customer's address, phone number, and complaint.

The third invariant is idempotency at the action boundary. A reconnect may deliver the same finalized segment twice. Moderating it twice is mostly wasted work; suspending an account twice, opening duplicate tickets, or sending two customer notices is an operational defect. Build an action key from the call ID, stable segment ID, policy version, and action type, then enforce uniqueness where actions are recorded.

The fourth invariant is provider-neutral semantics. Adapters may expose different labels and confidence-like values, but the rest of the system should consume a small internal vocabulary: `allow`, `review`, `restrict`, and `unknown`, plus evidence references and reasons. Preserve the raw response for authorized diagnosis, yet never let provider-specific fields leak into ticket routing rules. Otherwise a provider change becomes a rewrite of business logic precisely when the team needs an emergency exit.

I wouldn't use one global confidence threshold. I'm not sure any threshold transfers cleanly across accents, noisy mobile connections, product names, and rapidly spoken tracking numbers; a labeled evaluation set from the actual queue is what would resolve that uncertainty. Track false restrictions and missed policy events by language, channel, call reason, and audio quality. Aggregate accuracy can hide the subgroup that support hears from every day.

## Comparing the fallback options

The alternatives solve different failures. Choosing one universal fallback tends to create either an expensive review queue or an unsafe automatic allow path.

| Option | Works when | Main limitation | Portability effect |
| --- | --- | --- | --- |
| Buffered streaming transcription | Live access exists but final segments arrive late | Adds decision latency and requires bounded audio buffering | Good if adapters emit the same segment schema |
| Asynchronous transcription and review | Live regional access or credentials are pending | Cannot promise intervention during the call | Strong; the queue decouples capture from processing |
| Self-managed speech recognition | Data placement or provider access rules out a hosted path | The team owns capacity, updates, language coverage, and evaluation | Strong at the API boundary, heavier operationally |
| Human-first routing | High-risk calls are rare and agents are available | Review capacity and response time become the bottleneck | High; automation remains optional |
| Menu or text intake | The request can be narrowed before free-form speech | Poor fit for nuanced disputes or accessibility needs | High; reduces dependence on speech recognition |

For an incoming ticket queue, asynchronous transcription is often the cleanest degraded mode because it preserves evidence and makes latency explicit. The catch is important: it is not suitable when the product promises intervention during the live call. Human-first routing is the better choice for a small, high-risk queue. Stick with a managed live pipeline when regional processing, credentials, language coverage, and measured latency all satisfy the call's requirements; take on self-managed recognition only when the control gained is worth owning its operational surface.

Cost belongs in this record, but not as a single per-minute quote. Compare retained audio, streaming connection time, transcription, moderation calls, retries, human review, and engineering on-call load under the same traffic assumptions. Your mileage may vary sharply because the review rate, not the headline inference rate, can dominate a support operation.

## The critical path in Python

The following code is deliberately an internal boundary rather than a vendor client. It keeps partial text away from irreversible actions, treats a missing stage as `unknown`, and returns a deterministic ticket instruction. The `1.5`-second budget is illustrative, not a universal target; production values must come from the call experience and measured tail latency.

```python
from __future__ import annotations

import asyncio
from dataclasses import dataclass
from enum import Enum
from typing import Protocol


class Decision(str, Enum):
    ALLOW = "allow"
    REVIEW = "review"
    RESTRICT = "restrict"
    UNKNOWN = "unknown"


@dataclass(frozen=True)
class Segment:
    call_id: str
    segment_id: str
    text: str
    is_final: bool


@dataclass(frozen=True)
class PolicyResult:
    decision: Decision
    reason_codes: tuple[str, ...]
    policy_version: str


class SpeechAdapter(Protocol):
    async def finalize(self, audio: bytes, call_id: str) -> Segment: ...


class PolicyAdapter(Protocol):
    async def evaluate(self, segment: Segment) -> PolicyResult: ...


async def build_ticket_instruction(
    audio: bytes,
    call_id: str,
    speech: SpeechAdapter,
    policy: PolicyAdapter,
    stage_timeout_seconds: float = 1.5,
) -> dict[str, object]:
    try:
        segment = await asyncio.wait_for(
            speech.finalize(audio, call_id),
            timeout=stage_timeout_seconds,
        )
    except (TimeoutError, asyncio.TimeoutError):
        return {
            "call_id": call_id,
            "decision": Decision.UNKNOWN.value,
            "route": "human_review",
            "reason_codes": ["TRANSCRIPT_LATE"],
        }

    if not segment.is_final or not segment.text.strip():
        return {
            "call_id": call_id,
            "segment_id": segment.segment_id,
            "decision": Decision.UNKNOWN.value,
            "route": "human_review",
            "reason_codes": ["FINAL_TEXT_REQUIRED"],
        }

    try:
        result = await asyncio.wait_for(
            policy.evaluate(segment),
            timeout=stage_timeout_seconds,
        )
    except (TimeoutError, asyncio.TimeoutError):
        return {
            "call_id": call_id,
            "segment_id": segment.segment_id,
            "decision": Decision.UNKNOWN.value,
            "route": "human_review",
            "reason_codes": ["POLICY_DECISION_LATE"],
        }

    route = "automation" if result.decision == Decision.ALLOW else "human_review"
    return {
        "call_id": call_id,
        "segment_id": segment.segment_id,
        "decision": result.decision.value,
        "route": route,
        "reason_codes": list(result.reason_codes),
        "policy_version": result.policy_version,
    }
```

In a full service, admission happens before this function. Credential state and regional readiness should be checked through a health capability that returns an application status such as `READY`, `KEY_PENDING`, or `REGION_UNAVAILABLE`. Those statuses belong in deployment checks and dashboards, not in customer-facing error text. A schema-constrained function interface can also keep downstream ticket actions inside an enumerated contract; the function-calling guide in Further reading documents that general pattern.

Test the state machine, not just the happy path. Feed it duplicate final segments, reordered partials, empty audio, code-switched speech, long pauses, and a final segment arriving just after the deadline. Then verify two properties: no `unknown` result reaches the automation route, and a repeated action key cannot create a second side effect. Run shadow evaluations before changing a policy version, with actions disabled, so label drift is visible without touching customer accounts.

Operations need stage-level latency, outcome counts, review-queue age, duplicate suppression, and action counts partitioned by policy version. Avoid high-cardinality raw transcript labels in metrics. A spike in `TRANSCRIPT_LATE` means something different from a spike in `review`; merging both into “moderation failures” makes the dashboard tidy and the incident opaque.

## Rejected option and its valid use case

This record rejects a direct `audio -> provider response -> account action` integration. It looks attractive because there are fewer components, but it binds policy semantics, retry behavior, data handling, and irreversible actions to one response shape. It also leaves nowhere honest to put `pending key`, `partial transcript`, or `regional capacity unavailable` except a generic failure branch.

The direct design still has a valid use case: an internal, low-risk annotation tool where every result is reviewed, no customer-facing action occurs, and stored data already fits the organization's access and retention controls. In that setting, fast experimentation can matter more than portability. It is not suitable for automated restriction of user calls.

The final decision rule is simple. Ship live moderation only when admission readiness, final-transcript semantics, policy evaluation, and action idempotency are independently observable and independently replaceable. Until then, label the mode as deferred review and staff the queue accordingly. Don't let a pending credential become a hidden policy decision.

## Further reading

- OpenAI, “Function calling”: https://platform.openai.com/docs/guides/function-calling
- Electronic Code of Federal Regulations, “45 CFR Part 164”: https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164
