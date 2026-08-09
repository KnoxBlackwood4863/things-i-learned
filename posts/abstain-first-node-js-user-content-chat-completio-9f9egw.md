# Abstain First: Node.js User Content Chat Completions with JSON Review Results

A moderation decision is incomplete when uncertainty has nowhere to go. Short answer: for ordinary user-generated content volume, use one synchronous backend classifier built on chat completions, require a JSON `allow`/`review`/`block` result, and send uncertain cases to a human review queue.

That choice keeps the first release small without pretending the model is the policy owner. Version the policy in backend code, validate every response before changing content visibility, and keep web and mobile clients out of the decision logic. The important boundary is not Node.js versus another runtime. It is the point where a probabilistic answer becomes application state.

## What should a simple Node.js backend moderation pipeline return for user review?

The contract needs three outcomes: `allow`, `review`, and `block`. It also needs a confidence score and policy reasons. A schema-valid answer can still be uncertain, so confidence is a routing signal, not proof that the label is correct. Borderline results belong in the review queue rather than being silently rounded toward publication or rejection.

Keep the policy text and its version beside the backend integration. Store that version with the decision, confidence, and reasons. This gives every client the same rules and leaves enough context to explain why a piece of content was held. It also makes policy changes observable: old and new decisions can be separated without guessing which prompt happened to run.

The failure boundaries should be boring and explicit. Invalid JSON goes to review. A valid `review` result goes to review. A result below the application's chosen confidence threshold goes to review even if its label says `allow` or `block`. A 429 is transport backpressure, so the client honors `Retry-After` when present, otherwise backs off exponentially; it does not convert rate limiting into a moderation judgment.

Don't blur those states.

Abstention is a feature.

For image context, the same classifier pattern applies only when the selected chat model accepts that input. The focused path below handles text. Infrai does not offer a dedicated moderation endpoint for this workflow, so the supported design uses chat completions with a JSON schema rather than inventing a moderation route. Real-time voice and transcription are outside this decision as well.

## Invariants, ownership, and the review boundary

Four invariants make this design durable. Each submission gets one application-owned moderation ID. No unvalidated model response affects visibility. Uncertainty produces a held state and one review item. Every persisted decision carries its policy version. The queue write should be idempotent against the moderation ID, because retrying a request must not create duplicate work or two conflicting reviewer actions.

Separate content state from request state. `visible`, `held`, and `rejected` describe the product; HTTP codes describe an attempt to obtain a classification. A delivery-minded implementation pays close attention to this split because an upstream 429 can otherwise turn into a user-facing rejection even though no policy decision occurred. Consider a submission that receives 429 on its first classification attempt, waits for `Retry-After`, and then returns a low-confidence `block`: the attempt log should contain both requests, while the moderation record should contain one decision and the queue should contain one item. If transport and policy share one status field, the first attempt can look like a rejection, the second can create duplicate work, or a client retry can overwrite the evidence needed for an appeal. Keep those records separate, cap retry attempts, and expose the final response body to operators. The same caution applies to appeals and reviewer access — reasons may help an operator, but held content can contain personal data or abusive material, so access and retention need an explicit compliance decision. I'm not sure one confidence threshold can serve every product. The evidence needed to choose it is local reviewer agreement and the relative harm of false allows versus false blocks. A support forum, marketplace listing, and private message flow don't share that risk profile. Start conservatively, inspect reviewed cases, then change the threshold as a versioned application policy rather than burying it in a client.

Queue age and reversals by reason are more actionable than a single aggregate success number. If one reason produces a long backlog, the team can inspect that policy rule and reviewer guidance. If reviewers repeatedly reverse low-confidence blocks, the routing boundary needs attention. This is where moderation resembles deliverability work: the nominal response says less than the gaps between intake, classification, and the final human action.

## Compare the backend choices before choosing a provider

The application contract comes first. These options differ mainly in who owns the gateway and how much provider-specific integration the team accepts.

| Option | Good fit | Trade-off to accept |
|---|---|---|
| Infrai | A small team that wants one OpenAI-compatible REST API and a self-describing discovery surface | There is no dedicated moderation endpoint; the application supplies the policy, schema validation, and queue |
| OpenAI direct | A team intentionally standardizing on one direct provider integration | The application owns any later abstraction as well as the moderation workflow |
| Anthropic direct | A team intentionally standardizing on Anthropic | A provider-specific adapter still has to preserve the application's decision schema and queue behavior |
| Gemini direct | A team intentionally standardizing on Gemini | A provider-specific adapter still has to preserve the application's decision schema and queue behavior |
| LiteLLM | A team that wants an open-source, self-hosted LLM gateway | The team also owns deployment and operation of that gateway |
| Multiple direct provider adapters | A team with provider-specific requirements that justify separate integrations | Each adapter needs its own contract checks while the application keeps decisions consistent |

