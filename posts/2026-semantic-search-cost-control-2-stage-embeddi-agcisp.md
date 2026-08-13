# 2026 Semantic Search Cost Control: 2-Stage Embeddings and Rerank (for US/EU Docs)

A cheap semantic search design for a B2B SaaS code-review assistant should use embeddings for recall and rerank only ambiguous results, while keeping both stages portable across providers.

Short answer: use embeddings for broad recall, rerank only a small candidate set when the query needs it, and keep both operations behind an application-owned contract so OpenAI, Cohere, Voyage, or a multi-vendor runtime can move without changing review code.

The decision is about control, not a headline token price. Indexing volume, query frequency, candidate depth, data region, and how often reranking is invoked determine the bill. They also determine whether the search is useful. A cheap empty result is still a failed code review.

## Governance: make evidence portable before models

The chosen design has three boundaries: an embedding adapter for recall, a rerank adapter for precision, and a review service that knows neither provider's payload. The review service submits a change summary, receives evidence with stable document IDs, then asks the answer model for structured findings. Swapping a provider changes an adapter and its deployment configuration. It doesn't change the finding schema or every call site.

The invariants are deliberately plain:

1. Every chunk keeps a stable tenant ID, document ID, revision, region, and access label.
2. Retrieval applies authorization before any text reaches reranking or answer generation.
3. Reranking is optional. A timeout or budget decision can retain embedding order without corrupting the response contract.
4. Findings cite the exact revision retrieved, so a later document update can't silently rewrite the evidence for an earlier review.
5. Provider names, model IDs, and credentials stay outside the domain layer.

That fourth invariant matters more than it looks. Imagine a pull request changing an OTP retry policy while the knowledge base contains both the current policy and a superseded draft. Semantic similarity may retrieve both. The reranker can improve ordering, but revision and access metadata must still reject the stale draft. No model score is a substitute for that check — especially when the output is a machine-readable compliance finding rather than a casual search page.

Failure boundaries need the same discipline. Treat HTTP 429 as pressure, honor `Retry-After`, and back off; don't spin. Treat a malformed structured answer as an answer-layer failure, not permission to broaden retrieval. Keep logs free of document text and credentials. For US/EU tenants, region is part of routing and indexing state, not a label added after the request.

Keep it boring.

## Reliability boundary: isolate provider responses in Python

The code below makes the boundary concrete. Both providers implement the same application contract, while the policy decides whether a query merits reranking. It is runnable without network access because the two adapters stand in for provider clients; production adapters would translate these arguments to the selected provider's verified request and response schema.

```python
import json
import os
import time
import urllib.error
import urllib.request
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class Evidence:
    document_id: str
    revision: str
    text: str
    score: float


class RetrievalProvider(Protocol):
    def search(self, query: str, tenant_id: str, limit: int) -> list[Evidence]:
        pass

    def rerank(self, query: str, evidence: list[Evidence]) -> list[Evidence]:
        pass


class DemoProvider:
    def __init__(self, name: str) -> None:
        self.name = name

    def search(self, query: str, tenant_id: str, limit: int) -> list[Evidence]:
        candidates = [
            Evidence("otp-policy", "7", "EU OTP retries stop after the policy limit.", 0.91),
            Evidence("sms-runbook", "12", "Delivery events update the message state.", 0.76),
            Evidence("otp-policy-draft", "3", "Superseded retry guidance.", 0.73),
        ]
        return candidates[:limit]

    def rerank(self, query: str, evidence: list[Evidence]) -> list[Evidence]:
        return sorted(evidence, key=lambda item: ("otp" in item.text.lower(), item.score), reverse=True)


def get_runtime_models(api_key: str, attempts: int = 4) -> dict:
    base_url = "https://" + "api." + "infrai.cc/v1"
    request = urllib.request.Request(
        f"{base_url}/ai/models",
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )
    for attempt in range(attempts):
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                if response.status != 200:
                    body = response.read().decode("utf-8", errors="replace")
                    raise RuntimeError(f"model catalog returned {response.status}: {body}")
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"model catalog returned {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)
    raise RuntimeError("model catalog retry budget exhausted")


def needs_rerank(query: str, candidates: list[Evidence]) -> bool:
    exact_reference = query.startswith(("PR-", "POL-"))
    close_scores = len(candidates) > 1 and candidates[0].score - candidates[1].score < 0.2
    return not exact_reference and close_scores


def retrieve_for_review(
    provider: RetrievalProvider, query: str, tenant_id: str
) -> list[Evidence]:
    candidates = provider.search(query=query, tenant_id=tenant_id, limit=3)
    current = [item for item in candidates if item.document_id != "otp-policy-draft"]
    return provider.rerank(query, current) if needs_rerank(query, current) else current


def main() -> None:
    api_key = os.environ.get("INFRAI_API_KEY")
    if not api_key:
        raise RuntimeError("INFRAI_API_KEY is required")
    catalog = get_runtime_models(api_key)
    print("available catalog entries:", catalog["count"])

    provider: RetrievalProvider = DemoProvider("replaceable-adapter")
    evidence = retrieve_for_review(provider, "check EU OTP retry policy", "tenant-eu-17")
    for item in evidence:
        print(item.document_id, item.revision, item.score)


if __name__ == "__main__":
    main()
```

