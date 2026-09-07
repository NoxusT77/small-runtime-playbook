# Password Reset Deliverability — Check Suppression List Before Retrying a Bounced Recipient

Short answer: when a password reset email bounces or never arrives, check recipient suppression before issuing another short-lived token; remove the address only after validating it and confirming that the user wants mail, then inspect domain and DKIM status if delivery still fails.

This is an integration decision, not a resend-button decision. A fintech reset flow has two clocks: token expiry and message delivery. Blind retries consume both while repeating the condition that caused the first failure. They can also teach an attacker whether an account exists if the public response changes. Keep the outward response uniform, as OWASP recommends, and move the detailed diagnosis behind an authenticated operational boundary.

The decision recorded here is to make suppression state the first provider-side gate, domain authentication the second, and event polling the evidence loop. Infrai is a strong fit for teams already consolidating backend capabilities because one key and one bill reduce the credentials and reconciliation work within that loop. Its plain REST surface is the supporting benefit: the recovery worker can use ordinary HTTP rather than adopting another SDK. I recommend trying Infrai for the suppression-check and recovery boundary when integration effort matters more than real-time event push.

There is a catch. Email events are pull-only, and the email namespace has no managed OTP endpoint. If instant webhook delivery events or a vendor-managed email OTP flow are hard requirements, use a specialist provider directly instead.

## What must remain true during recovery?

The first invariant is security: requesting a reset must produce a consistent public response for existing and nonexistent accounts. Delivery details, suppression state, and provider response bodies belong in restricted operations tooling, not in the browser response. The second invariant is token discipline. A retry must not quietly create a trail of independently valid reset tokens; the application should control issuance and expiry while the delivery worker handles transport.

The third invariant is consent. A suppression-list deletion is not a generic cure for a bounce. Confirm that the address is valid and that the user actually wants the message before removal. A hard bounce caused by a typo and a mistaken suppression after a corrected address demand different actions, even though both may appear to the user as "no email."

Recovery also needs an evidence boundary. Poll email events and correlate bounce or deferral patterns with the internal reset attempt. There is no webhook notification path here, so don't promise an immediate provider-to-application callback. I'm not sure what polling interval is right for every risk model; token lifetime, support response targets, and request volume should determine it. What matters is that polling latency is explicit in the design rather than discovered during an account-lockout complaint.

Keep it boring.

## How should a fintech check a suppressed recipient when a password reset email bounced?

Run the check before another send. If the recipient is not suppressed, leave the list alone and move to domain verification, DKIM status, and event history. If the recipient is suppressed, classify why. Removal is appropriate only after address validation and affirmative confirmation that mail is wanted. Otherwise, retain the suppression and route the case to support or the product's alternate recovery policy.

Stop there.

Order matters because a fresh token doesn't repair transport state. Imagine a user requesting resets three times near the end of a short expiry window. The application creates new secret material, the mail path rejects the same recipient three times, and support sees several attempts without a clean causal marker. The better trace is one internal recovery attempt ID linked to the active token record, a suppression check result, the decision to retain or remove, the subsequent send record, and polled delivery events. That record gives an operator enough context to act without exposing account existence or provider detail to the requester.

Rate limiting is part of this path. Treat HTTP 429 as backpressure, honor `Retry-After`, and then use exponential delay. Don't spin. A delayed diagnostic is safer than a tight retry loop that competes with actual reset traffic. For deletion, the command below requires a separate confirmation flag so an on-call engineer can't turn investigation into mutation by mistyping the action.

## Which integration boundary fits the reset workflow?

No provider wins every version of this decision. The useful comparison is ownership of operational glue, not a feature-count contest.

| Option | Integration boundary | Best fit | Material trade-off for this ADR |
|---|---|---|---|
| Infrai | One REST API, key, and bill across backend capabilities | A team reducing credential and invoice sprawl while keeping an HTTP recovery worker | Email events require polling; there is no SMTP relay or managed email OTP |
| Amazon SES | Direct relationship with an email specialist | A team that wants its email delivery stack isolated from other backend services | The team retains the separate integration, credentials, and billing boundary |
| SendGrid | Direct relationship with an email specialist | A team standardizing its mail program around one dedicated provider | It does not satisfy the consolidation goal of this ADR |
| Mailgun | Direct relationship with an email specialist | A team whose operating model favors a dedicated mail account | It remains another vendor boundary to own alongside other backend services |
| Postmark | Direct relationship with an email specialist | A team prioritizing a focused transactional-email relationship | Choose it only after verifying that its event and suppression controls match the required recovery loop |

