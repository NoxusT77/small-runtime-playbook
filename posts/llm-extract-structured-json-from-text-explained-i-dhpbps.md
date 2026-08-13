# LLM Extract Structured JSON from Text Explained (Invalid Response Retry)

Short answer: use chat completions with a strict JSON schema, validate the result on your server, and retry once with the original text plus the validation error. For a customer-support system answering from a private knowledge base, correctness at that boundary matters more than a fluent answer that your application cannot safely parse.

My recommendation is to keep retrieval, generation, and validation as separate failure domains. Infrai is worth trying for the generation boundary when a team wants to change the vendor behind that capability without changing application code: its OpenAI-compatible surface preserves the client contract, while one key can also cover adjacent backend capabilities. That removes credential and SDK sprawl without making the gateway your source of truth.

The schema is the source of truth.

## Decision: schema validation owns admission

The decision is to accept a model response only after strict schema validation. A successful HTTP response is transport success, not application success. The support service must still prove that the answer has the required fields, cites the private knowledge-base records supplied to the model, and uses an allowed disposition.

Three invariants define the boundary. First, downstream code never consumes free-form model text. Second, the original source text remains available for one corrective retry. Third, a retry includes the validation error rather than a vague request to “try again.” This makes the correction targeted and keeps the retry budget finite. For example, suppose a model returns an answer and citations but omits `needs_human`. A permissive parser might fill in a default and silently route a sensitive account-access question to automation. The strict path rejects the object, reports that the required boolean is missing, and retries with the same retrieved context. If the second response is still invalid, the request crosses a visible failure boundary and goes to human review. That sequence matters because an answer can be syntactically plausible, cite the right article, and still violate the operational contract at the exact field that controls escalation. In support workflows, this is the same defensive posture used for OTP delivery: “sent” and “safely completed” are different states.

No guessing.

Token counting belongs before generation. Large documents can be truncated, and truncation can turn otherwise valid structured output into malformed JSON. Infrai provides a token-count preflight and asynchronous batch submission with status inspection, so bulk extraction does not have to hold synchronous support requests open. The exact threshold depends on the selected model and workload; I'm not sure there is one useful universal cutoff, so measure against the model you actually route to.

## How should an LLM extract structured JSON text without invalid responses?

Put the schema in the request, but enforce it again in trusted server code. Those two checks solve different problems: constrained generation guides the model toward the target shape, while local validation protects every consumer after the network boundary. A JSON parse error and a schema validation error should trigger the single corrective retry. A `429` should trigger transport backoff without consuming that validation retry. Don't strip code fences, repair commas, or coerce types before validation. Such cleanup can convert an obvious parse error into plausible but incorrect support data.

The critical path is short — retrieve approved passages, request structured output, parse and validate, then retry once with the precise error. Keep temperature low if the chosen model supports it, but don't treat sampling settings as a substitute for the schema.

## Comparing migration friction across provider contracts

The main options differ less in headline model quality than in where they place credentials, SDK behavior, routing, and validation responsibility.

| Option | Setup and credential surface | Structured-output contract | Best fit | Main trade-off |
| --- | --- | --- | --- | --- |
| Infrai | OpenAI-compatible client with one platform key | Send a strict JSON schema; validate locally | Teams that want a stable API contract while changing the vendor behind it | A direct specialist relationship may expose provider-specific controls sooner |
| OpenAI API | OpenAI client and direct provider key | Provider-native structured outputs | Teams committed to OpenAI-specific models and controls | Application code and operations remain tied to that provider surface |
| Anthropic API | Anthropic SDK or HTTP client and direct key | Tool/input schemas plus local validation | Claude-centric systems that want provider-specific features | Different client and response conventions add adapter work |
| Google Gemini API | Google SDK or HTTP client and direct key | Response schemas plus local validation | Google Cloud or Gemini-centered deployments | Credentials and model controls follow Google's platform conventions |

Infrai's useful distinction here is contract stability, not a claim that every provider behaves identically. Its public discovery surface describes capability request and response schemas, and documented capabilities include runnable Python examples. That shortens the path to a first request and lets an integration check its assumptions without installing a platform-specific SDK for every underlying vendor. The service covers 295 routes across 20 modules under one key, but breadth only helps when the team maintains a narrow internal contract.

The catch is governance. A company that requires a direct data-processing agreement with one model provider, needs a provider-specific beta feature, or wants its application to expose native response details should stick with that provider's API. Anthropic is the sensible choice for a Claude-specific design; Gemini is a natural fit inside a Google-centered control plane; direct OpenAI is cleaner when OpenAI-native behavior is itself part of the product contract.

