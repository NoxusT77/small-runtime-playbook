# Property Signup Verification: Node.js SMS OTP Cooldowns and Rate-Limit State

Short answer: for a property-management signup flow, keep SMS delivery behind a narrow Node.js adapter, but own the verification challenge, resend cooldown, attempt counter, and rate-limit state in your application; integration is slightly more work than delegating the whole flow, yet it prevents carrier retries or a provider change from redefining who may create an account.

The important boundary is not the `send()` call. It is the challenge record that connects a phone number, a pending account, a single-purpose verification link, an expiry, and an abuse budget. Treating those as one state machine gives support staff an explainable outcome without exposing the OTP itself. It also keeps the property record out of a half-created state when the message is delayed.

## How can OTP code verification failures reveal cooldown and rate-limit gaps?

It should verify intent and state before it sends anything: the signup is still pending, the destination has been normalized, the current challenge is eligible for resend, and both the account and network abuse budgets permit another attempt. Then it should create a fresh opaque link token, store only a keyed digest, invalidate the prior token, and enqueue one delivery operation. Verification consumes the challenge exactly once.

That ordering matters in property management. A leasing agent may invite a resident whose phone number is later corrected, two browser tabs may submit the same signup, or a household may share a contact number. A phone number therefore cannot stand in for an account identifier. Bind the challenge to a pending-signup ID, and put the phone number in the delivery context rather than using it as the database key. NIST SP 800-63B treats use of the public switched telephone network for out-of-band authentication as restricted and asks verifiers to consider risks such as SIM change and number porting. That does not make SMS useless for every signup. It means a delivered message is evidence of access to a number, not proof of a durable real-world identity. For a high-impact action such as changing payout instructions or taking over a property-owner account, require a stronger authenticator instead of stretching the signup OTP beyond its job.

Purpose first.

I keep the externally visible contract small — request, verify, and status — while the internal state is explicit. The names can vary, but these transitions should not:

| Current state | Event | Result | Client response |
|---|---|---|---|
| absent or expired | request allowed | create challenge and queue delivery | `202 Accepted` |
| active | request inside cooldown | preserve current challenge | `429 Too Many Requests` with `Retry-After` |
| active | resend allowed | rotate token and queue delivery | `202 Accepted` |
| active | correct token | consume challenge and activate signup | success |
| active | wrong token | increment bounded attempt count | generic rejection |
| consumed or expired | verify | do not reactivate | generic rejection |

Keep the rejection generic. An API that says “phone exists,” “token expired,” or “fifth attempt failed” gives an attacker a useful account-enumeration and timing oracle. Internally, those outcomes should remain distinct event codes for operations and fraud analysis. Externally, they can share one status and one bland message.

## Make the challenge store authoritative

The database, not an in-process timer, must decide whether a resend is allowed. Node.js processes restart, autoscaling creates parallel workers, and two HTTP requests can arrive before either handler observes the other's memory. A transaction or atomic conditional write around the pending challenge is the simplest defensible answer.

Use a server-side record along these lines: `challenge_id`, `signup_id`, normalized destination, token digest, purpose, `expires_at`, `next_send_at`, `attempts_remaining`, `send_count`, status, and a version. Exact cooldowns, lifetimes, and attempt budgets are policy choices; I'm not sure there is one universal set of numbers because carrier behavior, threat exposure, and support tolerance differ. What resolves that uncertainty is production delivery data split by country and carrier, plus abuse and completion rates — not a copied constant from a tutorial.

Do not store a reusable verification code in plaintext. Generate the token with a cryptographically secure random source, place the opaque value in an HTTPS link, and store a keyed digest that is scoped to the challenge and purpose. A keyed digest matters when the human-entered code space is small: a plain fast hash can be exhaustively tested after a database leak. Compare digests in constant time, expire the record server-side, and consume it in the same transaction that advances the pending signup.

The following Python sketch is intentionally about the state transition rather than a messaging vendor. A Node.js handler should preserve the same atomic boundary through its database client and return the same outcome classes.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
import hashlib
import hmac
import secrets


@dataclass
class Challenge:
    signup_id: str
    digest: bytes
    expires_at: datetime
    next_send_at: datetime
    attempts_remaining: int
    consumed: bool = False


def token_digest(secret: bytes, signup_id: str, token: str) -> bytes:
    message = f"signup-link:{signup_id}:{token}".encode()
    return hmac.new(secret, message, hashlib.sha256).digest()


def rotate_challenge(secret: bytes, signup_id: str, now: datetime):
    token = secrets.token_urlsafe(24)
    challenge = Challenge(
        signup_id=signup_id,
        digest=token_digest(secret, signup_id, token),
        expires_at=now + timedelta(minutes=10),
        next_send_at=now + timedelta(seconds=45),
        attempts_remaining=5,
    )
    return challenge, token


def verify(secret: bytes, challenge: Challenge, token: str, now: datetime) -> bool:
    if challenge.consumed or now >= challenge.expires_at:
        return False
    candidate = token_digest(secret, challenge.signup_id, token)
    if not hmac.compare_digest(challenge.digest, candidate):
        challenge.attempts_remaining -= 1
        return False
    if challenge.attempts_remaining <= 0:
        return False
    challenge.consumed = True
    return True


