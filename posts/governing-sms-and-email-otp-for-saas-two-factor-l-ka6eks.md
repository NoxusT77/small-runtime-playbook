# Governing SMS and Email OTP for SaaS Two-Factor Login Evidence

Short answer: treat an SMS-versus-email change as a controlled authentication migration, not a messaging switch; preserve one evidence contract, replay a versioned eligibility policy, and move each SaaS two-factor login cohort only when its completed challenges support the move.

For a B2B marketplace, the concrete moment is a seller following a new-order notification and reaching an OTP challenge. The important trade-off is reachability versus an independent trust path. Neither SMS OTP nor email OTP is universally more secure, deliverable, convertible, or cheapest across US and EU sellers, and a provider's send dashboard cannot prove why a particular login was allowed.

The migration order is the design.

## How can a channel migration preserve a defensible seller login record?

Begin by emitting a new, channel-neutral event contract beside the existing login path. Do not change delivery allocation yet. Give every decision a challenge identifier and policy version; record a protected destination reference, destination verification state, channel eligibility reason, issue time, expiry, attempt count, and terminal outcome. Link transport identifiers in separate delivery events. Raw OTP values don't belong in logs.

Run that contract in observation mode until every challenge reaches one terminal state and recovery remains distinguishable from ordinary verification. Then run the proposed channel policy in shadow mode, using the same non-secret eligibility inputs but sending no extra message. Inspect disagreements between current and proposed choices. Only after those two checkpoints should a small, predeclared cohort receive a different channel, with rollback keyed to challenge completion, lockout, and recovery-entry signals instead of provider send counts.

This order solves an awkward compliance problem: a team can explain the decision before it changes traffic. It also keeps mixed application versions manageable during a rolling deployment. Event readers should tolerate the prior schema, old policy versions should retain their original meaning, and a rollback should stop new allocations without erasing evidence already written.

Keep it boring.

The catch is that a cautious migration delays a broad channel switch. It is not suitable when an emergency requires an immediate factor reset across the population; that situation needs a separately approved recovery procedure, explicit risk ownership, and its own event type. Quietly labeling mass recovery as normal OTP success would make the audit trail look clean while removing the distinction a reviewer needs.

## What must the evidence ledger remember about SMS and email OTP?

Use a small event vocabulary whose meaning survives a delivery-adapter change:

1. `challenge_created` records the policy version and eligible destination.
2. `delivery_submitted` links an attempt without claiming user receipt.
3. `delivery_observed` stores the strongest channel-specific observation available.
4. `challenge_verified`, `challenge_expired`, or `challenge_locked` closes the normal path.
5. `recovery_started` marks a different trust path.

Transport evidence and authentication evidence are different things. DKIM lets a signer claim responsibility for an email through a signing domain identifier, but it does not prove that the intended seller read the message or controlled the account. Email-open data is weaker still: Apple says Mail Privacy Protection prevents senders from learning Mail activity and downloads remote content in the background. An open isn't a defensible authentication event. For either channel, transport observations belong to the delivery trail; successful code verification belongs to the authentication trail.

Don't collapse those states.

An append-only sequence is more useful than a mutable `verified=true` row because it exposes resends, destination changes, late observations, and recovery in order. Retention and access rules should cover the smallest data set that answers an investigation. Copying a phone number or email address into every event adds exposure without strengthening correlation; a stable, protected reference can connect the records inside the authentication boundary.

Consent and notice evidence fits beside eligibility, not inside the delivery callback. For US and EU cohorts, retain the artifact that the organization's own legal and compliance review requires, its version, its collection time, and its withdrawal state. Region can influence a versioned rule. It shouldn't be hard-coded as a synonym for SMS or email.

## How does policy replay expose a weak two-factor trust path?

Put channel choice behind a pure policy function and delivery behind an adapter. A replay can then ask what the proposed policy would have selected for earlier, non-secret inputs without re-sending OTPs or changing historical outcomes. The returned reason matters as much as the channel because it tells an investigator which eligibility fact authorized the choice.

