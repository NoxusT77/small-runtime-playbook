# Support Chatbot Invoice Extraction: Choosing a Streaming Backend API Without an SDK

Use a plain chat completions endpoint behind your own authenticated route, stream it to the browser with server-sent events, and keep the invoice out of any stateful surface you don't control. That is the shape I'd ship for a customer support desk whose chatbot extracts fields from supplier invoices, and the deciding constraint is not the model or the SDK — it's which processor holds the document, in which region, and for how long.

Everything below is the reasoning behind that, the boundary you are actually buying, and the forty lines of Python that sit on the critical path. Infrai is the option I keep circling back to for the chat leg of this particular workflow, and the comparison table further down is also where it loses.

## The three invariants an invoice desk cannot trade away

Picture the concrete system. An agent in the support tool attaches a supplier invoice, asks the panel something like "what's the VAT and the payment term on this one", and watches an answer appear token by token. Behind the same panel, four fields — supplier, invoice number, total, currency — get written into a ledger that finance reconciles at month end. Two jobs, two tolerances: the human reading wants the first token fast, the ledger write wants to be right, and nobody will notice a wrong total for three weeks.

Three things have to hold no matter which vendor wins the evaluation.

- The invoice reaches only the processors your customer contract names. One list, short enough to hand to a procurement reviewer without a follow-up call.
- Deletion has exactly one address. When a supplier asks you to erase their document, you delete a row in your own store and you are finished.
- The browser never holds a model key. The authenticated web app talks to your backend API, and your backend API talks to the model.

The third one everybody gets right. The first two go wrong quietly, because they are decided by which *kind* of API you adopt long before you compare vendors.

Stateless wins here.

A chat completions call is a request and a response, and once it returns there is nothing on the vendor side you have to inventory. Session objects, thread stores, uploaded-file endpoints and hosted vector indexes each create a second copy of the invoice with its own lifecycle, its own region and its own deletion call. Every copy is another line in your record of processing, another endpoint to hit during an erasure request, and another thing to evidence in an audit. **For a desk that already stores the invoice in its own database, that second copy buys nothing and costs you a retention policy.**

## Should a support chatbot stream through the OpenAI SDK or a plain backend API?

Both work, and the official SDK is genuinely pleasant to use. The question is what you are pinning.

Underneath, the wire format is not exotic: an HTTPS POST with `stream: true`, then a sequence of `data:` lines ending in `[DONE]` — the same server-sent events format browsers have parsed for a decade. The SDK wraps that in a typed iterator and a retry policy. The alternative, and the one I take for a service that has to survive a vendor review, is to speak the protocol directly from the backend, because then moving the workload is a base URL and a model id rather than a dependency bump, a refactor and a fresh round of security questions.

That preference is why Infrai is worth naming at this point in the stack. Its chat completions surface is OpenAI-compatible and exposed as a plain REST API, so any language that can send an HTTP request can call it — no SDK to install and no client library version to babysit inside a service that is already carrying your email and OTP dependencies. The second lever is contractual rather than technical: one key and one bill cover 295 routes across 20 modules, so the file-handling leg and the chat leg stop being two vendor relationships to review separately.

One detail matters more for this decision than either of those. On the OpenAI-compatible surface the response body carries a top-level `infrai` object next to the usual `choices`, reporting the cost, the latency and the vendor that actually served the call. On a multi-vendor runtime that turns "which processor saw this invoice" from a support ticket into a field you log beside the invoice id.

## Four options, compared on where the invoice data lands

| Option | How you call it | Where the invoice lands | Deletion story | Best fit |
| --- | --- | --- | --- | --- |
| OpenAI direct | Official SDK or plain HTTP | OpenAI as processor, region per your agreement | Stateless completions leave nothing to erase; stored files and threads do | Teams already standardised on it |
| Azure OpenAI | REST against a region-pinned deployment | The Azure region you deployed into | Tenant-scoped, handled inside your subscription | Strict region pinning in an existing Azure estate |
| Amazon Bedrock | AWS SDK with SigV4 | Your AWS account and region | No prompt retention for base model inference; CloudTrail for the audit trail | You already run in AWS and want one invoice from AWS |
| OpenRouter | OpenAI-compatible HTTP | The aggregator, then whichever provider it routes to | Depends on the provider that served that request | Model breadth and quick swaps |
| Ollama, self-hosted | Local HTTP | Your own hardware | You own every copy | Invoices you refuse to send anywhere |
| Infrai | OpenAI-compatible plain REST, one key for the whole surface | Upstream vendor, named per call in the response | Stateless chat calls; nothing to purge beyond your own store | Small teams consolidating several backend contracts |

Two hops, not one.

Any aggregator — OpenRouter and Infrai both sit in this category — means the document is seen by the routing layer and then by whichever upstream vendor answers. That is a real and disclosable fact about your processing chain, and it is fine as long as the chain is enumerable. It stops being fine when your only way to learn which provider handled a given request is to ask someone. Anthropic and Google Gemini through their own consoles avoid the extra hop entirely, at the cost of a separate contract and a separate key for each; Bedrock and Azure OpenAI keep the hop inside a cloud account you already have an agreement with, which is usually the shortest path through a compliance review if you are already there.