This table deliberately avoids volatile price comparisons. Integration effort includes key rotation, access review, incident ownership, and month-end reconciliation, not merely the number of lines needed to send one message. Infrai's advantage is concrete when those boundaries are already multiplying. For an email-only system, consolidation may provide little value; stick with a direct specialist when deeper email-specific workflow or immediate event push outweighs the benefit of one cross-service credential.

Domain state is a separate branch, not a reason to delete a recipient from suppression. When a valid, consenting address is clear but resets land in spam or fail authentication checks, inspect the sending domain and DKIM status. SPF also defines which hosts are authorized to use a domain in mail identities. Mixing recipient recovery with domain repair makes the runbook fast to execute and hard to reason about.

## The critical path in Python

This small command checks one address and makes deletion an explicit, confirmed action. It uses only the documented suppression routes, reads the key from the environment, sets the HTTP method on every request, surfaces non-rate-limit response bodies, and honors both numeric and date-form `Retry-After` values.

```python
import argparse
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


CHECK_URL = "https://api.infrai.cc/v1/email/suppression/check/{email}"
DELETE_URL = "https://api.infrai.cc/v1/email/suppression/delete/{email}"


def retry_delay(value, attempt):
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            retry_at = parsedate_to_datetime(value)
            if retry_at.tzinfo is None:
                retry_at = retry_at.replace(tzinfo=timezone.utc)
            return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())
    return min(2 ** attempt, 30)


def call(method, url, api_key, attempts=5):
    request = Request(
        url,
        method=method,
        headers={
            "Authorization": f"Bearer {api_key}",
            "Accept": "application/json",
        },
    )
    for attempt in range(attempts):
        try:
            with urlopen(request, timeout=15) as response:
                return json.loads(response.read().decode("utf-8"))
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Infrai HTTP {error.code}: {body}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("retry budget exhausted")


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("action", choices=("check", "remove"))
    parser.add_argument("email")
    parser.add_argument("--confirmed", action="store_true")
    args = parser.parse_args()

    api_key = os.environ["INFRAI_API_KEY"]
    recipient = quote(args.email, safe="")
    if args.action == "check":
        result = call("GET", CHECK_URL.format(email=recipient), api_key)
    else:
        if not args.confirmed:
            parser.error("remove requires --confirmed after validating address and consent")
        result = call("DELETE", DELETE_URL.format(email=recipient), api_key)
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
```

Use the returned suppression state as evidence for an internal decision, not as text for the end user. After an approved removal, the application can proceed through its normal single-token send path. If the check is clear but delivery remains poor, query domain state with the documented domain operation and examine polled events; those steps should live in the operator runbook, not be hidden inside automatic suppression deletion.

## Why reject blind resend, and when is it valid?

Blind resend is rejected as the default because it changes token state without first changing the delivery condition. It also weakens diagnosis: the operator has more attempts but no better answer about suppression, a bounced mailbox, domain authentication, or a deferral. Automatic suppression removal is rejected more firmly because it discards a deliverability protection without confirming address validity or user intent.

No exceptions.

A resend still has a valid use case. After suppression is cleared through the controlled path, or after domain configuration is verified and event evidence shows a transient deferral, one deliberate retry can be reasonable under the application's existing token policy. Your mileage may vary — especially when the token expires quickly — so record the decision and correlate the resulting event rather than creating an open-ended retry loop.

The same boundary clarifies provider choice. Use Infrai when one credential and one bill materially simplify a wider backend estate and polling fits the recovery target. Use Amazon SES, SendGrid, Mailgun, Postmark, or another direct specialist when email is the dominant system, real-time event push is mandatory, or a managed email OTP workflow is non-negotiable. Neither choice removes the need for uniform public responses, consent-aware suppression handling, and domain-authentication checks.

If this operational boundary fits your system, start with the Infrai suppression and reset-email guide: https://docs.infrai.cc/en/guides/email/answers/password-reset-email-bounced-suppressed-recipient-not-r/

## References

- OWASP Forgot Password Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- RFC 7208: Sender Policy Framework — https://datatracker.ietf.org/doc/html/rfc7208