now = datetime.now(timezone.utc)
```

In real storage, the decrement and consume operations need conditional updates. For example, consumption should succeed only where the status is active, the expiry is in the future, and the stored version still matches. One request wins. The other gets the same generic rejection as an already-used link.

There is a subtle race in resend handling too. If the service rotates a token and then fails to enqueue its delivery, the resident has an active token they never received; if it enqueues first and rotates later, an older link may remain valid. Avoid a dual write by committing the challenge and an outbox row together, then letting a worker deliver the outbox item with an idempotency key based on the challenge version. A retry may repeat the delivery operation, but it cannot mint a second challenge or reset an abuse counter. This is the kind of dull detail that prevents a busy Monday morning from becoming a support queue.

## Layer cooldowns instead of trusting one counter

A resend cooldown answers “how soon may this signup ask again?” It does not answer “how many destinations may this client probe?” or “how many accounts target the same number?” Those are different abuse shapes. The design needs layered limits with different keys: pending-signup ID, normalized destination, network prefix, and a broader service-wide circuit threshold. Don't let any one key act as a permanent identity claim.

Return `429 Too Many Requests` when a client can safely retry later, and include `Retry-After` based on server time. The UI should disable the resend control using that response, but the server remains authoritative. Browser countdowns are presentation. They are trivial to reset.

Be careful with network-address limits. Apartment offices, universities, and corporate networks can place many legitimate residents behind one address; mobile carriers can do the same at a much larger scale. A hard per-IP denial used alone will reject the exact shared-network users a property platform expects. Weight it as one signal, use a bounded window, and preserve an accessible recovery path through support or a stronger authentication method.

Rate-limit checks also need to happen before expensive delivery work, while counters should record accepted requests rather than only successful sends. Otherwise a downstream rejection becomes a way to avoid the budget. On the verify side, decrement attempts atomically for each wrong candidate and never reset them on resend. Resetting attempts turns the resend endpoint into an unlimited guessing button.

Short limits aren't automatically safer. An aggressively short expiry can increase resend traffic because a late carrier delivery arrives after the challenge has died; a long expiry extends an interception window. Watch the distribution from request acceptance to verified completion, then set policy with a margin that fits the risk tier. Your mileage may vary across regions.

## Separate delivery truth from authentication truth

The send worker knows whether a request was accepted by the delivery integration. The authentication service knows whether the link was verified. Those are separate facts, and merging them creates misleading dashboards. An accepted API request does not prove handset delivery, while a provider callback should never activate an account.

Record stable, non-secret event fields: challenge ID, signup ID, challenge version, template version, country code, provider message reference, event type, timestamps, and a redacted destination. Never log the raw token or full verification URL. Keep provider payloads out of general application logs; if compliance or dispute handling requires retention, isolate access and define a deletion schedule. This is where deliverability discipline overlaps with backend design. SMS does not use email's DMARC mechanism; DMARC aligns email authentication domains and is relevant only if the fallback or invitation channel is email. Do not treat a passing email-domain policy as evidence that an SMS reached a handset. For either channel, model accepted, delivered when evidenced, bounced or rejected, and verified as separate events. Operational alerts should describe ratios and queues, not page on every failed destination. Useful signals include outbox age, request-to-enqueue latency, delivery-event lag, verification completion by challenge version, resend frequency, exhausted attempt budgets, and rate-limit decisions by key class. A sudden rise in resends with stable request volume can indicate delivery degradation or confusing UI copy; a rise concentrated on destinations may indicate abuse. The data won't identify the cause by itself, but it tells the on-call engineer where to look without reading message content.

Logs aren't proof.

Test those boundaries before testing a vendor sandbox. With a fake clock and fake delivery adapter, cover concurrent resend requests, verification at the expiry boundary, a link opened twice, a wrong token racing a correct token, a destination correction, an outbox worker retry, and a stale provider callback. Then run a small integration suite against the real adapter contract. The bulk of correctness should not depend on carrier access.

## Choose the smallest integration boundary you can replace

Integration effort comes in three shapes. A fully managed verification product can remove challenge-state code, but it also owns more policy and may make event semantics or migration harder to control. A raw messaging API leaves all challenge logic with the application and fits teams that already operate transactional state and queues. A gateway abstraction over multiple delivery services can reduce later switching work, but building it before traffic or regional requirements justify it adds mappings, callback normalization, and more tests.

The catch is ownership. A small team without an on-call rotation or experience protecting authentication state should not choose the raw API solely because the initial endpoint looks easy; a managed verification flow may be the safer operational boundary. Stick with one delivery adapter when one region and one channel meet the requirement. Add a second route only after measured delivery gaps, regulatory constraints, or continuity objectives create a concrete need.

Keep the adapter narrow: submit a message, map provider acceptance into an internal reference, authenticate callbacks, and translate delivery events into a fixed internal vocabulary. Provider-specific status strings stop there. The signup service should never branch on a commercial product name, and the token verifier should never call the delivery provider.

Roll out in a compact sequence: deploy the challenge and outbox tables behind the existing signup path, shadow event recording without changing account activation, enable the new verifier for internal test accounts, then expand by property group while watching completion and resend distributions. Preserve the old path only for a defined rollback window, and prevent both paths from accepting the same challenge.

One owner should be able to answer which component can mint a token, which transaction consumes it, and which metrics justify changing a cooldown. If those answers cross three teams and two vendor dashboards, the integration is already more expensive than the API call suggests.

## References

- https://pages.nist.gov/800-63-3/sp800-63b.html
- https://datatracker.ietf.org/doc/html/rfc7489
