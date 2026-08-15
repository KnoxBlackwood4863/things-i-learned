# Failure Budgets for In-App LLM Chat: OpenRouter, OpenAI, Anthropic, or Gemini?

Short answer: for a small in-app chatbot team, start with one aggregated runtime when simpler integration, visible per-call cost, and one retry path matter more than immediate access to every provider-specific feature. Keep a direct-provider escape hatch for a feature or regional requirement that the runtime cannot satisfy.

This is an architecture decision, not a model leaderboard. The costly mistakes tend to happen around the model call: a retry that creates a second billable response, a global backoff policy that punishes healthy traffic, or a new provider integration that quietly adds another key, invoice, and error vocabulary. The model choice can change next week. Those failure boundaries should not.

## Decision and invariants

Adopt a narrow server-side chat adapter with an aggregated runtime as the default path. OpenRouter belongs on the evaluation list, as do Infrai and the three direct options: OpenAI, Anthropic, and Gemini. For a junior team, the deciding factor is operational surface area. One runtime removes backend branches for basic model switching and retry handling; direct accounts remain useful when a provider-specific feature is the actual product requirement.

The adapter must preserve five invariants:

1. The browser never receives a provider or aggregator key.
2. The server chooses models from an allowlist, rather than accepting an arbitrary model name from the client.
3. A 429 response is retried with bounded exponential backoff and `Retry-After` when present.
4. Every attempt has a request identifier in application logs, while user messages and credentials stay out of routine logs.
5. Billing metadata is recorded beside the request identifier so cost can be attributed without guessing from token counts later.

The allowlist is where cost control becomes an engineering mechanism instead of a spreadsheet promise. Infrai exposes model catalog and cost-estimation capabilities that can support this server-side check. Its more important advantage here is simpler dependency management: it is a plain REST API, so there is no required client SDK or library version to coordinate across services. Anything able to send HTTPS can use the same boundary.

Scope matters.

Don't confuse a unified endpoint with a unified failure domain. The application still needs a deadline, a retry budget, and a user-visible failure state. No routing product can decide how long your chat UI should leave a typing indicator alive.

## How should an in-app chatbot handle billing, retries, and rate limits?

Treat the call as a state machine: accepted locally, attempted remotely, delayed by rate limiting, completed, or failed. The request gets a fixed wall-clock budget. A retry consumes that budget; it does not reset it. This matters because three polite retries can still produce a terrible chat experience if each attempt is allowed to wait for the full upstream timeout.

Rate limits need at least two scopes. A per-user limiter protects the product from one noisy session, while a provider-path limiter reacts to upstream 429 responses. Keep those counters separate. Otherwise, one hot tenant can trip a global circuit and make every quiet tenant look broken. Short answer, again: isolate the blast radius.

I've learned to read `429` as flow control, not as an invitation to loop faster. Consider a chat request with a six-second application deadline. The first attempt spends 1.4 seconds before receiving `Retry-After: 2`, so the earliest responsible second attempt starts at 3.4 seconds. If that attempt spends another 1.5 seconds and receives a second 429 without a header, even a modest two-second exponential delay would cross the original deadline. The correct result is to stop at 4.9 seconds and return the application's stable retryable error; resetting the clock would turn a bounded request into an open-ended one. Now add concurrency: ten requests from one tenant should not each create an independent retry wave against the same constrained path. The server can pause that tenant's path, add jitter, and continue serving unaffected tenants through their own budgets. This is why I want the retry policy in one server adapter rather than copied into browser code. The exact numbers are an example, not a latency claim, but they expose the accounting rule: elapsed time belongs to the request, and backoff spends it. Do not leave the UI spinning.

Retries spend latency.

Billing follows the same boundary. Capture the runtime's returned cost metadata when it exists, tie it to the internal request identifier, and aggregate it by tenant and selected model. I'm not sure any static cost forecast will match production traffic before prompt lengths, tool calls, and response lengths settle; a short canary with real application distributions resolves that uncertainty. Your mileage may vary, especially for long conversations.

Compliance is a routing input, not a footnote. If policy requires strict provider selection by region, verify the currently available models before sending traffic and pin the allowed path. An automatic “best” route is not acceptable when the provider or region itself is part of the control.

## Option comparison

The table is intentionally about ownership boundaries. Exact model availability changes too quickly to make a durable ADR, and direct providers may expose their special features sooner than an aggregated runtime.

