# Media Mail Cutover: Generate DNS Instructions and Verification PDFs Together

The operational bill for a media publisher's mail-domain cutover includes generating DNS instructions, retaining a verification PDF, and reconciling both with published records. Generate those documents from one source to remove a recurring editing cost. Start by counting versions: with R records and V revisions, retaining every complete snapshot keeps R × V record entries. This is a planning model, not a measured customer bill. A separately edited instruction copy doubles the places that can disagree without making propagation faster.

Short answer: generate customer DNS instructions and the verification PDF from one immutable intended record set. Include exact names and content values, regenerate both artifacts when that set changes, and let the customer forward the document to their DNS administrator. For a publisher installing SPF, DKIM, and DMARC, accelerate the handoff without treating a freshly generated PDF as evidence that mail is ready.

## What actually makes the handoff expensive?

The dominant term to target here is the number of independently maintained representations. For an illustrative three-record revision, an instruction spreadsheet plus a separately composed evidence document creates six editable record entries. One three-entry source rendered twice leaves three authoritative entries. Those are counts of editing surfaces, not a promise about provider charges or engineering hours. The useful change is removing a reconciliation task every time the intended set changes.

Keep values exact.

An administrator receiving a forwarded PDF may never log into the publisher's application. That reader needs the record type, exact name, exact content, and an explicit statement of which revision the document describes. A prose summary such as “add our authentication settings” makes the administrator reconstruct the most consequential part of the request. A recognizable document title and a revision digest are more useful than a polished paragraph about email security. Keep the record source separate from presentation: instruction wording can evolve without changing the target values, while a changed target must produce a new revision that both renderers consume.

Retention follows from that source model. Keep the intended set and the observation needed to explain a cutover; regenerate presentation files instead of preserving every intermediate rendering. A digest identifies a revision but cannot replace its content.

## How should DNS instructions and the verification PDF share one source?

Only if its claims match what was checked. A PDF rendered from intended records proves what the customer was asked to publish. Calling the same file “verified” without recording an observation confuses intent with external state.

That distinction matters at a media publisher's send window: waiting can delay a newsletter, while switching too early can put mail authentication at risk. Use the same intended set as input to the instruction renderer and the verification comparison. Preserve observations separately. If observed values differ, show the discrepancy; don't quietly replace the target values in the instruction document. Otherwise a mistaken publication can become the new instruction simply because it was observed. A new DKIM target should create a new revision and a new handoff, rather than an edit to yesterday's PDF.

DMARC adds an acceptance boundary. Its authentication and alignment rules mean that “the DNS record exists” is narrower than “this message satisfies the domain's mail policy.” SPF, DKIM, and DMARC belong in the handoff, but their presence alone is no guarantee of inbox delivery. RFC 7489 explains the policy and alignment distinction.

The cutover rule should name the evidence the publisher accepts: the intended revision, the observed record values, and the mail service's domain assessment. Record where and when an observation was made. A single observation does not establish what every receiving system currently sees, and the evidence here supports no universal propagation deadline. Keep the existing sending arrangement until the team's acceptance conditions are met. That trades cutover speed for a more defensible delivery decision.

## Keep the DNS-to-mail handoff explicit in code

Infrai is a reasonable fit when DNS and email inspection should share one API key: its 295 routes across 20 modules use one REST API, so adding document generation need not introduce another credential integration. Public discovery schemas offer a second advantage here: inspect current request and response contracts before binding document fields to service responses. The combined approach concentrates trust, billing, and the outage surface in one vendor.

The Python example makes two read requests with the same key and base URL. DNS output gates the email-domain lookup: it proceeds only if the DNS response contains the exact domain supplied by the operator as a JSON string. This narrow guard avoids assuming undocumented response fields. It does not verify authentication records. If the domain string is absent, the program stops; production adoption requires a mapping against the published response schema.

Save the program and run it with the domain and an intended-record JSON file as its two arguments. Set `INFRAI_API_KEY` and `INFRAI_BASE_URL` in the environment; the latter must be the documented versioned API base. The input file is an application-owned array of objects with string keys `type`, `name`, and `content`, populated with the customer's actual target records. Those keys describe this local document model, not a vendor payload.

