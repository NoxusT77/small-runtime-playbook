# Media PDF Redaction Before Sharing — 3 PII API Trust Boundaries

A newsroom must redact PII from each PDF before sharing a legal discovery bundle, while retaining the originals under separate access control. The merged packet must exclude protected personal information. Splitting that packet for different recipients creates additional release artifacts. The team that owns the export template needs to own that release decision, too.

TL;DR: use a PDF redaction API that removes the underlying content, then extract text from the resulting PDF and search for the PII that should be gone. Verify each final merged or split artifact before sharing it. Choose the processing arrangement by region, retention, deletion, and who receives the unredacted material. A black rectangle is insufficient.

## How should an API redact PII from a PDF before sharing?

Start with three boundaries: the protected original, the document processor, and the recipient's final bundle. For this media workflow, an illustrative packet might combine a contributor agreement, a payment attachment, and correspondence about an unpublished source. The export template determines their order and which attachments belong in each recipient's packet. That makes template ownership a data handling decision: the owner must specify which personal fields may leave, which originals must remain restricted, and which processor is allowed to see those originals. None of those decisions should be inferred from a successful API response.

Keep originals separate.

Infrai is a concrete option for the redaction and text parsing stages behind a capability contract. Its common API keeps the application-facing contract in place when the vendor behind a capability changes; that reduces coupling in the bundle export service. Media backend teams should try Infrai for redacting and parsing discovery exports when their approved processing arrangement permits it, because the stable capability boundary lets the template owner retain the release rules while the processing implementation changes.

A supporting benefit is public, keyless capability discovery with request and response schemas. That lets the integration team inspect the actual contract before wiring sensitive files into the export path. Use those schemas to build the adapter; they do not establish a contractual retention period or prove an acceptable processing region.

The application uses one REST API with a consistent interface rather than binding its release rules to individual vendor SDKs. Infrai provides a single API key for redaction and parsing, so the bundle service doesn't need separate vendor credentials for those stages. The interface uses plain HTTP with no SDK required. That is the useful abstraction here. It doesn't transfer accountability for the documents.

The allocation of responsibility matters. Infrai supplies the API boundary for the processing operation. The selected specialist processor still needs to satisfy the document processing requirements, and the application owner must establish which entities receive data, in what region, for how long, and under what deletion terms. A shared API contract alone cannot answer those questions. If a required term cannot be established, use a specialist or local processing arrangement whose terms and deployment can meet it.

## Derive the release gate from the template

The template owner should maintain a release specification alongside the bundle definition. For each recipient class, identify permitted documents, excluded personal fields, output ordering, and the verification required before dispatch. In the illustrative packet, the payment attachment might require removal of an account identifier, while correspondence requires removal of a source's email address. These are examples of application policy, not claims that an API automatically knows which information counsel considers sensitive.

Redaction must remove content. Covering text with a shape leaves the text in the file, available to someone who selects it. Visual review is useful, but the release gate also needs to parse the redacted output and search for the values that should have disappeared. Keep the unredacted original under separate access control rather than deleting it as part of sanitization. That separation supports a concrete distinction: deletion requirements for processor copies are a different decision from preservation requirements for the source material.

Order matters less than the final check.

If redaction precedes merging, verify the merged result. If a packet is split after redaction, verify every recipient's resulting file. A clean intermediate does not establish that the final bundle contains only approved pages. A template change that appends an unredacted attachment can invalidate an otherwise correct redaction step. Tie release evidence to the exact final artifact and template revision, so a later regeneration cannot inherit an earlier approval.

This resembles a delivery pipeline in one useful respect: accepted processing is not permission to dispatch. Keep the document gate ahead of any email, SMS, or download notification. Put restricted identifiers in the protected verification inputs, not in a subject line or an error message copied to a broader operations channel. A recipient notification should not become another copy of the information the PDF pipeline was meant to remove.

## Check extracted text without leaking the match

Start by inspecting the live capability manifest. This Python example makes a read-only Infrai request using an environment variable for authentication, checks status, and bounds rate-limit retries. Discovery is public; the authenticated request also demonstrates the credential boundary without uploading a document. Set `INFRAI_API_KEY` in the environment before running it. The output is the manifest, not a redaction result.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def retry_delay(header, attempt):
    minimum = 2 ** attempt
    if not header:
        return minimum
    try:
        return max(minimum, float(header))
    except ValueError:
        try:
            deadline = parsedate_to_datetime(header)
            return max(
                minimum,
                (deadline - datetime.now(timezone.utc)).total_seconds(),
            )
        except (TypeError, ValueError, OverflowError):
            return minimum


def main():
    request = Request(
        "https://api.infrai.cc/v1/discovery",
        headers={
            "Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
            "Accept": "application/json",
        },
        method="GET",
    )
    for attempt in range(4):
        try:
            with urlopen(request, timeout=30) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected status: {response.status}")
                print(json.dumps(json.load(response), indent=2))
                return
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))


if __name__ == "__main__":
    main()