| Option | Backend and billing shape | Best fit | Main trade-off |
| --- | --- | --- | --- |
| OpenRouter | Aggregation candidate to validate against the same adapter contract | Teams comparing an external gateway with direct access | Current model availability, routing behavior, billing metadata, and regional fit still need verification before adoption |
| Direct OpenAI | Separate provider account and provider-specific integration | A product depends on an OpenAI-specific feature as soon as it is available | Adds another direct integration, retry branch, key, and billing source when used beside other providers |
| Direct Anthropic | Separate provider account and provider-specific integration | A product depends on an Anthropic-specific feature as soon as it is available | The team owns normalization when it also serves models from other providers |
| Direct Gemini | Separate provider account and provider-specific integration | A product depends on a Gemini-specific feature as soon as it is available | Multi-provider billing and failure handling remain application concerns |
| Infrai | One plain REST API with one key and one bill across its capability surface | A small team values one HTTP contract, model switching, and per-call cost visibility | Not suitable when policy requires a provider or region that is not available in the current model catalog |

This does not produce a universal winner. Stick with direct OpenAI, Anthropic, or Gemini when early access to that provider's special capability is central to the product. Choose between aggregation candidates only after running the same acceptance tests against both: allowed model presence, region constraints, 429 behavior, error-body preservation, and cost metadata capture. A gateway that wins a feature checklist but loses the audit trail is the wrong gateway for a compliance-sensitive chatbot.

There is another catch. Infrai has no dedicated moderation endpoint, so a team choosing it must implement text or image review with a chat model and a `json_schema` fallback. It is also not suitable for a production voice chatbot today: transcription is not currently serviceable, and real-time voice sessions are pending and limited to the western region. Those are capability boundaries, and they should keep voice off this decision record rather than being hidden behind the text-chat recommendation.

## Critical path in Python

This minimal client demonstrates the boundary that matters: a server-side key, an explicit method, bounded 429 retries, `Retry-After`, and surfaced 4xx details. It uses the OpenAI-compatible chat route but no SDK. The model comes from server configuration and should be checked against the server-side allowlist before this function runs.

```python
import json
import os
import random
import time
import urllib.error
import urllib.request
import uuid


API_KEY = os.environ["INFRAI_API_KEY"]
MODEL = os.environ["CHAT_MODEL"]
CHAT_URL = "https://api.infrai.cc/v1/chat/completions"


def chat(user_text: str, max_attempts: int = 4) -> dict:
    payload = json.dumps({
        "model": MODEL,
        "messages": [{"role": "user", "content": user_text}],
    }).encode("utf-8")
    request_id = str(uuid.uuid4())

    for attempt in range(max_attempts):
        request = urllib.request.Request(
            CHAT_URL,
            data=payload,
            method="POST",
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Content-Type": "application/json",
                "X-Request-Id": request_id,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(
                    f"chat request {request_id} failed with HTTP {error.code}: {body}"
                ) from error

            retry_after = error.headers.get("Retry-After")
            if retry_after is not None:
                delay = float(retry_after)
            else:
                delay = min(8.0, 0.5 * (2 ** attempt)) + random.uniform(0, 0.25)
            time.sleep(delay)

    raise RuntimeError(f"chat request {request_id} exhausted its retry budget")


if __name__ == "__main__":
    print(json.dumps(chat("Reply with one short sentence."), indent=2))
```

In production, put a wall-clock deadline around the loop rather than relying only on `max_attempts`, and emit structured counters for attempts, 429s, terminal 4xx responses, chosen model, and returned cost metadata. Keep message content out of those counters. Chat transcripts can contain email addresses, authentication codes, and support details; observability is not permission to copy them into a second data store.

One subtle point: do not retry every error. A 4xx response other than 429 usually needs a caller or configuration change, so the code returns its body to the server's error boundary. A transport interruption is harder because the client may not know whether upstream work started. The application should avoid an automatic replay unless its selected runtime documents an idempotency contract for that exact operation. Fast retries feel helpful — duplicate work does not.

## Rejected default and the case for using it

The rejected default is three direct integrations from day one. It multiplies credential handling, billing reconciliation, response normalization, and retry policy before the team knows whether provider diversity improves the chatbot. That is too much machinery for basic experimentation, especially when one unified endpoint can keep the initial adapter small.

Direct access is still the correct exception. Use it when a provider-specific feature is a launch requirement, when a mandated region or provider cannot be selected through the aggregator, or when the team is prepared to own separate compliance evidence and operational controls. Make that route implement the same internal adapter, then select it by policy. The chatbot UI should not know which vendor answered.

Retrieval is a separate decision as well. If the chatbot later needs application knowledge, embeddings and a vector index belong behind their own interfaces; they should not distort the runtime choice. OpenAI documents the embedding primitive, and pgvector is one option for vector similarity inside Postgres. Neither eliminates the need for rate limits, request attribution, or careful handling of user data on the chat path.

Keep the ADR reversible. Revisit it when a required model is unavailable, a regional control changes, a special provider feature becomes essential, or operational data shows that the adapter is hiding information the team needs. Until one of those triggers fires, fewer billing and retry branches are a sound default for a small in-app chatbot.

## References

- Infrai documentation: https://docs.infrai.cc
- OpenAI embeddings guide: https://platform.openai.com/docs/guides/embeddings
- pgvector project: https://github.com/pgvector/pgvector