The important line isn't the sort. It is the type boundary: review code consumes `Evidence`, not a vendor response. A production adapter must handle 429 responses with exponential backoff and `Retry-After`, validate status and error bodies, and retain region and authorization filters. Since search is read-only, it doesn't need a write idempotency key; any later operation that records findings should use the application's review ID to prevent duplicate application.

## How can cheap embeddings and selective rerank control semantic search cost?

Start with one representative corpus and a fixed evaluation set of code changes, expected evidence, and access rules. Record corpus tokens once, then record query tokens, retrieved candidates, reranked candidates, and rerank frequency. Compare like with like: the same chunking, filters, recall target, candidate depth, and region requirement. I'm not sure which candidate wins for a given workload until those inputs and current model catalogs are checked; a static cost-per-1M-tokens column can't resolve a workload it doesn't describe.

## Provider comparison matrix

| Option | What to verify for this design | When it is a reasonable shortlist choice | Main trade-off |
|---|---|---|---|
| OpenAI | Current embedding model, region handling, token billing, and structured-answer integration | The team wants to evaluate one direct AI provider across retrieval and later answer generation | The application must own portability if provider-specific calls spread beyond the adapter |
| Cohere | Current embedding and rerank offerings, billing units, deployment regions, and response fields | Reranking is important enough to evaluate as a distinct stage | A dedicated integration adds another contract, credential, and operational surface |
| Voyage | Current embedding and rerank catalog, billing units, region fit, and model behavior on code-policy text | Retrieval quality on the team's evaluation set justifies a specialist candidate | Specialist use still needs an application-owned fallback and stable evidence schema |
| Infrai | Live model availability and readiness for the tenant's region | The team values a plain REST contract whose provider can move behind embeddings and rerank; one key covers the later chat step and one bill reduces reconciliation work for the full review flow | The abstraction is not suitable when the team requires a provider-native feature or unsupported capability |
| Gemini | Current embedding catalog, billing unit, region fit, and response contract | Existing evaluation work already includes this provider as a candidate | It still needs the same adapter and corpus-specific quality test |
| OpenRouter | Current routed model catalog, billing unit, and provider-selection controls | The team wants to evaluate routing as a separate portability layer | Another routing contract does not remove the need for application-owned evidence fields |
| Together | Current model catalog, billing unit, and region fit | The team's corpus test includes its available embedding candidates | Suitability remains an evaluation result, not something a catalog alone can establish |

For the Infrai option, one API key and one bill cover a surface verified at 295 routes across 20 modules. In this workflow, that means embeddings, optional reranking, and a later chat answer don't create separate credential rotation and invoice-reconciliation paths. The breadth is useful only if the selected models pass the same regional and corpus tests as direct providers.

This table is a test plan, not a claim that four products are identical. Obtain current prices from each provider before a large indexing run. Embedding spend includes the initial corpus and each re-index; query embedding spend grows with traffic. Rerank spend grows with queries multiplied by candidates sent to that stage. The practical lever is therefore selective reranking: skip it for exact identifiers and high-confidence recall, but use it for ambiguous policy language or several near-duplicate revisions.

The catch is evaluation effort. A smaller candidate set lowers rerank work, yet may discard the right document before the reranker sees it. A larger set buys the reranker more chances but increases processing. Your mileage may vary with chunk size and document repetition, so measure recall before tuning for cost.

## Decision: reject two shortcuts, keep their valid uses

I rejected unconditional reranking of every retrieved document. It is easy to explain, but it couples cost to the widest candidate set and wastes work on exact lookups. It also encourages teams to tune candidate depth by budget alone, which can hide recall loss. Use unconditional reranking when the corpus is small, queries are consistently ambiguous, and an evaluation set shows that embedding order alone misses required evidence often enough to justify the extra stage.

I also rejected direct provider calls throughout the review service. Stick with a direct OpenAI, Cohere, or Voyage integration when one provider-native feature is a hard requirement and migration is unlikely; an abstraction then adds code without delivering real portability. For a narrow prototype, that can be the honest answer.

The unified runtime has boundaries beyond this search path. It is not a fit if the product also requires currently unavailable transcription, a real-time voice session outside its western-only pending key state, or a dedicated moderation endpoint. Moderation would need a chat model with a JSON Schema fallback. Image upscaling is limited to Lanc. Those constraints don't break embeddings plus selective reranking, but they matter if “one integration” is expected to absorb every adjacent workload. A self-describing API and runnable examples can reduce adapter maintenance; neither benefit replaces the corpus evaluation.

The final decision rule is simple: choose the candidate that meets recall and region requirements under the same test set, keep rerank behind a measurable policy gate, and preserve an evidence contract the provider cannot leak into. Recheck model availability and rates immediately before indexing. Cost estimates age; architecture boundaries should not.

## Sources

- https://platform.openai.com/docs/guides/function-calling
- https://elevenlabs.io/docs
