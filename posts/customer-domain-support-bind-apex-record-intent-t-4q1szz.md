# Customer Domain Support: Bind Apex Record Intent to Gaming Ownership Proof

For a gaming platform, complete customer domain onboarding only when a fresh ownership challenge and the published routing records match the same approved configuration. An apex A record and a www CNAME are different routing choices; neither is sufficient proof that the tenant requesting onboarding controls the domain.

TL;DR: Keep ownership proof separate from routing. Use an apex A record when the destination supplies suitable IPv4 addresses, or put a CNAME at www when the destination supplies a hostname. A standard CNAME cannot share its owner name with other data, which makes it unsuitable at a zone apex that already carries SOA and NS records. Gate activation on observed DNS evidence tied to a configuration revision, and preserve the domain's existing mail records.

This decision record uses a proposed gaming onboarding flow: a studio attaches example.com to its player portal while account verification and recovery messages continue using the studio's mail domain. The decision axis is drift between what the studio approved and what DNS currently publishes. Price does not settle that question.

## Can an apex A record prove control of a customer domain?

An A record maps a name to an IPv4 address. A CNAME identifies a canonical name instead. RFC 1034 defines the CNAME coexistence restriction; RFC 1035 defines the record formats. At the zone apex, the required SOA and NS data create the conflict. Moving the alias to www changes the owner name and avoids that particular apex restriction, provided www has no conflicting data. It does not configure the bare domain.

Names matter here.

For the proposed portal, example.com and www.example.com are separate hostnames with separate acceptance checks. A customer who publishes the requested www CNAME has not thereby configured apex routing. Nor does an A record pointing at shared platform infrastructure distinguish one tenant from another. The application needs its own challenge that binds a DNS name to an onboarding request. That binding is an application policy, not a property supplied by an address record.

Use an unpredictable challenge at a dedicated name such as _onboarding.example.com. Store its expected value against the tenant, the exact requested hostname, an expiration time, and the configuration revision. Require the issued value to appear as a TXT record at that name. This is a proposed protocol for this system; it is not a claim that DNS defines a universal onboarding challenge. Keep challenge values out of ordinary application logs.

For this design, DNS publication demonstrates control sufficient for this onboarding step.

It doesn't establish legal ownership of a studio, permission to send marketing messages, or consent from players. Those are separate checks.

## Decision: bind proof and routing to one revision

The invariant is narrow: activation must consume evidence for the exact intent revision being activated.

Any edit to the hostname, record mode, or destination invalidates the previous activation decision. A successful probe for revision 7 cannot authorize revision 8, even if the user interface still displays a green check from the earlier attempt.

Consider an illustrative revision 7 that requests example.com A 192.0.2.40. While its DNS probe is running, the studio changes revision 8 to request www.example.com CNAME portal.example.net. Both addresses are documentation examples. If the old probe finishes last, an unconditional update to verified=true would let an apex observation approve a different hostname. Comparing the revision during the database commit closes that race.

The DNS boundary remains separate. A lookup is an observation, and DNS caches can retain earlier answers according to their TTLs. Negative answers can also be cached under RFC 2308. Record the observation time, query name, record type, result category, and intent revision.

A timeout is an inconclusive observation; an absent challenge is a failed observation at that time. Neither justifies activation.

Mail deserves its own boundary. The studio's MX, SPF TXT, DKIM selector records, and DMARC policy must survive the website change. DMARC evaluates alignment involving SPF or DKIM; a portal ownership challenge does not establish that alignment. DNS onboarding should not display a claim such as ready to deliver OTP email merely because the portal resolves. Recovery messages need their own mail authentication and delivery validation.

## Options against the published-record contract

The following choices concern routing. Each still requires the independent ownership challenge.

| Routing option | Acceptance evidence | Trade-off for the studio |
| --- | --- | --- |
| Apex A record | The observed IPv4 address set matches the approved set | Supports the bare name when suitable addresses are supplied; destination address changes require record coordination |
| www CNAME | The direct CNAME target matches the approved hostname | Allows a hostname destination; the bare name needs a separate routing decision |
| Provider-specific apex alias or flattening | The effective answer satisfies a separately documented contract | Configuration semantics depend on the DNS provider; a public lookup alone may not reveal the configured alias target |

A provider-specific alias is not a standard apex CNAME exemption.

Treat it as a separate integration contract. If the onboarding flow requires proof of an exact alias target, effective A answers alone cannot supply that proof; obtain configuration evidence through an explicitly supported integration or choose a record mode the verifier can actually inspect.

For this proposed system, accept apex A and www CNAME first. Add other modes only after specifying their evidence requirements. Exact address-set matching deliberately rejects unexpected extra addresses: this prevents an unapproved destination from being silently accepted, at the cost of requiring an intent update for legitimate destination changes.

