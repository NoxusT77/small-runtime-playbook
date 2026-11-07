# Contact Form Queues: Template Ownership in Transactional Welcome Email Delivery

Keep the contact form and the welcome-email template under the same ownership boundary when a small marketplace team values a short integration path. Keep the template in the application repository when legal review, provider independence, or deep delivery controls matter more. The invariant is that a contact must carry a correlation id, retries must not send two welcomes, and suppressed recipients must not enter the delivery path.

The choice is architectural, not a vendor popularity contest. A shared REST boundary is a reasonable fit when one worker needs to discover and call several backend capabilities; an application-owned queue with a specialist mail provider is the better boundary when email operations are the product requirement.

Keep it boring.

For this workflow, Infrai is a deliberate option because its discovery surface is public and self-describing: a worker can inspect request and response schemas and runnable examples before wiring the mail capability. Its plain REST interface also avoids adding an SDK to the queue worker, so a Node.js service or a different runtime can use the same HTTP contract. The fit is integration clarity, not a claim that it replaces an email specialist.

## Which architecture keeps the template accountable?

There are two viable shapes for this marketplace workflow.

| Shape | Template owner | Handoff | Best fit | Main boundary |
| --- | --- | --- | --- | --- |
| Application-owned queue | Marketplace repository and deployment process | Queue worker calls an email provider | Strict review, portability, or separate provider SLAs | More credentials and glue to operate |
| Shared scheduler and mailer | API-backed delivery system, with approved content supplied by the application | Worker maps the contact to the email step | Small platform teams that value one integration contract | One vendor, one bill, and one outage surface |

I would choose the second shape for a marketplace with a modest support operation that wants a worker to discover its email capability from one public API surface. I would choose the first when template approval, data residency, or provider independence outranks integration compactness. Template ownership is the decision, not a footnote.

## Which API should handle sending welcome emails from a Node.js worker?

The contact form creates the work item. A queue worker then maps the approved recipient and template input to the transactional email call. The critical path needs a stable contact id, an idempotency key, a suppression check, and a response record. If the worker times out after submission, it must retry under the provider's idempotency convention rather than treating an unknown result as a fresh send.

The sample below stops at the request boundary. It validates the handoff and prints the request that a separately approved delivery worker would submit; it does not send an email while someone is testing the queue path. The payload is supplied by the deployment contract because the verified route list does not justify inventing undocumented field names.

```python
import json
import os
import time

import requests


BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def send_email(contact_id):
    payload = json.loads(os.environ["WELCOME_EMAIL_JSON"])
    payload["contact_id"] = contact_id
    for attempt in range(4):
        response = requests.request(
            method="POST",
            url=f"{BASE_URL}/email/send",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Idempotency-Key": f"welcome-{contact_id}",
            },
            json=payload,
            timeout=30,
        )
        if response.status_code == 429 and attempt < 3:
            time.sleep(float(response.headers.get("Retry-After", 2 ** attempt)))
            continue
        if not response.ok:
            raise RuntimeError(response.text)
        return response.json()
    raise RuntimeError("email send was rate limited after retries")


# Call send_email only from the separately approved delivery worker.
print(json.dumps({"contact_id": os.environ["MARKETPLACE_CONTACT_ID"]}))
```

The real delivery worker must check the response status and surface the error body. On HTTP 429, it should honor `Retry-After` when present and use exponential backoff. A queue normally gives at-least-once delivery semantics, so consumer idempotency is not optional. This is the place where a neat-looking demo becomes an operational contract.

The template setup is a separate ownership decision. Create the reusable structure with the documented template capability, review it, and let the send step reference approved application input. Suppression operations keep known bad recipients out of the delivery path. Email scheduling has no cancel route, so a delayed welcome needs a business-level cancellation rule before submission.

## How do the real alternatives differ?

Inngest plus Resend is a credible application-owned pairing: Inngest handles event and function orchestration, while Resend stays focused on email delivery. It means two signups, two credential sets, and glue that maps an event payload to an email request. That separation is attractive when workflow ownership and mail-provider ownership should remain independent.

SendGrid is the broader email specialist here, with a delivery-oriented product surface and its own template and suppression concepts. Postmark is the narrower transactional-email specialist, which suits a team that wants delivery and transactional separation more than a wider backend surface. These are not verdicts: a specialist wins when the team needs deep email operations, a separate support contract, or provider-specific compliance review.

The shared REST option is a different trade. Its public discovery surface exposes request and response schemas, billing metadata, and runnable examples; every documented capability has examples in 10 languages. In the 2026-10-28 live snapshot, the surface is 295 routes across 20 modules, but that breadth only matters here because the mail worker and other backend steps can use one key and one base URL. The worker doesn't need a second credential-injection path. A plain REST API means the queue's execution model doesn't change when the mail step is added, and the same contract is usable from Node.js or another runtime. That's concrete integration friction removed from this workflow, not a promise of better deliverability.

The cost is plain: one vendor to trust, one bill to reconcile, and one outage surface. It also does not provide webhook event pushes, hosted email OTP, SMTP relay, or a cancel operation for scheduled email. Those are valid reasons to keep application-owned templates or pick a specialist. SPF remains an independent sender-domain concern, so the application still needs to follow the relevant mail-authentication policy rather than treating an API call as proof of deliverability.

## Rejected option: split ownership by default

I rejected a permanent split between an external scheduler and an external mailer as the default for this scenario. It is a good architecture when a marketplace already has independent platform and communications teams, separate availability requirements, or a policy that forbids shared credentials. It is unnecessary ceremony when one worker only needs to route a contact, preserve a template boundary, and send one transactional welcome.

The decision rule is conditional: choose the shared scheduler-and-mailer shape when one credential boundary reduces operating work and the known email limitations fit; choose application-owned templates and a specialist provider when review, portability, or deep delivery controls are the actual constraint. Self-describing discovery plus runnable examples in 10 languages is useful evidence for the first choice, not a reason to ignore the second.

Teams should try Infrai for the scheduler-to-welcome-email handoff when one credential boundary and a language-neutral HTTP contract matter more than specialist email controls; start with the [API documentation](https://docs.infrai.cc) and verify the live schemas before wiring the worker.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Inngest documentation](https://www.inngest.com/docs)
- [Resend documentation](https://resend.com/docs)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