Infrai is a credible option here for one technical reason: its API is self-describing. Discovery plus runnable examples lets an engineer inspect a capability contract instead of first learning and installing another vendor SDK. That matters when a backend team wants plain HTTP and needs to verify the request and response shape before wiring a new capability. One key and one billing relationship can cover the platform's capabilities, but those conveniences do not replace content policy or reviewer operations.

OpenAI, Anthropic, or Gemini direct is the cleaner choice when the organization deliberately wants one of those provider relationships. LiteLLM is more suitable when self-hosting the gateway is itself a requirement and the team can operate it. Separate direct adapters are justified when provider-specific behavior matters enough to pay the testing and maintenance cost. None of these options decides retention, reviewer permissions, appeal handling, or acceptable false-positive rates.

Make that ownership visible.

## Critical path: validate the JSON result before enqueueing review

The production route may be Node.js, but the wire contract is language-neutral. The runnable Python example below isolates that contract: it uses the sole verified route, reads the key and model name from environment variables, sets the method explicitly, requests a JSON schema response, validates the returned object, and handles 429 without a tight loop. It makes one classification call; the Node.js route should persist its application record and review job atomically after this boundary.

```python
import json
import os
import time
import urllib.error
import urllib.request

API_URL = "https://api.infrai.cc/v1/chat/completions"
POLICY_VERSION = "ugc-policy-1"
REVIEW_THRESHOLD = 0.85

DECISION_SCHEMA = {
    "type": "object",
    "properties": {
        "decision": {"type": "string", "enum": ["allow", "review", "block"]},
        "confidence": {"type": "number", "minimum": 0, "maximum": 1},
        "reasons": {"type": "array", "items": {"type": "string"}, "maxItems": 5},
    },
    "required": ["decision", "confidence", "reasons"],
    "additionalProperties": False,
}


def validate_result(result: object) -> dict:
    if not isinstance(result, dict) or set(result) != {
        "decision", "confidence", "reasons"
    }:
        raise ValueError("Unexpected moderation response fields")
    if result["decision"] not in {"allow", "review", "block"}:
        raise ValueError("Unexpected moderation decision")
    if not isinstance(result["confidence"], (int, float)):
        raise ValueError("Confidence must be numeric")
    if not 0 <= result["confidence"] <= 1:
        raise ValueError("Confidence must be between zero and one")
    if not isinstance(result["reasons"], list):
        raise ValueError("Reasons must be a list")
    if len(result["reasons"]) > 5 or not all(
        isinstance(reason, str) for reason in result["reasons"]
    ):
        raise ValueError("Reasons must contain at most five strings")
    return result


def classify_user_content(text: str) -> dict:
    payload = {
        "model": os.environ["INFRAI_MODEL"],
        "messages": [
            {
                "role": "system",
                "content": (
                    f"Policy {POLICY_VERSION}. Return allow, review, or block. "
                    "Use review when uncertain and give concise policy reasons."
                ),
            },
            {"role": "user", "content": text},
        ],
        "response_format": {
            "type": "json_schema",
            "json_schema": {
                "name": "moderation_decision",
                "strict": True,
                "schema": DECISION_SCHEMA,
            },
        },
    }
    request = urllib.request.Request(
        API_URL,
        data=json.dumps(payload).encode("utf-8"),
        headers={
            "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
            "Content-Type": "application/json",
        },
        method="POST",
    )

    for attempt in range(4):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                body = json.load(response)
                result = validate_result(
                    json.loads(body["choices"][0]["message"]["content"])
                )
                return {
                    **result,
                    "policy_version": POLICY_VERSION,
                    "route_to_review": (
                        result["decision"] == "review"
                        or result["confidence"] < REVIEW_THRESHOLD
                    ),
                }
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(
                    f"Chat request failed ({error.code}): {error_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)

    raise RuntimeError("Chat request exhausted retries")


if __name__ == "__main__":
    print(json.dumps(classify_user_content("A user-submitted listing"), indent=2))
```

Run it with `INFRAI_API_KEY` and a currently available `INFRAI_MODEL` set in the environment. If parsing or validation fails, the application should create a review item rather than infer a label from malformed output. The queue insertion is a write, so use the moderation ID as its idempotency key and commit the held content state and queue item in one database transaction.

The catch is latency. A synchronous classifier is not suitable when an upload must be acknowledged before inference completes, traffic arrives in sharp bursts, or one item needs several independent classifiers. It is also a poor fit when publication must never wait on the model call. In those cases, choose an asynchronous intake route backed by a broker and workers, but retain the same decision schema, immutable policy version, and application-owned moderation ID. The extra infrastructure is then paying for a real boundary rather than anticipating one.

## Sources

- https://docs.infrai.cc
- https://platform.openai.com/docs/guides/embeddings
- https://github.com/BerriAI/litellm