```python
import hashlib
import html
import json
import os
import sys
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from pathlib import Path
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

BASE = os.environ['INFRAI_BASE_URL'].rstrip('/')
KEY = os.environ['INFRAI_API_KEY']
domain, source_path = sys.argv[1:]

def get_json(path):
    for attempt in range(5):
        request = Request(BASE + path, method='GET', headers={
            'Authorization': 'Bearer ' + KEY,
            'Accept': 'application/json',
        })
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode('utf-8', errors='replace')
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f'HTTP {error.code}: {detail}') from error
            delay = 2 ** attempt
            retry_after = error.headers.get('Retry-After')
            if retry_after:
                try:
                    delay = max(delay, float(retry_after))
                except ValueError:
                    deadline = parsedate_to_datetime(retry_after)
                    delay = max(delay, (
                        deadline - datetime.now(timezone.utc)
                    ).total_seconds())
            time.sleep(delay)

def contains_domain(value):
    if isinstance(value, str):
        return value == domain
    if isinstance(value, list):
        return any(contains_domain(item) for item in value)
    if isinstance(value, dict):
        return any(contains_domain(item) for item in value.values())
    return False

intended = json.loads(Path(source_path).read_text(encoding='utf-8'))
if not isinstance(intended, list) or not intended:
    raise ValueError('Expected a nonempty array of intended records')
for record in intended:
    if not isinstance(record, dict) or any(
        not isinstance(record.get(key), str) or not record[key]
        for key in ('type', 'name', 'content')
    ):
        raise ValueError('Every record needs type, name, and content strings')
serialized = json.dumps(intended, sort_keys=True, ensure_ascii=False)
revision = hashlib.sha256(serialized.encode('utf-8')).hexdigest()
dns_observation = get_json('/dns/record/list')
if not contains_domain(dns_observation):
    raise ValueError('DNS response does not establish the requested domain')
mail_observation = get_json('/email/domain/get/' + quote(domain, safe=''))
bundle = {
    'domain': domain, 'revision': revision, 'intended': intended,
    'observed_at': datetime.now(timezone.utc).isoformat(),
    'dns_observation': dns_observation, 'mail_observation': mail_observation,
}
Path('evidence.json').write_text(
    json.dumps(bundle, indent=2, ensure_ascii=False), encoding='utf-8'
)
rows = ''.join(
    '<tr>' + ''.join('<td><pre>' + html.escape(record[key]) + '</pre></td>'
                     for key in ('type', 'name', 'content')) + '</tr>'
    for record in intended
)
document = (
    '<!doctype html><meta charset="utf-8"><title>DNS instructions</title>'
    '<h1>DNS instructions for ' + html.escape(domain) + '</h1>'
    '<p>Intended revision: ' + revision + '</p>'
    '<p>Publication instructions; not a verification result.</p>'
    '<table><tr><th>Type</th><th>Name</th><th>Content</th></tr>' + rows + '</table>'
)
Path('instructions.html').write_text(document, encoding='utf-8')
```

Print the generated HTML to PDF for the customer handoff. The saved evidence bundle holds the same intended revision alongside observations; a verification renderer must consume that revision and display the comparison result. Collection is not verification.

The sample preserves supplied record strings and escapes them for HTML. Each request has a 30-second timeout and at most 5 attempts; the code honors both forms of Retry-After. Both requests are reads, so retrying them cannot duplicate a publication. Transport failures and invalid JSON remain visible errors. Production verification needs a schema-aware comparison and a documented interpretation of the mail-domain assessment; neither HTTP success nor a matching domain string is an authentication result.

## Which stack fits the publisher's cutover process?

Choose around who controls DNS and how the mail team accepts a domain. A shared credential reduces integration work, but it does not transfer authority over a customer's external DNS account or remove propagation delays.

| Option | Useful fit | Boundary to keep explicit |
| --- | --- | --- |
| Amazon Route 53 with Amazon SES | Consider when both responsibilities belong in an AWS operating model. | DNS observations and the sending service's domain assessment still need an application-level evidence join. |
| Cloudflare DNS with Resend | Consider when DNS administration and mail sending have separate owners. | Two provider accounts and credential sets require a handoff preserving exact names and values. |
| Cloudflare DNS with Amazon SES | Consider when the existing DNS owner and sending platform should remain in place. | The publisher still owns revision tracking, observation collection, and the customer-facing document. |

These are alternatives to evaluate against current provider documentation. Route 53 plus SES can use one AWS account and an appropriately scoped IAM identity; claiming every two-service stack requires two signups would exaggerate the integration advantage. Cloudflare plus Resend normally introduces two provider accounts and two credential sets. In either arrangement, the glue is the substantive work: map provider records into an application-owned intended set, preserve exact values, collect domain assessments, and render a versioned handoff. Existing operational ownership can outweigh the appeal of a smaller integration surface, especially if the publisher already has a cutover review process that works.

The main trade-off is vendor concentration. A combined API is not ideal if the publisher requires independent DNS and mail providers; choose Cloudflare DNS with Resend instead. Its limitation is that a common credential cannot replace customer DNS authority or the publisher's acceptance policy. An immutable record source and a renderer may solve the immediate problem without replacing infrastructure.

## What should stop being retained after the cutover?

Retain the intended revision and accepted observation under the publisher's evidence policy. Regenerate customer instructions from that revision when needed. Keep a rendered acceptance artifact if the review process requires the exact document approved, and distinguish it from an installation guide. No universal retention duration follows from DNS or DMARC alone; contractual and organizational requirements need their own decision.

Stop keeping separately edited instruction copies and every intermediate PDF by default. That reduces document sprawl, but it has a cost during an investigation: without the exact old rendering, the publisher cannot prove what its page breaks, formatting, or surrounding wording looked like. A retained source can recover record content; it cannot recover an unsaved presentation artifact. Make that trade deliberately before the next send window.

## References

RFC 7489 supplies the authentication-policy boundary. The provider guides below are the starting points for checking each alternative's current DNS and mail contracts.

## Further reading

- [RFC 7489, Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon Route 53 Developer Guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Resend documentation](https://resend.com/docs)