```python
from dataclasses import dataclass
from enum import Enum


class Channel(str, Enum):
    SMS = "sms"
    EMAIL = "email"


@dataclass(frozen=True)
class FactorState:
    phone_verified: bool
    email_verified: bool
    sms_allowed: bool
    email_controls_recovery: bool
    preferred_channel: Channel | None


@dataclass(frozen=True)
class PolicyDecision:
    channel: Channel
    reason: str
    policy_version: str = "seller-login-v4"


def choose_channel(state: FactorState) -> PolicyDecision:
    sms_eligible = state.phone_verified and state.sms_allowed
    email_eligible = state.email_verified and not state.email_controls_recovery

    if state.preferred_channel is Channel.SMS and sms_eligible:
        return PolicyDecision(Channel.SMS, "eligible_verified_preference")
    if state.preferred_channel is Channel.EMAIL and email_eligible:
        return PolicyDecision(Channel.EMAIL, "eligible_independent_preference")
    if sms_eligible:
        return PolicyDecision(Channel.SMS, "eligible_verified_phone")
    if email_eligible:
        return PolicyDecision(Channel.EMAIL, "eligible_independent_mailbox")
    raise ValueError("no_eligible_factor")
```

The final refusal is intentional. Email OTP is a weak second path when the same mailbox controls password reset, OTP delivery, and recovery. SMS independence is also weakened when support can replace the verified phone through a lightly governed procedure. Replay should therefore cover enrollment, destination change, login, fallback, and recovery inputs rather than comparing channel labels alone.

Channel preference comes after eligibility. Keep SMS for an eligible seller when the verified phone supplies the stronger independent path and observed completion supports it. Keep email when the verified mailbox is independent of password recovery and performs reliably. Choose neither when the evidence cannot support the factor. That rule can stop someone whose only destination is also the recovery destination, so the product must present a governed recovery path instead of silently weakening the normal policy.

I'm not sure which eligible channel will complete better for a particular seller population. Your mileage may vary with destination age, device context, seller behavior, and routing conditions. Shadow results and a controlled cohort under identical expiry, resend, and recovery rules can resolve that uncertainty; imported benchmark percentages can't.

## Which state races can invalidate an otherwise valid OTP challenge?

A resend may replace a code while the first delivery is still in transit. Two tabs may verify concurrently. A seller may change a destination during an active challenge, or a delayed delivery observation may arrive after expiry. The authentication service should own expiry and attempt limits, make verification atomic, relate each resend to its original challenge, and prevent a destination change from mutating an issued challenge. Delivery callbacks should be idempotent observations, never commands that mark a login successful.

Test those transitions before widening the cohort. A synthetic suite can cover expiry, replacement and reuse resend policies, duplicate observations, concurrent verification, destination change, recovery entry, and unknown policy versions without sending real OTPs. The production canary then answers the transport question with controlled accounts while the state machine has already been exercised deterministically.

Metrics must follow the same state model. Measure verified challenges over eligible login attempts, then measure new orders opened over verified challenges so a broken session handoff or notification link does not masquerade as OTP failure. For transport analysis, verified challenges over submitted deliveries is useful, but it combines delivery and user behavior and should be labeled that way. Don't optimize email on opens; Mail Privacy Protection makes that signal unsuitable as person-level completion evidence.

Cost also needs a completed-login denominator. Count transport, retry, support, and recovery work for the same cohort and time window. The cheapest listed message can lead to a more expensive completed login, so price should remain one policy input rather than the architecture's organizing principle.

One last check matters: can the audit record explain a seller's path without consulting mutable provider state? If it can show eligibility, policy version, transport observations, verification, and any recovery transition, the team can change delivery adapters or cohort allocation without changing what “successful two-factor login” means. That stable meaning is the durable result. The winning transport may change.

## References

- https://datatracker.ietf.org/doc/html/rfc6376
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
