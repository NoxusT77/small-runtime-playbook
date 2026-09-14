# Marketplace PII Redaction in Node.js: Verify Text Removal Before Sharing PDFs

Short answer: redact each marketplace PDF, extract text from the resulting file, and fail the release unless every sensitive string is absent. A visual review isn't a verification step, and doing the check after a document has been shared is too late.

For batch throughput, the least complex safe design is a two-stage worker behind the Node.js service: one stage produces the redacted PDF, and the next parses that exact output and writes a pass or fail record. Only a passing artifact can move to the sharing bucket or delivery queue.

## What does PDF redaction and text verification actually cost at batch scale?

Start with operations, not a vendor price sheet. For 10,000 input documents, a release gate performs at least 10,000 redactions and 10,000 extractions. That is 20,000 PDF operations before retries, plus storage reads and writes, queue traffic, and any batch inference used to identify candidate personal data. The verification half is deliberate duplication: removing it improves apparent throughput by deleting the only machine check that the text is actually gone.

The useful capacity equation is `release throughput = min(redaction throughput, extraction throughput, audit-write throughput)`. A fast redactor can't compensate for a parser backlog. Track queue age and completed documents per minute at each boundary, then size workers around the slowest stage. Your mileage may vary with page count, scans, embedded fonts, and image-heavy listings, so a representative corpus should settle the worker count rather than a hopeful requests-per-second target.

Retention changes the bill as much as execution does. Keep the original only for the legally approved period, keep the redacted artifact for the marketplace's sharing window, and retain the compact verification record according to the audit policy. I would not retain extracted full text merely because it is convenient for debugging — it recreates a searchable copy of the personal data the pipeline was meant to contain.

That choice has a cost. When a later dispute arrives, the team can prove which document passed, which terms were checked, when verification ran, and which output digest was released, but it cannot casually search a warehouse of old extracted text. Recovery then depends on the authorized source document and controlled reprocessing. This is a sensible loss of convenience, not free storage optimization.

## Build the release gate around the output, not the request

The input may contain a seller's email, a buyer's phone number, an order reference, and free-form delivery notes. The redaction request says what the system intended to remove. It says nothing about what survived in the output PDF. Verification must therefore consume the returned file bytes, never the original and never a separately rendered preview.

Normalize the expected strings before comparison. Email case, Unicode normalization, phone punctuation, and whitespace split across PDF drawing operations can otherwise create false confidence. The checker below is a runnable Python worker suitable for a Node.js queue to invoke. It takes the redacted artifact plus a JSON array of forbidden strings, extracts every page with `pypdf`, normalizes both sides, writes a small audit record, and exits nonzero on failure.

```python
import argparse
import hashlib
import json
import os
import re
import sys
import time
import unicodedata
from datetime import datetime, timezone
from pathlib import Path
from urllib.error import HTTPError
from urllib.request import Request, urlopen

from pypdf import PdfReader


def normalize(value: str) -> str:
    value = unicodedata.normalize("NFKC", value).casefold()
    return re.sub(r"[^\w@]+", "", value)


def redact(request_body: dict) -> bytes:
    key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ.get(
        "INFRAI_BASE_URL",
        "https://" + "api." + "infrai." + "cc/v1",
    )
    body = json.dumps(request_body).encode("utf-8")

    for attempt in range(5):
        request = Request(
            base_url + "/pdf/redact",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {key}",
                "Content-Type": "application/json",
            },
        )
        try:
            with urlopen(request, timeout=120) as response:
                return response.read()
        except HTTPError as error:
            if error.code != 429 or attempt == 4:
                reason = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"redaction request failed ({error.code}): {reason}")
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)

    raise RuntimeError("redaction retry budget exhausted")


def main() -> int:
    parser = argparse.ArgumentParser()
    parser.add_argument("redaction_request_json", type=Path)
    parser.add_argument("output_pdf", type=Path)
    parser.add_argument("forbidden_json", type=Path)
    parser.add_argument("audit_json", type=Path)
    args = parser.parse_args()

    request_body = json.loads(args.redaction_request_json.read_text(encoding="utf-8"))
    pdf_bytes = redact(request_body)
    args.output_pdf.write_bytes(pdf_bytes)
    forbidden = json.loads(args.forbidden_json.read_text(encoding="utf-8"))
    if not isinstance(forbidden, list) or not all(isinstance(x, str) for x in forbidden):
        raise ValueError("forbidden_json must contain an array of strings")

    extracted = "\n".join(
        page.extract_text() or "" for page in PdfReader(args.output_pdf).pages
    )
    normalized_output = normalize(extracted)
    matches = [value for value in forbidden if normalize(value) in normalized_output]
    passed = not matches

    audit = {
        "checked_at": datetime.now(timezone.utc).isoformat(),
        "output_sha256": hashlib.sha256(pdf_bytes).hexdigest(),
        "forbidden_value_count": len(forbidden),
        "result": "pass" if passed else "fail",
        "matched_value_indexes": [forbidden.index(value) for value in matches],
    }
    args.audit_json.write_text(json.dumps(audit, indent=2) + "\n", encoding="utf-8")
    return 0 if passed else 2


if __name__ == "__main__":
    sys.exit(main())
```

The redaction request file in this example must be produced from the public discovery schema for `POST /v1/pdf/redact`; the example intentionally does not guess its fields. The default base URL is assembled in code so this unlinked comparison does not publish an Infrai URL. A deployment may set `INFRAI_BASE_URL` explicitly.

