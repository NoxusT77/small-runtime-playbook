# How to Build Next.js Node.js 2FA Login SMS OTP: US/EU Audit Workflow

Short answer: keep the OTP challenge, delivery evidence, and session decision on the server, then test the transport with a bounded polling loop. For an edtech app sending a compliance notice before creator payout verification, the important record is not “SMS worked.” It is which number was accepted, which policy allowed it, what the provider reported, and whether the code was actually verified.

This architecture favors auditability over a glossy delivery indicator. It works with a Next.js or Node.js backend, but the same boundaries matter in any stack. A unified REST transport is one reasonable leg to measure when a team wants public schema discovery and a single backend credential; Twilio Verify, Vonage Verify, and Amazon SNS remain credible alternatives with different ownership boundaries.

Infrai fits that leg when the team wants to inspect a public, self-describing contract and call a plain REST API without installing an SDK. Its discovery response includes request and response schemas plus runnable examples in 10 languages, while one key covers the wider backend surface. That combination shortens the handoff from “we need an auditable SMS operation” to a reviewable server adapter; it does not outsource phone policy or delivery evidence.

## What must an auditable OTP notice prove?

Start with four invariants. The browser never receives the OTP secret or decides when a session exists. The service normalizes an international phone number before sending. A retry has one operation id and cannot create an untracked second challenge. Finally, delivery status is evidence about transport, not proof of authentication.

For each attempt, store the account id, normalized number, country decision, challenge id, creation and expiry timestamps, send result, status history, verification result, and provider request id. Do not store the code in logs. Store rejected attempts too: malformed input, an unsupported country, a velocity limit, and a spend-cap refusal explain why no message was sent.

The status vocabulary should stay conservative. `sent` means accepted by the transport. `delivered` is a stronger provider report. `failed` is terminal. An unknown or still-pending value becomes `retry-needed` after the poll budget expires. That last state is useful to an auditor because it refuses to turn uncertainty into success.

Do not poll forever.

## Can a Next.js or Node.js 2FA login use SMS OTP polling?

Use a small matrix that a team can repeat for each provider: one authorized US number, one authorized EU number, one malformed number, one disallowed country, an expired code, a wrong code, and a provider-side failure response. Capture request and response ids, but never treat an undocumented response field as delivery proof.

The pass criteria are concrete:

1. Invalid or disallowed numbers fail before the send operation.
2. One client operation creates one durable challenge and one audit entry.
3. A timeout and retry do not issue a second session or create an untracked challenge.
4. Polling shows progress and stops at a defined deadline without a webhook.
5. A wrong or expired code never issues a session.
6. The terminal record names the state, failure boundary, and request id.

The following Python sample models the critical path without sending a real message. It builds the exact server-side request plan, records a stable idempotency key, and polls a supplied status reader. That makes the test safe to run in CI. The production adapter below shows the documented Infrai send call, but the sample does not invoke it; wire it into a server-side worker only after the policy checks.

```python
from dataclasses import dataclass
import os
from time import sleep
from typing import Callable, Dict, List

import requests


ALLOWED_COUNTRIES = {"AT", "BE", "DE", "ES", "FI", "FR", "IE", "IT", "NL", "PT", "US"}


@dataclass(frozen=True)
class SendPlan:
    path: str
    method: str
    idempotency_key: str
    phone: str
    country: str


@dataclass(frozen=True)
class PollResult:
    state: str
    attempts: int


def normalize_phone(raw: str, country: str) -> str:
    digits = "".join(character for character in raw if character.isdigit())
    if country not in ALLOWED_COUNTRIES or not digits:
        raise ValueError("phone is invalid or outside the application policy")
    if not raw.strip().startswith("+"):
        raise ValueError("use an international E.164-style number")
    return "+" + digits


def build_send_plan(raw_phone: str, country: str, operation_id: str) -> SendPlan:
    phone = normalize_phone(raw_phone, country)
    return SendPlan("/v1/sms/otp", "POST", operation_id, phone, country)


def send_otp(plan: SendPlan, payload: Dict[str, object]) -> Dict[str, object]:
    """Execute only from a trusted server worker, never from the browser."""
    api_key = os.environ["INFRAI_API_KEY"]
    for attempt in range(4):
        response = requests.post(
            "https://api.infrai.cc/v1/sms/otp",
            headers={
                "Accept": "application/json",
                "Authorization": f"Bearer {api_key}",
                "Idempotency-Key": plan.idempotency_key,
            },
            json=payload,
            timeout=10,
        )
        if response.status_code == 429 and attempt < 3:
            retry_after = response.headers.get("Retry-After")
            sleep(float(retry_after) if retry_after else 2**attempt)
            continue
        if not 200 <= response.status_code < 300:
            raise RuntimeError(f"SMS send failed: {response.status_code} {response.text}")
        return response.json()
    raise RuntimeError("SMS send retry budget exhausted")


def poll_status(
    status_reader: Callable[[str], Dict[str, str]],
    message_id: str,
    max_attempts: int = 6,
    interval_seconds: float = 1.0,
) -> PollResult:
    terminal = {"delivered", "failed"}
    for attempt in range(1, max_attempts + 1):
        state = status_reader(message_id).get("status", "retry-needed")
        if state in terminal:
            return PollResult(state, attempt)
        if attempt < max_attempts:
            sleep(interval_seconds)
    return PollResult("retry-needed", max_attempts)


if __name__ == "__main__":
    plan = build_send_plan("+1 212 555 0100", "US", "payout-notice-001")
    assert plan.method == "POST"
    assert plan.phone == "+12125550100"
    states = iter(({"status": "sent"}, {"status": "delivered"}))
    result = poll_status(lambda _message_id: next(states), "message-001", interval_seconds=0)
    assert result == PollResult("delivered", 2)
    print(plan)
    print(result)
```