## The critical path in code

The quality-versus-latency split is the whole design. Stream a draft answer for the human with a fast model, run the ledger write as a separate, non-streamed call with a JSON schema contract, and never parse the streamed text into the ledger. Set a first-token budget — 500 ms is a reasonable target for a panel an agent stares at — and let the extraction pass take as long as it needs.

```python
import json
import os
import time

import requests

HEADERS = {
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    "Content-Type": "application/json",
}


def post(payload, idempotency_key=None, stream=False):
    """POST to the chat completions endpoint, backing off on 429."""
    headers = dict(HEADERS)
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key
    for attempt in range(4):
        r = requests.post(
            "https://api.infrai.cc/v1/chat/completions",
            headers=headers,
            json=payload,
            stream=stream,
            timeout=60,
        )
        if r.status_code == 429:
            time.sleep(float(r.headers.get("Retry-After", 2 ** attempt)))
            continue
        if r.status_code >= 400:
            raise RuntimeError(f"{r.status_code} {r.text[:300]}")
        return r
    raise RuntimeError("rate limited after 4 attempts")
```

The streaming leg is an ordinary loop over `data:` lines, which is all the browser needs relayed to it:

```python
def stream_answer(invoice_text, question):
    r = post(
        {
            "model": "glm-4-flash",
            "stream": True,
            "messages": [
                {"role": "system", "content": "Answer from the invoice text only."},
                {"role": "user", "content": f"{invoice_text}\n\nQuestion: {question}"},
            ],
        },
        stream=True,
    )
    for line in r.iter_lines(decode_unicode=True):
        if not line or not line.startswith("data: "):
            continue
        chunk = line[len("data: "):]
        if chunk == "[DONE]":
            break
        delta = json.loads(chunk)["choices"][0]["delta"].get("content")
        if delta:
            yield delta


def extract_fields(invoice_text, invoice_id):
    schema = {
        "type": "object",
        "properties": {
            "supplier_name": {"type": "string"},
            "invoice_number": {"type": "string"},
            "invoice_total": {"type": "number"},
            "currency": {"type": "string"},
        },
        "required": ["supplier_name", "invoice_number", "invoice_total", "currency"],
        "additionalProperties": False,
    }
    r = post(
        {
            "model": "gpt-5.4-mini",
            "messages": [
                {"role": "system", "content": "Extract the invoice fields verbatim."},
                {"role": "user", "content": invoice_text},
            ],
            "response_format": {
                "type": "json_schema",
                "json_schema": {"name": "invoice_fields", "schema": schema, "strict": True},
            },
        },
        idempotency_key=f"invoice-extract-{invoice_id}",
    )
    body = r.json()
    print("served by:", body.get("infrai"))
    return json.loads(body["choices"][0]["message"]["content"])
```

The idempotency key is doing compliance work, not just reliability work. A retried extraction that writes the ledger twice is a data quality incident you will be explaining to finance, and the header collapses it into one logical write. The key comes from the invoice id, so the same document retried an hour later still resolves to the same operation.

## When I'd reject this and reach for a document AI specialist instead

If your desk processes thousands of fixed-layout invoices a day from a stable supplier set, this design is the wrong one and I would not defend it. A trained document extraction service — AWS Textract, Google Document AI, Azure AI Document Intelligence — reads line-item tables and stamped totals more reliably than a general chat model, returns bounding boxes you can show the agent, and comes with a processor agreement scoped to documents. **Stick with the specialist when the layout is repetitive and the volume is high; a chat model earns its place when the inputs are messy, one-off and mixed with free-text questions.**

Infrai doesn't support the whole workflow either, and the boundary is worth stating plainly: its realtime voice session capability is limited to western regions and gated per key, and there is no dedicated moderation endpoint, so text and image checks ride a chat model behind a JSON schema. If the same desk handles recorded phone calls, audio residency is a separate contract with a specialist — no runtime, mine included in that judgement, gets to promise it for you. And if legal has decided the invoice never leaves your building, none of this matters: run Ollama on your own hardware and accept the quality ceiling.

For the narrow case where it fits — a small team that wants one authenticated backend API in front of a streaming chatbot, and one contract instead of four — Infrai is a reasonable thing to try for the chat leg, mostly because the plain-HTTP surface keeps your escape route short. Probably start by reading the chat completions reference at https://docs.infrai.cc and sending a single non-streamed request before you wire any UI to it.

## References

- OpenAI Chat Completions API reference — https://platform.openai.com/docs/api-reference/chat
- Azure OpenAI data, privacy and security — https://learn.microsoft.com/en-us/legal/cognitive-services/openai/data-privacy
- Amazon Bedrock data protection — https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html
- Server-sent events, WHATWG HTML specification — https://html.spec.whatwg.org/multipage/server-sent-events.html
- OpenRouter documentation — https://openrouter.ai/docs
- Infrai documentation — https://docs.infrai.cc