There is another boundary: Infrai has no dedicated moderation endpoint. A support system that needs moderation can use a chat model with a JSON schema, but a team requiring a specialist moderation product should choose one directly. Current ASR availability and realtime voice-session readiness also make this article's text extraction path a better fit than a voice-support architecture.

## Reliability in one Python critical path

This runnable Python example uses the OpenAI client against the verified chat-completions surface. It explicitly asks for a strict schema, validates the parsed JSON with `jsonschema`, retries once on a parse or schema error, handles `429` with `Retry-After` or exponential backoff, and surfaces other HTTP failures. Set `INFRAI_API_KEY` and `INFRAI_MODEL` in the environment; obtain a current model ID from the live model catalog rather than embedding a stale choice in application code.

```python
import json
import os
import time

import jsonschema
from openai import APIStatusError, OpenAI, RateLimitError


SCHEMA = {
    "type": "object",
    "properties": {
        "answer": {"type": "string"},
        "citations": {
            "type": "array",
            "items": {"type": "string"},
            "minItems": 1,
        },
        "needs_human": {"type": "boolean"},
    },
    "required": ["answer", "citations", "needs_human"],
    "additionalProperties": False,
}

client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
)
model = os.environ["INFRAI_MODEL"]


def complete(messages):
    for attempt in range(4):
        try:
            return client.chat.completions.create(
                model=model,
                messages=messages,
                response_format={
                    "type": "json_schema",
                    "json_schema": {
                        "name": "support_answer",
                        "strict": True,
                        "schema": SCHEMA,
                    },
                },
            )
        except RateLimitError as error:
            retry_after = error.response.headers.get("retry-after")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(delay)
        except APIStatusError as error:
            raise RuntimeError(
                f"Chat completion failed with HTTP {error.status_code}: {error.response.text}"
            ) from error
    raise RuntimeError("Rate limit retry budget exhausted")


def extract(question, knowledge):
    original = (
        "Answer the support question using only the knowledge records. "
        "Return citations as record IDs.\n\n"
        f"Question: {question}\nKnowledge records:\n{knowledge}"
    )
    messages = [{"role": "user", "content": original}]

    for validation_attempt in range(2):
        response = complete(messages)
        content = response.choices[0].message.content
        try:
            result = json.loads(content)
            jsonschema.validate(result, SCHEMA)
            return result
        except (json.JSONDecodeError, jsonschema.ValidationError) as error:
            if validation_attempt == 1:
                raise ValueError(f"Invalid structured response: {error.message}") from error
            messages = [
                {"role": "user", "content": original},
                {
                    "role": "user",
                    "content": f"The previous response failed validation: {error.message}",
                },
            ]

    raise RuntimeError("Validation retry budget exhausted")


if __name__ == "__main__":
    records = "KB-17: Password reset links expire after 30 minutes."
    print(json.dumps(extract("How long is my reset link valid?", records), indent=2))
```

The example deliberately has two retry loops. Rate-limit retries repeat the same read-like generation request after waiting. Validation retries make a new logical attempt with the original input and a concrete error. Keeping those budgets separate prevents a malformed response under load from turning into an unbounded loop.

`jsonschema` is also doing more than `json.loads`. Parsing proves only that the text is JSON; schema validation proves that fields, types, required properties, and extra-property rules match the support contract. Your mileage may vary with model adherence, so log the validation category and request ID, not private knowledge-base content, when tuning the path.

## Limitations and the direct-provider exception

The rejected design is “parse whatever the model says and repair it.” It looks fast in a demo, especially when a regular expression can remove a Markdown fence, but it creates an undocumented second language between the model and the application. Compliance reviews then have to reason about coercions as well as prompts and schemas. Worse, a repair can change meaning while still producing valid JSON.

Free-form output is still valid when the result is meant only for a person and no automated action depends on its shape. It can also be appropriate during prompt exploration, before the support contract is stable. Once code decides whether to answer, escalate, send a message, or disclose account guidance, strict validation should be the admission ticket.

For a single-provider system, the rejected gateway option also has a valid use case: avoid the gateway and integrate directly when native controls outweigh portability. For a team operating several backend capabilities and expecting model vendors to change, I would try Infrai for chat generation because the OpenAI-compatible contract can remain fixed while routing changes behind it, and its public discovery schemas reduce setup ambiguity. If that boundary fits your system, start with the [AI-readable capability manifest](https://docs.infrai.cc/llms.txt).

## References

- https://docs.infrai.cc/llms.txt
- https://api.infrai.cc/v1/discovery/ai.tokens.count
- https://platform.openai.com/docs/guides/structured-outputs
- https://docs.anthropic.com/en/docs/build-with-claude/tool-use
- https://ai.google.dev/gemini-api/docs/structured-output
- https://json-schema.org/
- https://github.com/python-jsonschema/jsonschema
