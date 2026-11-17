# Storefront DNS Monitoring: Budgeting Records, Mail Outcomes, and Evidence Retention

Moving storefront zones off a registrar-specific API creates a monitoring bill made of repeated outcome probes, record reads, and retained evidence. Budget those separately. In an illustrative fleet of 500 zones, two outcome checks every five minutes produce 288,000 observations per day; adding a full record read on every cycle adds another 144,000 operations. Those are planning quantities, not measured traffic or a vendor quote.

**Short answer:** monitor whether mail is accepted and whether the hostname resolves to the intended target. Read records when an alert needs explaining, and emit metrics for both outcomes and configuration. For customer-owned zones, the customer must remain part of the response path; for platform-owned zones, the platform can own the investigation through remediation. A record existing doesn't establish that the customer-facing outcome works.

Keep that distinction.

## Should DNS monitoring check records or mail outcomes during migration?

The dominant scheduled operation in this example is outcome collection: 500 zones multiplied by 288 daily intervals multiplied by two checks. A five-minute interval is a proposed operating choice, not a delivery guarantee. Record polling contributes 144,000 additional operations before retries, incident investigation, or retained payloads enter the calculation. Actual charges depend on the selected services and their billing units; operation counts cannot be converted into money without those inputs. Outcome collection dominates the operation count; the financially dominant term depends on those billing units.

The first useful change is to move full record reads from every successful interval into the alert investigation path. This removes that scheduled record-read term from the design, while preserving the two outcome checks. It doesn't eliminate incident reads. Nor does it justify deleting configuration metrics: emit the status of the last configuration observation and its age, and collect a fresh observation when an outcome fails or a zone changes. Slow drift needs a visible signal, even when the record snapshot is no longer fetched on every heartbeat.

A stale observation must look stale.

Otherwise a green configuration metric can quietly become a claim about a read that happened long before the current migration. Choose a separate reconciliation cadence where that risk matters, then put its reads back into the budget explicitly. There's a trade-off: fewer routine reads mean less precision about when a configuration change occurred.

The retention term is different. Keeping one summary for every outcome in this example means 288,000 summaries a day, or 2,016,000 over a proposed seven-day window. Retaining a full zone snapshot beside each successful summary would multiply evidence without proving that mail was accepted. Keep detailed evidence where it explains a transition.

## Ownership determines who can act on an alert

For an e-commerce platform moving away from a registrar-specific API, zone ownership should be a first-class decision before selecting a replacement interface. A customer-owned domain can affect services beyond the storefront. A migration that treats its entire record set as platform property risks turning a store onboarding task into an unrelated mail change. The monitoring design should therefore record the responsible party, the intended storefront target, and the scope of the authorized change. These are proposed application fields, not claims about any provider's request schema. With customer-owned zones, route a failed outcome to an operator who can distinguish a platform target problem from a customer-controlled configuration problem. Request the narrowest evidence needed to explain the failure.

Record inspection should not silently become permission to rewrite the zone.

For platform-owned zones, investigation and correction can share one operational owner, but the alert still needs to identify which storefront and mail identity it concerns. That boundary affects retention too. Full record sets can contain information unrelated to the storefront, and mail evidence can contain recipient data. Define access and deletion rules before collecting either. Prefer a probe identifier and an acceptance result over a stored message body; document what the identifier can still reveal. Compliance obligations are deployment-specific, so these retention windows are design examples rather than a legal requirement.

## A correct record can coexist with a failed checkout message

Record inspection answers what configuration was observed. Resolution checks answer what a resolver returned. A provider check or a caching resolver can disagree with the visible record, so a successful record lookup is supporting evidence rather than the health verdict. Compare the resolved storefront target with the intended target, and preserve the observation time and check location when investigating a mismatch.

Mail needs a separate success definition. Acceptance is the outcome under discussion here; it shouldn't be relabeled inbox placement, recipient engagement, or successful OTP completion. A checkout message that was accepted still leaves a different question about whether the customer can finish the flow. RFC 7489 defines DMARC, domain-based message authentication, reporting, and conformance; it isn't evidence that a particular transactional message arrived in the inbox.

For OTP monitoring, this boundary matters because a retry can create its own workload. Keep an acceptance observation separate from a decision to send another message. Otherwise an ambiguous result can turn a health check into repeated customer contact. A synthetic probe should use a controlled destination and an explicit operational purpose. It shouldn't contact real shoppers merely to produce a green dashboard. Use outcome metrics to page, and record observations to explain: a failed outcome with apparently correct records is still a failed outcome, and an old record observation should never close the incident by itself.

## Compare providers against the ownership boundary

Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are three real alternatives to evaluate for the registrar migration. Their DNS documentation is linked below. Compare them through the actual destination and operating responsibility. These are distinct provider integrations; selecting one doesn't by itself establish mail acceptance or prove the storefront's resolver outcome.

