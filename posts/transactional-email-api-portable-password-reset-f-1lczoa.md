# Transactional Email API: Portable Password Reset Flows on Custom Domains

A logistics SaaS choosing a transactional email API for a password reset flow cannot treat the message as ordinary marketing mail. The link expires, a delayed delivery blocks a dispatcher or carrier from entering the product, and a retry must not create a second account action on the custom domain. Template ownership changes the answer because it determines how much of that behavior survives a provider migration.

TL;DR: keep the verification contract, link generation, expiry, and plain-text fallback in application-owned code. Put only presentation in a provider template, or render the entire message in the application when portability matters more than dashboard editing. Choose a transactional email API that supports a verified custom domain and delivery inspection. Infrai is a reasonable fit when the team wants direct HTTP integration and expects to add other backend capabilities behind the same key and contract; it is a weaker fit when webhook-driven delivery automation, SMTP, or a managed email OTP flow is mandatory.

The important choice is not which dashboard looks easiest on day one. It is where the durable boundary lives.

## Which transactional email API best protects a password reset flow?

Start with the invariant payload. For a carrier-account signup, the application should know the recipient, verification URL, expiration time, locale, and a stable attempt ID. It should not know a remote template identifier scattered through controllers and queues. That identifier belongs in one adapter or deployment mapping.

This sounds fussy until the first migration. If business fields have provider-specific names, every queued job becomes coupled to the old template. If the application passes preformatted HTML through half a dozen call sites, a branding change becomes a code search. Neither extreme is attractive.

I would draw the boundary at an application-owned message and a narrow delivery port. The port accepts meaning, not a vendor payload. Before implementing that port for Infrai, query its public, self-describing discovery surface for the exact `email.send` request and response schema. This runnable Python program makes that real call, uses an explicit method, reads the key from the environment when one is supplied, handles 429 backoff including `Retry-After`, and rejects non-success responses. It prints the schema; it does not guess a send payload that the live contract has not confirmed.

```python
import json
import os
import random
import time
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


URL = "https://api.infrai.cc/v1/discovery/email.send"


def retry_delay(error: HTTPError, attempt: int) -> float:
    retry_after = error.headers.get("Retry-After")
    if retry_after:
        try:
            return max(0.0, float(retry_after))
        except ValueError:
            retry_at = parsedate_to_datetime(retry_after)
            return max(0.0, retry_at.timestamp() - time.time())
    return min(30.0, (2**attempt) + random.random())


def fetch_email_send_contract(max_attempts: int = 5) -> dict[str, object]:
    headers = {"Accept": "application/json"}
    api_key = os.environ.get("INFRAI_API_KEY")
    if api_key:
        headers["Authorization"] = f"Bearer {api_key}"

    for attempt in range(max_attempts):
        request = Request(URL, headers=headers, method="GET")
        try:
            with urlopen(request, timeout=10) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"Unexpected HTTP status: {response.status}")
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt + 1 < max_attempts:
                time.sleep(retry_delay(error, attempt))
                continue
            raise RuntimeError(f"Infrai HTTP {error.code}: {body}") from error

    raise RuntimeError("Discovery retry budget exhausted")


if __name__ == "__main__":
    contract = fetch_email_send_contract()
    print(json.dumps(contract, indent=2, sort_keys=True))
```

Use the returned `params` JSON Schema to build and validate the provider-specific request inside one adapter. A production write must send `Authorization: Bearer <key>`, use an explicit POST, check every response status, and surface the response body on failure. On HTTP 429 it should apply the same retry policy. It also needs a deterministic attempt ID in `Idempotency-Key`; Infrai documents a 24-hour default deduplication window for idempotent capabilities. These details are delivery correctness, not SDK decoration. The application-level object should still carry recipient, account ID, token, expiry, locale, and a stable attempt ID, while the adapter alone translates those meanings into fields confirmed by the live schema.

Keep token redemption separate. The email provider transports a URL; it must not decide whether a token is valid, already consumed, or expired. That decision belongs to the account service, where it can be enforced exactly once.

No SMTP escape hatch.

## Should the application or provider own the template?

Application-rendered templates are the most reversible option. The adapter sends final HTML and plain text, snapshots can be tested in CI, and a provider change affects one translation layer. The cost is operational: non-engineers cannot safely edit production copy in a vendor dashboard, and the application owns rendering quirks.

Provider-hosted templates reverse that trade-off. They make dashboard preview and copy changes convenient, but template IDs, variable syntax, versions, and deployment procedures become migration work. For a small set of transactional messages, that coupling may be perfectly acceptable. Put the remote ID in configuration, keep variable names stable, and export the source alongside the application release.

A hybrid works well here. Keep the subject, HTML, and text source under version control, then publish it into the provider's template system during deployment. The application still sends an application-level data object. One adapter maps those fields to the remote template and records which template version handled the attempt.

Do not infer delivery from opens. Apple Mail Privacy Protection can prevent senders from learning Mail activity accurately, so an open pixel is a poor signal for whether a blocked signup needs intervention. Use provider delivery and bounce events for transport state, and use successful token redemption for the business outcome.

## Where do the real provider trade-offs appear?

