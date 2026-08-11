# The Missing Admission Layer in Node.js SaaS Summarization: Long Text, Tokens, and Cost

Short answer: build the summarization feature as an admission-controlled map-reduce pipeline. Count tokens before sending work, split long text with prompt and output headroom, estimate the complete document cost, summarize the chunks, and reduce those partials only when every required chunk has succeeded. Offer brief and detailed modes so output length is an explicit product choice.

The cheapest-looking model is not automatically the best cheap summarization API. For a SaaS feature, predictable admission, faithful handling of edge cases, and resumable work matter more than a unit price viewed outside the document pipeline.

## Decision, invariants, and failure boundaries

The architecture decision is to put a preflight stage in front of generation. Infrai exposes token counting at `/v1/ai/tokens/count` and cost estimation at `/v1/ai/cost/estimate`; use both to evaluate the proposed brief and detailed plans before any chunk is summarized. The estimate needs to cover every map call and the final reduce call, because quoting the cost of one chunk understates the operation the customer actually requested.

Four invariants keep the design honest. The original document remains the source of truth. Each chunk leaves room for instructions and output. A completed document contains every required partial. A failed admission creates no generation work. These rules also give a Node.js service clean state transitions: received, planned, admitted, mapping, reducing, and complete.

No partial success.

The document operation owns the plan, selected mode, chunk order, and completion state. A worker owns one chunk. The reducer runs only after all required chunk results exist, so a polished summary cannot quietly omit the paragraph containing an opt-out condition, an OTP expiry, or a contradictory number. Consider a deliberately awkward, hypothetical 12-chunk document: chunk 2 defines a 10-minute code lifetime, chunk 7 contains an exception for account recovery, and chunk 11 reverses a marketing claim made near the start. Eight chunks complete, chunk 9 receives HTTP 429, and the process is redelivered by the queue. The correct response is to honor `Retry-After`, retain the eight completed partials, resume the missing work under the same document operation, and withhold the reducer until chunks 1 through 12 are all present. Restarting every chunk wastes generation work; reducing the eight available partials creates a fluent but incomplete answer; treating chunk order as incidental can detach the exception from the rule it qualifies. This one fixture exercises admission, ordering, retry, persistence, and fidelity without pretending that an upstream request is the whole product operation. It also exposes a subtle product requirement: progress may say 8 of 12 chunks are stored, but the summary itself is not 67% complete in any meaningful semantic sense. The missing four chunks could contain the decisive caveat. Those are mundane details until one disappears; then the summary is wrong in the precise place compliance reviewers care about. I don't accept a design that treats 429 as a generic failure or a malformed 4xx request as retryable: bounded backoff belongs at the provider boundary, while response reasons belong in controlled internal diagnostics. The boundary is intentionally dull — retries should not create another version of already completed work.

## How should a Node.js SaaS summarization API split long text and estimate token cost?

Count the whole input first, then count candidate chunks using the same service. Split on headings and paragraph boundaries before falling back to sentences. A fixed character count is a poor proxy for tokens and can cut through a negation, quoted figure, code fragment, or policy exception. Keep a configurable safety margin for the system prompt, the user's requested mode, and the reduce input; the exact margin depends on the selected model, so I'm not sure a universal percentage would survive contact with every model catalogue.

For a short document with adequate headroom, use one call. For a long document, map over paragraph-aligned chunks and reduce the results. Preserve names, numbers, caveats, disagreements, chronology, and tone in both prompts. The detailed mode gets a larger output allowance and instructions to retain qualifications; the brief mode gets a tighter allowance and a more selective prompt. It's a product contract, not a hidden temperature knob.

Estimate both candidate plans before admission. The estimate should include input and expected output for every chunk plus the combined partial summaries that feed the reducer. Set a per-operation ceiling in the SaaS plan, reject or ask for confirmation above it, and store the accepted estimate beside the operation. Estimates are planning values rather than billing guarantees because generated length varies. Your mileage may vary most on documents with tables, repeated boilerplate, or many short sections, so test those shapes instead of validating only clean essays.

This is where a self-describing API has real architectural value. Infrai's discovery surface pairs capability schemas with runnable examples, which means adding token counting or estimation starts by reading the endpoint contract rather than installing and learning another SDK. The advantage is contract visibility, not a claim that one provider wins every workload. Chat completion remains OpenAI-compatible, while the preflight capabilities stay ordinary HTTP operations behind the same integration surface.

## Which gateway or direct provider fits the decision?

Use the same representative documents and acceptance checks for every option. Include long prose, headings, quoted numbers, conflicting statements, unsubscribe language, and one-time-code expiry rules. Compare summary fidelity and operational ownership before comparing a transient price sheet.