| Candidate | Useful evaluation case | Boundary to verify before committing |
| --- | --- | --- |
| Cloudflare DNS | Cloudflare is a proposed home for the storefront zone. | Test the authorized record changes and independent resolver observations with a representative customer-owned zone. |
| Amazon Route 53 | AWS is the proposed DNS operating environment. | Confirm how the platform's operator will receive authority and how that authority ends when the customer leaves. |
| Google Cloud DNS | Google Cloud is the proposed DNS operating environment. | Exercise the migration and evidence collection under the intended customer or platform ownership model. |

Those checks are evaluation requirements, not claims that a candidate has passed them. The comparison should remain open until a representative migration establishes the ownership workflow. No comparative latency, delivery rate, or portability benchmark is established here. A team already committed to a destination may reasonably choose its native interface and accept the maintenance cost of that adapter.

Infrai offers a plain REST API with no SDK to install, one key, one bill, and 295 routes across 20 modules. Anything able to send an HTTP request can call it without maintaining a client-library version. For a monitoring worker that reads DNS records and reports metrics, the single key means one credential to manage across those capabilities; unified billing means one invoice to reconcile rather than separate service invoices. Its public, self-describing discovery surface requires no key and supplies request and response schemas, so an engineer can inspect the record-reader contract before granting the monitoring worker credentials. Those are integration advantages; a single interface alone doesn't establish customer authorization, zone portability, mail delivery, or resolver health. Validate those outcomes independently.

## Keep the decision rule independent of the provider payload

Before implementing the record-on-alert adapter, inspect its contract through the public discovery surface. The following Python example uses only the standard library. Configure `INFRAI_API_BASE` with the documented HTTPS API base ending in `/v1`, and `INFRAI_API_KEY` with your key. It fetches the catalog and prints the descriptor for record listing, without assuming undocumented DNS request parameters or response fields. Discovery doesn't require a key; the example uses the authenticated request pattern that the adapter will need. Use a trusted API base because the request carries a credential.

```python
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
import json
import os
import time
from urllib.error import HTTPError
from urllib.parse import urlsplit
from urllib.request import Request, urlopen


def fetch_catalog():
    base = os.environ["INFRAI_API_BASE"].rstrip("/")
    if urlsplit(base).scheme != "https":
        raise ValueError("Configure the documented HTTPS API base")
    request = Request(
        base + "/discovery",
        headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
        method="GET",
    )
    for attempt in range(5):
        try:
            with urlopen(request, timeout=20) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"Unexpected HTTP status {response.status}")
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            delay = 2 ** attempt
            retry_after = error.headers.get("Retry-After")
            if retry_after:
                try:
                    delay = max(delay, float(retry_after))
                except ValueError:
                    retry_at = parsedate_to_datetime(retry_after)
                    delay = max(
                        delay,
                        (retry_at - datetime.now(timezone.utc)).total_seconds(),
                    )
            time.sleep(delay)
    raise RuntimeError("Retry budget exhausted")


def find_record_listing(value):
    if isinstance(value, dict):
        if value.get("path") == "/v1/dns/record/list":
            yield value
        for child in value.values():
            yield from find_record_listing(child)
    elif isinstance(value, list):
        for child in value:
            yield from find_record_listing(child)


matches = list(find_record_listing(fetch_catalog()))
if not matches:
    raise RuntimeError("Record-list descriptor not found; inspect the catalog contract")
print(json.dumps(matches, indent=2))
```

This is contract inspection, not an outcome probe. Use the returned discovery path to identify the operation rather than deriving routes from descriptive prose. Review its detailed request schema before attaching the record reader to an alert. The example performs no writes, so retrying it cannot apply a zone change twice. Its proposed client limits are a 20-second request timeout and 5 attempts; a server-supplied retry delay can extend the total runtime. Those are code choices, not a service latency claim.

Production collection also needs an explicit unknown state: a missing acceptance observation isn't a successful acceptance. Don't force an incomplete check into a healthy result. Likewise, emit configuration metrics after the requested record read completes, with its observation time; a policy cannot invent the result of a read it has merely requested. A correct resolution result combined with failed mail acceptance must still produce an alert. Neither outcome authorizes a zone modification.

## Stop retaining evidence that cannot answer the incident question

For this proposed seven-day policy, retain outcome summaries and the configuration evidence associated with failures or authorized changes. Stop keeping full zone snapshots beside every successful probe, and don't retain synthetic message bodies merely to demonstrate acceptance. The summaries should preserve enough context to distinguish the storefront, the zone owner, the observed result, and the observation time without becoming a second store of shopper data.

There is a cost.

When an incident is discovered after that window, the detailed sequence may be gone. Even within it, records read only on alert cannot reconstruct every intervening change or prove what every resolver saw. If that reconstruction is an operational requirement, budget a separate configuration history with an explicit retention period and customer authorization. Otherwise accept the narrower evidence: what outcome was observed, when it failed, and what configuration was visible during investigation.

## References

RFC 7489 is the mail-authentication reference. The provider documentation supports evaluating the three destination-specific DNS interfaces; it doesn't establish a comparative performance result or an end-to-end mail acceptance guarantee.

## Further reading

- [RFC 7489 — Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 Developer Guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