IPv6 needs an explicit AAAA contract too; passing the A check should not silently approve an additional address family.

## Critical path: evaluate, then commit conditionally

The Python example below evaluates normalized observations without performing DNS or database I/O. Its caller must query the direct record at the requested owner name, assemble TXT character strings into logical TXT values, and distinguish complete answers from errors. RFC 1035 permits multiple character strings within one TXT record. Keep distinct TXT records separate when comparing challenge values.

```python
from dataclasses import dataclass
from typing import FrozenSet


@dataclass(frozen=True)
class Intent:
    revision: int
    proof_name: str
    proof_value: str
    route_name: str
    route_type: str
    route_values: FrozenSet[str]
    proof_expires_at: int


@dataclass(frozen=True)
class Observation:
    revision: int
    proof_name: str
    proof_values: FrozenSet[str]
    route_name: str
    route_type: str
    route_values: FrozenSet[str]
    complete: bool
    observed_at: int


def eligible(intent: Intent, seen: Observation, now: int,
             max_age_seconds: int) -> bool:
    if max_age_seconds < 0 or not seen.complete:
        return False
    if not (now - max_age_seconds <= seen.observed_at <= now):
        return False
    if now >= intent.proof_expires_at:
        return False
    if seen.revision != intent.revision:
        return False
    if seen.proof_name != intent.proof_name:
        return False
    if intent.proof_value not in seen.proof_values:
        return False
    if (seen.route_name, seen.route_type) != (
        intent.route_name, intent.route_type
    ):
        return False
    if intent.route_type not in {"A", "CNAME"}:
        return False
    if not intent.route_values:
        return False
    if intent.route_type == "CNAME" and len(intent.route_values) != 1:
        return False
    return seen.route_values == intent.route_values
```

Canonicalize DNS names consistently before constructing these objects, including case and the trailing root dot.

Validate address syntax and reject unsupported record modes when accepting intent. The observation collector should impose deadlines, bound retries, and record which resolver supplied the evidence. Its freshness budget is an application policy; it does not flush DNS caches or guarantee that every player will receive the same answer immediately.

The commit is the critical boundary. In one database transaction, recheck that the challenge is unexpired and activate only if the stored intent revision still equals the evaluated revision. Record which revision was verified. If that conditional update affects no record, discard the result and schedule a new probe for the current intent. Do not reuse the old challenge after an intent change.

Test the race directly: pause a probe, edit the destination, then release the old result and assert that activation remains blocked. Also test a stale negative answer, a split TXT value, an unexpected extra address, and a CNAME chain whose terminal address matches while its direct target differs. The last case catches an easy mistake: following a chain answers where traffic may land, but does not establish that the studio published the exact CNAME it approved.

Deploy the evaluator in observation mode before enabling its activation gate. Compare decisions with support-reviewed configurations, then enable enforcement for a limited cohort. Track revision conflicts, challenge mismatches, routing mismatches, and inconclusive lookups separately. A support engineer needs the requested record and the observed record, plus their timestamps; a generic DNS failed banner creates unnecessary tickets. Bound polling to control resolver load and onboarding latency.

## Rejected: make routing success the ownership gate

Reject the shortcut that activates a tenant whenever its hostname resolves to the platform. It drops the tenant-bound challenge and makes a stale or shared routing answer carry more authority than it contains. It also leaves no useful answer when the studio changes its mind while a lookup is in flight.

Routing-only checks still have a valid role after ownership has been established: they can report whether the current DNS destination matches the approved configuration. They should not silently grant a new tenant access to a previously attached name. Define reassignment and removal as explicit lifecycle operations, and retain the decision metadata needed to explain them without retaining challenge secrets indefinitely.

The acceptance rule for the gaming portal is therefore concrete: a current challenge proves the requested DNS control, the routing evidence matches the approved revision, and activation commits against that same revision. Mail readiness remains separately evaluated. That gives the studio an actionable correction when records drift and keeps a portal DNS change from becoming an unsupported promise about account recovery delivery.

## References

- RFC 1034, Domain Names: Concepts and Facilities: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 1035, Domain Names: Implementation and Specification: https://datatracker.ietf.org/doc/html/rfc1035
- RFC 2308, Negative Caching of DNS Queries: https://datatracker.ietf.org/doc/html/rfc2308
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
- RFC 2606, Reserved Top Level DNS Names: https://datatracker.ietf.org/doc/html/rfc2606
- RFC 5737, IPv4 Address Blocks Reserved for Documentation: https://datatracker.ietf.org/doc/html/rfc5737