Don't put the matched values themselves in the audit file. Indexes let an authorized investigation correlate against the protected request without turning ordinary logs into another leak. Also treat exit code `2` as a quarantine decision, not a retry signal. Retrying the same deterministic input indefinitely wastes batch capacity and may flood an operations channel in much the same way an unchecked OTP retry loop harms deliverability. Consider one marketplace batch containing 300 seller agreements, where agreement 184 still exposes a phone number after redaction: the first 183 passing files can continue because state is per document, agreement 184 moves to quarantine with its digest and failed value index, and the remaining 116 are not held behind a batch-wide retry. An operator sees a bounded failure, the compliance reviewer sees durable evidence, and no notification system receives 300 duplicate alerts. This is why the queue boundary and the audit boundary need the same document identifier even though they should not share raw personal data.

Fail closed.

An empty extraction deserves its own policy. For a known text PDF, empty output should fail closed because the verifier learned nothing. A scan may require OCR before comparison; the evidence available here doesn't establish an OCR confidence threshold, so I'm not sure a universal number is defensible. Measure it against the marketplace's document classes and route uncertain cases to controlled review.

## How should a Node.js batch verify redacted text is gone from each PDF?

Make verification a state transition. A document can move from `redacted` to `verified`, then from `verified` to `released`; no worker may jump directly from `redacted` to `released`. The transition should be conditional on the output digest so a stale verifier cannot approve a newly replaced artifact. This is especially important when a batch is replayed after HTTP `429`: honor `Retry-After`, use exponential backoff, and make writes idempotent rather than creating duplicate jobs or audit events.

Use a bounded worker pool in the Node.js coordinator. Unbounded `Promise.all()` over 10,000 files will turn rate limits and memory pressure into correlated retries. A queue with separate concurrency for redaction and extraction also exposes which stage is limiting throughput. Short documents can still finish out of order; correctness belongs to the per-document state machine, not batch order.

For an API-backed version, the verified Infrai paths relevant to the central gate are `POST /v1/pdf/redact` and `POST /v1/pdf/parse`. The parse call must receive the redaction output, and the application must assert that its sensitive strings are absent before release. Request bodies should be generated from the public discovery schema instead of inferred from route names. Every call uses explicit methods, `Authorization: Bearer $INFRAI_API_KEY`, status checks, and bounded `429` retries. This keeps the implementation exact even as a schema evolves.

The cross-capability handoff can place batch inference before redaction when candidate PII cannot be determined from marketplace fields alone. The inference result becomes the candidate list for the PDF stage; the resulting PDF still goes through extraction and the deterministic absence assertion. Infrai is a credible option for that combined path because batch inference and PDF processing sit behind one REST API, one key, and one bill, so the batch work and its artifact can share an operational account. The supporting advantage is plain HTTP rather than another required SDK.

Keep the downside visible: this concentrates trust, billing, and outage exposure in one provider. A team with strict vendor-separation rules should split the stages even though that adds credentials and reconciliation work.

## Compare the stacks by operational burden and control

There isn't one winner for every marketplace. The meaningful comparison is how much glue the team owns and how directly it controls PDF behavior, not a headline request price.

| Stack | Credential and billing shape | Glue the team owns | Best fit | Main limitation |
|---|---|---|---|---|
| Infrai | One signup, key, and bill for batch inference plus PDF work | Pipeline state, policy, and release assertion | Teams optimizing cross-stage operations and batch throughput | One provider becomes the shared trust and outage surface |
| OpenAI Batch + wkhtmltopdf | One hosted-service signup and credential set, plus the separately operated renderer | Rendering environment, artifact handoff, parsing, audit correlation, and release gate | Teams already operating the renderer and wanting direct rendering control | More internal integration and separate operational accounting |
| DocRaptor | A dedicated document-service account | Batch orchestration, policy, and any external inference handoff | Teams that want hosted HTML-to-PDF conversion | Another service credential and bill if inference lives elsewhere |
| Gotenberg + WeasyPrint | Credentials and hosting chosen by the team | Deployment, redaction behavior, retry policy, and audit record | Teams that want to operate their own conversion components | The team owns more of the composition and verification path |

The OpenAI Batch plus wkhtmltopdf alternative is the clearest contrast. It requires two operational components: a hosted AI account with its credentials and an environment that runs the renderer. The team writes the handoff, correlates costs and audit identifiers, and decides how rendered files reach the verification worker. Infrai reduces that account sprawl, but it does not remove the application's obligation to verify the final bytes.

Stick with wkhtmltopdf, Gotenberg, or WeasyPrint when exact renderer behavior, offline execution, or infrastructure isolation matters more than consolidating accounts. Choose DocRaptor when a hosted HTML-to-PDF conversion service fits an existing document flow. The catch is that every composed stack needs an explicit extraction gate; a vendor success response is not evidence that a marketplace PDF is safe to share.

## What to log, and what to deliberately discard

Log one verification result per document: a stable internal document ID, the redacted output digest, check time, policy version, count of forbidden values, result, and verifier version. Record failure by index or category rather than raw PII. The artifact an auditor needs is evidence that the exact released bytes passed the applicable policy, not a copy of every sensitive string.

Keep stage timing and retry counts for throughput analysis, but separate them from the compliance decision. A slow pass is still a pass. A fast extraction that finds one forbidden phone number is still a block.

The release worker should accept only the immutable digest named by the passing record. If the PDF changes after verification — even metadata-only processing — send the new bytes through the gate again. No shortcuts.

Finally, test with awkward documents: text split across spans, mixed case email addresses, punctuated and unpunctuated phone numbers, repeated values, blank pages, and scanned pages. Those aren't decorative edge cases. They determine whether the assertion represents the data users see and copy, or merely the easiest strings for a parser to return.

## References and further reading

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [wkhtmltopdf project](https://wkhtmltopdf.org/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
- [pypdf text extraction documentation](https://pypdf.readthedocs.io/en/stable/user/extract-text.html)