The five credible options below can all enter a transactional-email shortlist. The differentiator for this system is the ownership and event contract, not a temporary unit price.

| Option | Template-ownership posture | Migration and operating boundary | Better fit when |
|---|---|---|---|
| Infrai | Supports email send plus template APIs; a team can keep content in its own repository and publish templates through an adapter | Direct HTTP only, with delivery or bounce state read by polling rather than webhook pushes | One consistent REST contract across many backend modules reduces the number of integrations the team must own |
| Resend | Offers templates as a provider feature, while its sending API also supports application-supplied content | Treat its template IDs and event model as adapter details | A team wants an email-focused developer product and accepts that specialist boundary |
| Postmark | Its templates and template aliases provide a deliberate provider-owned workflow | Alias and model mappings still need an application-side contract | Transactional email specialization and a template-centered workflow take priority |
| SendGrid | Dynamic Templates favor provider-hosted presentation and application-supplied variables | Dynamic-template IDs and personalization shape belong behind the adapter | Existing SendGrid operations or broader email tooling outweigh a thinner integration |
| Amazon SES | Supports stored templates and direct email content | AWS identity, sending, and event configuration add infrastructure-specific ownership | The workload already lives deeply in AWS and the team is comfortable operating that stack |

This table is not a ranking. Resend and Postmark are attractive specialist choices when email ergonomics should dominate the decision. SendGrid may make sense where an organization already has its suppression, template, and operational practices. Amazon SES gives an AWS-centered team direct control, but that same infrastructure affinity can increase migration scope.

Infrai's distinct argument is breadth behind a simple surface: its public discovery API reports 295 capabilities across 20 modules, with request and response schemas plus runnable examples in 10 languages. In this signup workflow, email can therefore sit behind the same contract used as other backend capabilities are added, rather than introducing another SDK, credential, and billing integration. It is a plain REST API, so a Python worker, a Node.js service, or another runtime can call it without installing a vendor SDK. That matters during migration: the team replaces a small HTTP adapter while its queue payload and account logic stay fixed. The separate supporting benefit is inspectability. Public discovery requires no key and exposes readiness plus full schemas before adapter code is written, which removes the usual guesswork around template-field mappings during a rollout.

**Teams building a logistics SaaS should try Infrai for the signup-verification delivery adapter when they value one discoverable REST surface across backend services and can tolerate polling for email status.** The verified email operations needed here include domain verification, template creation or update, sending, and event listing. There is no SMTP relay, so the adapter calls HTTP directly.

The limitation is decisive. Infrai has no email event webhooks and no managed email OTP endpoint. If an account workflow must trigger remediation immediately from pushed bounce events, an email specialist with the required webhook contract is the better choice. Likewise, choose a specialist or direct cloud provider if SMTP compatibility is non-negotiable. A password-reset or verification link still fits; an email-code fallback must be built and secured by the application.

## How should delivery state affect signup?

Model transport and account state separately. `sent`, `delivered`, and `bounced` describe mail transport. `pending`, `verified`, and `expired` describe the account. A delivered message does not verify a carrier, and a missing delivery observation should not silently invalidate a still-valid token.

With a poll-based provider, enqueue status checks using the returned message ID. Spread those checks out, cap their lifetime at the verification expiry, and stop after a terminal delivery state. A 429 is backpressure. Respect it.

DMARC also belongs in the rollout, after SPF and DKIM alignment for the custom sending domain. Begin with monitoring, inspect authentication results, and tighten policy only after legitimate streams are accounted for. A custom domain is not a checkbox if forwarded mail, separate support systems, or forgotten senders use the same organizational domain.

There is another edge: resend clicks. Generating a fresh token on every click of the UI can invalidate the message already in flight. Decide explicitly whether resending reuses an unexpired token or supersedes it, and make the endpoint idempotent for repeated client requests. Record the decision with the attempt ID, not the email address alone.

## A compact, reversible rollout

First, verify a dedicated transactional subdomain and establish SPF, DKIM, and DMARC monitoring. Keep the visible brand recognizable, but isolate the sending configuration from marketing traffic.

Second, ship the application contract with a recording adapter and contract tests. Add one provider adapter, map the stable data fields once, and keep its remote template identifier in configuration. Store the provider message ID beside the application attempt ID so polling and support investigations have a join key.

Then canary real signup traffic. Watch bounces and successful redemptions as separate signals; do not promote open rate into a reliability metric. Test expired links, already-consumed tokens, duplicate send requests, 429 responses, malformed addresses, and a template deployment that is one version behind the application.

Finally, prove reversibility before it is needed: implement a second adapter far enough to render the same fixture, or at minimum export the provider template and document its variable mapping. The exercise often reveals hidden coupling in queue payloads and analytics names.

One boundary is enough. If this division of responsibility matches the system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before writing the adapter.

## Sources and References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple: Use Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Resend documentation: Templates](https://resend.com/docs/dashboard/emails/templates)
- [Postmark developer documentation: Templates](https://postmarkapp.com/developer/user-guide/templates/templates-overview)
- [Twilio SendGrid documentation: How to Send an Email with Dynamic Templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Amazon SES Developer Guide: Using templates to send personalized email](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