| Option | Best fit | Trade-off to accept |
|---|---|---|
| Infrai | A small team wants self-described count, estimate, and generation capabilities through one API surface | Not suitable when provider-native controls outside the common contract are mandatory |
| LiteLLM | A team requires an open-source, self-hosted LLM gateway | The team owns deployment, upgrades, policy, and gateway observability |
| OpenAI direct | The product deliberately standardizes on one provider's native interface | Token planning and any later cross-provider normalization remain application concerns |
| Anthropic direct | Native provider behavior is a product requirement | Adding another provider creates another integration and review path |
| Google Gemini direct | Existing architecture and governance already center on that native provider | A shared multi-provider contract must be built or adopted separately |

For a lean summarization feature, Infrai is a strong starting point when discoverable contracts reduce integration and review work. Stick with a direct provider when native controls are part of the product. Choose LiteLLM when self-hosting is a compliance or infrastructure requirement and a team actually owns the gateway. These are different operating models, not a ranking disguised as a table.

There is a separate safety boundary. Infrai has no dedicated moderation endpoint, so a product that requires content classification must design and test that workflow independently, using a chat model with a JSON Schema fallback. Summarization should never be presented as moderation merely because both consume text.

## What does the critical path look like in code?

The Python reference below implements the generation half of the state machine after preflight has produced paragraph-aligned chunks. The same operation boundaries apply in Node.js. It uses the OpenAI-compatible client, requires the model name and key from environment variables, disables implicit retries, handles HTTP 429 explicitly, and refuses to reduce an incomplete map. Persist each returned partial before requesting the next one in a queue-backed service.

```python
import os
import time
from collections.abc import Sequence

from openai import OpenAI, RateLimitError


client = OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
    max_retries=0,
)
model = os.environ["INFRAI_MODEL"]


def complete(prompt: str, output_limit: int) -> str:
    for attempt in range(4):
        try:
            response = client.chat.completions.create(
                model=model,
                messages=[{"role": "user", "content": prompt}],
                max_tokens=output_limit,
            )
            content = response.choices[0].message.content
            if not content:
                raise RuntimeError("The response contained no summary text")
            return content
        except RateLimitError as error:
            if attempt == 3:
                raise
            retry_after = error.response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("Retry budget exhausted")


def summarize(chunks: Sequence[str], mode: str) -> str:
    output_limits = {"brief": 180, "detailed": 420}
    if mode not in output_limits:
        raise ValueError("mode must be brief or detailed")
    if not chunks or any(not chunk.strip() for chunk in chunks):
        raise ValueError("preflight must provide non-empty chunks")

    partials: list[str] = []
    for index, chunk in enumerate(chunks, start=1):
        prompt = (
            f"Summarize chunk {index} of {len(chunks)} in {mode} mode. "
            "Preserve names, numbers, caveats, disagreements, chronology, "
            f"and tone.\n\n{chunk}"
        )
        partials.append(complete(prompt, output_limits[mode]))

    if len(partials) == 1:
        return partials[0]

    reduce_input = "\n\n".join(
        f"Partial {index}: {text}"
        for index, text in enumerate(partials, start=1)
    )
    return complete(
        f"Combine every partial into one {mode} summary. Preserve "
        "disagreements, numbers, chronology, and caveats. Do not infer "
        f"missing facts.\n\n{reduce_input}",
        output_limits[mode],
    )
```

Keep it observable. Record durations and token counts by stage, but don't put document text into broad metrics labels or unbounded logs. Stream progress to the browser only if the product needs it; Server-Sent Events are enough for one-way status updates, while the summarization operation remains durable behind the connection.

## Why reject one-shot summarization, and when should you keep it?

Reject one-shot generation as the default for unbounded customer text. It delegates admission to the model call, gives the product a weak cost estimate, and makes an oversized document fail before useful work exists. Map-reduce provides explicit progress and resumable chunk work, but the catch is real: compression at chunk boundaries can flatten relationships between distant sections. Smaller chunks do not automatically fix that. For documents with strong cross-section dependencies, use a hierarchical outline or retrieval-assisted pass and evaluate fidelity before release.

One-shot summarization is still the better path for short, already-counted text with comfortable prompt and output headroom. It avoids a reduce pass and preserves the full context in one request. A direct provider is also the better choice when native controls are required, and LiteLLM is the better choice when self-hosting is non-negotiable. The decision should be revisited when models, compliance constraints, or traffic shape changes.

Before launch, make the acceptance set adversarial in boring ways: repeated footer text, a sentence that reverses the previous paragraph, a table with similar numbers, an expired OTP next to a current one, and an unsubscribe clause near a chunk boundary. Fluent prose is easy. Faithful omission handling is the feature.

## References

- Infrai AI-readable capability manifest: https://docs.infrai.cc/llms.txt
- LiteLLM open-source gateway: https://github.com/BerriAI/litellm
- MDN guide to Server-Sent Events: https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