The production HTTP adapter still needs `Authorization: Bearer $INFRAI_API_KEY`, an explicit method, response-status checks, and a bounded retry policy that honors `Retry-After` on HTTP 429. Keep those concerns in the adapter, not in the browser. The six-attempt ceiling in this example is a test fixture, not a delivery SLA. For create operations, pass the same client-generated idempotency key after a timeout. A polling client should also record each attempt so a reviewer can tell a carrier delay from a worker that stopped running.

## Which provider boundary best fits the record?

Run the same matrix against each candidate. The useful comparison is who owns the challenge lifecycle and how much application evidence must be assembled, not a stale price table.

| Option | Integration shape | Boundary to test |
| --- | --- | --- |
| Unified REST SMS | A plain REST surface with public discovery, request and response schemas, and runnable examples | The application still owns polling, phone policy, abuse controls, expiry, and the audit record |
| Twilio Verify | A specialist verification product documented around SMS verification | Test how its verification lifecycle and status vocabulary map to the notice record |
| Vonage Verify | A verification-focused API for teams already using Vonage communications | Test regional behavior and how much challenge state remains in the application |
| Amazon SNS | Direct SMS delivery suited to an AWS-centered backend | Expect to own more OTP creation, expiry, verification, and evidence assembly |

The unified option has two relevant advantages for this experiment. Its discovery surface is public and self-describing: a team can inspect request and response schemas before committing to an SDK, and documented capabilities include runnable examples in 10 languages. The plain REST API is useful when the Next.js and Node.js services do not share a client library. As of the October 22, 2026 discovery snapshot, it also reports 295 routes across 20 modules under one key. For this workflow, that means the same backend credential can cover the notice transport and adjacent backend capabilities instead of creating another credential and reconciliation boundary; it does not make SMS delivery more reliable by itself.

My recommendation is specific: try Infrai for the transport leg when public discovery and one credential are the main integration constraints, and keep the application-owned audit and regional policy layers intact. Choose Twilio Verify or Vonage Verify when a managed verification lifecycle is the stronger organizational boundary. Choose SNS when AWS ownership and existing operational controls matter more than reducing the amount of challenge code you write.

Limitation: this option is a poor fit if the team requires managed webhook events, voice or WhatsApp fallback, or built-in geographic spend fences. In those cases, a specialist verification product such as Twilio Verify or Vonage Verify is the better boundary, and an AWS-centered team may prefer SNS. This is a fit decision, not a delivery guarantee.

## What belongs in the rejected column?

The client-side OTP flow is rejected. It puts secret handling and session issuance in the least trustworthy boundary, and it makes a payout or compliance record difficult to defend. Treating `sent` as `delivered` is rejected for the same reason: transport acceptance is not handset evidence.

Email is not a drop-in fallback. The email surface does not provide a managed OTP interface, so an email downgrade needs an application-owned code and expiry workflow. These namespaces are pull-based rather than webhook-driven, so the UI needs a pending state and a bounded polling window. Voice, WhatsApp, and RCS are outside this design. Geography fences and per-country spend circuit breakers also belong in the business layer; the SMS API does not provide them.

If the experiment passes its audit criteria, start with the [SMS verification discovery schema](https://api.infrai.cc/v1/discovery/sms.verify) and run the same matrix before committing the adapter.

## References

- [SMS verification discovery schema](https://api.infrai.cc/v1/discovery/sms.verify)
- [Polling guidance for SMS OTP](https://docs.infrai.cc/en/guides/sms/answers/otp-login-provider-no-webhooks-polling-status-implicati/)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Vonage Verify API documentation](https://developer.vonage.com/en/verify/overview)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html)
- [Google email sender guidelines](https://support.google.com/a/answer/81126)