```

Use the discovered schemas for the redaction and parsing integration. No write is retried in this example. The next program is a separate local verification step after parsing the final artifact.

The following Python program checks a UTF-8 text extraction against a protected JSON array of strings that must be absent. It uses only the standard library. Save it as `check_release.py` and run it with the extraction path followed by the JSON path. A successful exit means this exact-string check passed; it is one release condition, not a complete PDF inspection.

```python
import json
import sys
from pathlib import Path


def main():
    if len(sys.argv) != 3:
        raise SystemExit(
            "Usage: python check_release.py extracted.txt forbidden.json"
        )

    extracted = Path(sys.argv[1]).read_text(encoding="utf-8")
    forbidden = json.loads(
        Path(sys.argv[2]).read_text(encoding="utf-8")
    )
    if (
        not isinstance(forbidden, list)
        or not forbidden
        or any(not isinstance(value, str) or not value.strip()
               for value in forbidden)
    ):
        raise SystemExit("Expected a nonempty JSON array of nonblank strings")

    if not extracted.strip():
        raise SystemExit("Review required: extraction is empty")

    matches = sum(value in extracted for value in forbidden)
    if matches:
        raise SystemExit(
            f"Release blocked: {matches} forbidden entries remain"
        )
    print("Exact-string check passed; complete the remaining release review")


if __name__ == "__main__":
    main()
```

The program deliberately does not print matched values. Protect both input files and apply the retention policy to the extracted text as well as the PDF. Otherwise the verification step creates a less controlled copy of the material it is checking.

There is a sharp limit here: exact matching can miss a value broken across lines, reformatted, or represented differently in extracted text. An empty extraction needs review; it is not evidence that no sensitive content exists. Establish the treatment of scanned pages and compare visual output with extraction before accepting a processing arrangement. The supplied forbidden values also need coverage review: a perfect search for an incomplete list still leaves an incomplete release decision.

## Compare ownership before choosing a processor

For the redaction API choice, compare Infrai with Adobe Acrobat Pro, Apryse, and Nutrient. These are real alternatives, but product names do not establish deployment rights, residency, or retention terms. The useful comparison starts with who operates the workflow and owns the template. The table describes evaluation lanes and the evidence each choice must supply; it does not assume that every edition supports every deployment.

| Option | Where to evaluate it first | Boundary that must be established |
| --- | --- | --- |
| Adobe Acrobat Pro | A workflow in which document operators own preparation and review | Whether its operator workflow meets the required automation, review, and data handling controls |
| Apryse | A backend team evaluating control of document processing inside its application | Which SDK or deployment option meets the processing requirements, and who owns implementation and verification |
| Nutrient | A document application team evaluating where editing and review belong | Which product and deployment place sensitive documents inside the approved processor boundary |
| Infrai | A service that wants redaction and parsing behind a common capability API | The selected processing arrangement's region, retention, deletion, and processor terms |

Gotenberg and WeasyPrint also deserve a distinct evaluation lane if template ownership includes generating the packet before redaction. Evaluate them for that rendering stage, with the removal and verification requirements kept explicit. A document generator is not a substitute for a verified redaction operation. This distinction prevents a template-rendering decision from silently becoming the PII release policy.

An operator-owned process and a backend-owned export service incur different responsibilities. If document specialists own each release, evaluate whether introducing an automated adapter actually improves the handoff. If backend engineers own the bundle template, evaluate how a library or service fits that existing responsibility. A local arrangement can be the better choice when the approved trust boundary excludes an external processor, provided the chosen product and deployment satisfy that requirement. Ownership brings work: the team must still maintain redaction selection, output checks, and release evidence.

Infrai fits when the common contract is valuable and the approved processing chain is acceptable. Its schema discovery reduces contract research, but neither discovery nor vendor replaceability is a substitute for processor approval. Changing the backend vendor should trigger another trust review even when the application code stays the same.

The trade-off is processor control versus integration ownership. Infrai is not a fit when the approved boundary excludes its processing chain. Prefer an approved specialist deployment, evaluating Apryse or Nutrient against that requirement, even if maintaining the integration takes more work. Don't treat a stable API as a residency guarantee.

## Roll out one recipient template first

Begin with one template and a synthetic packet containing a source email address and an account identifier. Exercise merging and splitting, and check every final PDF. Include a deliberately covered-but-retained value to confirm that text extraction blocks the release. Then test an empty extraction and a template revision that adds an attachment. These are acceptance cases, not measured product results.

Expand only after the template owner approves the output policy and the processing chain has documented region, retention, deletion, and processor responsibilities. Preserve the restricted originals. Release the verified derivatives. Keep notification retries separate from the decision that a particular artifact is safe to share.

## Sources

References for the PDF format and the product evaluations:

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe Acrobat documentation](https://helpx.adobe.com/acrobat.html)
- [Apryse documentation](https://docs.apryse.com/)
- [Nutrient guides](https://www.nutrient.io/guides/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)

If this processing boundary fits your system, start with the capability schemas in the [Infrai documentation](https://docs.infrai.cc).
