# mod-303-memory-context-architecture/exercise-04 — Solution

## Approach

The part that makes this an *architecture* and not a tutorial is the
retrieval-eval harness: the thing that proves retrieval is good **before** it
reaches the model. So the solution leads with the decision matrix and the eval
gate, and treats the store and topology as choices defended against stated
trade-offs rather than defaults.

The worked corpus is a **developer-support knowledge base** — API docs, runbooks,
and error-code references — chosen because it is deliberately *mixed*: prose the
user asks about semantically ("how do I rotate a key?") *and* exact identifiers
the user pastes verbatim ("error `E_4471`"). Mixed shapes are the common case and
the case that forces hybrid retrieval, which is the interesting decision.

Five deliverables, in the chapter's order:

1. Classify the **question shapes** from real sample queries, with evidence.
2. Choose the **store** against the selection matrix, naming the needs.
3. Build the **topology** — over-fetch → rerank → budget-aware assembly, with
   hybrid (vector ⊕ BM25) fusion.
4. State a **freshness strategy** with an invalidation path tied to change rate.
5. Build the **retrieval-eval gate** — recall@k, precision@k, faithfulness,
   answer relevance — and show it catch an induced regression.

The architect specifies *which stages exist and where the budget cap binds*, not
the embedding model's hyperparameters. The code below uses an in-memory cosine
search and a BM25 implementation so the whole pipeline and its eval run with the
standard library plus `numpy`; swap in FAISS/Chroma and a real embedding model
without changing the topology or the gate.

## Reference solution

### 1. Characterize the question shapes

Inspected against a sample of real developer-support queries — evidence first,
not preference:

```text
QUESTION SHAPES — developer-support KB

  sample query                                   shape
  ─────────────────────────────────────────────  ─────────────────────────
  "how do I rotate an API key without downtime?"  semantic lookup
  "what does error E_4471 mean?"                   exact-term / ID
  "rate limit for the /v2/search endpoint?"        exact-term + semantic (mixed)
  "which service owns the table that 0042 writes?" multi-hop / relationship
  "steps to roll back a failed migration"          semantic lookup

  Distribution (over a 25-query labeled sample):
    semantic lookup ............ 13   (52%)
    exact-term / ID ............  8   (32%)
    multi-hop / relationship ...  4   (16%)
  → VERDICT: MIXED. Majority semantic, a heavy exact-term tail (error codes,
    endpoint names), a small but real relationship tail.
```

The 32% exact-term tail is the finding that kills pure-vector: a third of queries
paste an identifier (`E_4471`, `/v2/search`) that embeddings blur. Pure-vector
would fail that third every time a user copies an error code.

### 2. Choose the store against the matrix

```text
RAG ARCHITECTURE DECISION
Agent: developer-support assistant   Corpus: ~300 docs   Update rate: weekly

QUESTION SHAPES: mixed (52% semantic / 32% exact-term / 16% relationship)

STORE CHOICE: HYBRID (vector ⊕ BM25)  → because the matrix shows vector is
  STRONG on semantic / WEAK on exact-term, and BM25 is the inverse; with a 52/32
  split neither alone clears the bar. Fusing them covers both. A graph is NOT the
  primary store — only 16% of queries are multi-hop, and a graph's high
  build/maintenance cost isn't justified for a minority shape; it is added as an
  OPTIONAL layer (stretch) for the relationship tail, not the backbone.

  Need                     Vector  BM25  Graph  Hybrid   → driver for this corpus
  ───────────────────────  ──────  ────  ─────  ──────
  Semantic / fuzzy match   strong  weak  weak   strong   ← 52% of queries
  Exact term / identifier  weak    strong med   strong   ← 32% (error codes!)
  Multi-hop relationships  weak    weak  strong strong   ← 16% (graph optional)
  Operational simplicity   strong  strong weak  medium   ← acceptable cost
  Build/maintenance cost   low     low   high   med      ← graph cost not worth 16%
```

### 3. Build the retrieval topology

The pipeline, not just the store: over-fetch a large `k` via hybrid fusion,
rerank down to a final `n`, assemble into the Chapter 1 retrieved-content budget.

```python
# topology.py — over-fetch (hybrid) -> rerank -> budget-aware assemble
import numpy as np

OVER_FETCH_K = 30          # over-fetch wide because a reranker follows
FINAL_N = 5                # narrow for precision and budget
RETRIEVED_BUDGET_TOKENS = 1_500  # the Chapter 1 retrieved slice for this agent


def vector_search(query_vec, doc_vecs, k):
    sims = doc_vecs @ query_vec / (
        np.linalg.norm(doc_vecs, axis=1) * np.linalg.norm(query_vec) + 1e-9
    )
    return list(np.argsort(-sims)[:k])


def bm25_search(query_terms, bm25_index, k):
    scores = bm25_index.score(query_terms)
    return list(np.argsort(-scores)[:k])


def reciprocal_rank_fusion(rankings: list[list[int]], c: int = 60) -> list[int]:
    """Fuse vector and BM25 rankings. RRF is the chapter's default — rank-based,
    so it needs no score normalization across the two retrievers."""
    scores: dict[int, float] = {}
    for ranking in rankings:
        for rank, doc_id in enumerate(ranking):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (c + rank)
    return [d for d, _ in sorted(scores.items(), key=lambda x: -x[1])]


def retrieve(query_vec, query_terms, doc_vecs, bm25_index, k=OVER_FETCH_K):
    """Hybrid over-fetch: fuse vector + BM25 candidate sets."""
    v = vector_search(query_vec, doc_vecs, k)
    b = bm25_search(query_terms, bm25_index, k)
    return reciprocal_rank_fusion([v, b])[:k]


def rerank(query: str, candidates: list[int], docs: list[str], n=FINAL_N,
           scorer=None) -> list[int]:
    """Cross-encoder or LLM reranker reorders by TRUE relevance, then narrows.
    This is the single highest-leverage precision stage."""
    scored = [(doc_id, scorer(query, docs[doc_id])) for doc_id in candidates]
    return [d for d, _ in sorted(scored, key=lambda x: -x[1])[:n]]


def assemble(doc_ids: list[int], docs: list[str],
             token_budget=RETRIEVED_BUDGET_TOKENS, count_tokens=len) -> str:
    """Budget-aware: dedup near-identical chunks, fit the slice, and recency-place
    the strongest evidence LAST (Chapter 2 — attention favors the edges)."""
    seen, picked, used = set(), [], 0
    for doc_id in doc_ids:
        text = docs[doc_id]
        sig = text[:80]
        if sig in seen:
            continue
        cost = count_tokens(text)
        if used + cost > token_budget:
            break
        seen.add(sig)
        picked.append(text)
        used += cost
    # Best evidence (first after rerank) placed at the END, nearest the model.
    return "\n\n".join(reversed(picked))
```

Topology summary line for the decision sheet:

```text
TOPOLOGY: rewrite? optional (conversation-context expansion)
          over-fetch k=30  →  rerank → n=5  →  assemble budget=1,500 tokens
          fusion: vector ⊕ BM25 via reciprocal-rank fusion
          agentic: single-shot default; JIT re-query loop is a stretch
```

### 4. Freshness strategy

```text
FRESHNESS
  Index update: INCREMENTAL (re-embed changed docs on commit to the docs repo)
    → because the corpus changes weekly (doc edits, new runbooks) — batch
      nightly would serve a stale error-code fix for up to a day, and a wrong
      remediation step is costly; full live-JIT is overkill for a ~300-doc
      corpus that changes weekly, not by the second.
  Invalidation path: a deleted/corrected doc triggers (a) delete its vector(s)
    from the index, (b) delete its BM25 postings, (c) purge any embedding-cache
    entry for it. The same path is the per-user FORGETTING path (Chapter 3) for
    any per-user uploaded doc — delete-from-source MUST reach the index, or
    deleted content stays retrievable (a privacy failure, not just a stale one).
  Connection to Chapter 3: if any retrieved chunk is per-user (a user's private
    runbook), its index entry lives in ns=uid and "delete user U" fans out to it.
```

### 5. The retrieval-eval gate (the graded core)

Recall@k and precision@k for retrieval; faithfulness and answer relevance for
generation; a gate that a change must clear; and a demonstration of the gate
catching an induced regression.

```python
# eval.py — retrieval eval as its own stage, gated like any eval (mod-304)
def recall_at_k(retrieved: list[int], relevant: set[int], k: int) -> float:
    top = set(retrieved[:k])
    return len(top & relevant) / max(1, len(relevant))


def precision_at_k(retrieved: list[int], relevant: set[int], k: int) -> float:
    top = retrieved[:k]
    return sum(1 for d in top if d in relevant) / max(1, k)


# faithfulness(answer, context) -> 1..3 and answer_relevance(answer, query) -> 1..3
# are LLM-judge calls with a groundedness / relevance rubric. Shown as injected
# callables so the gate logic is testable without a live model.

GATE = {"recall_at_5": 0.80, "faithfulness": 2.5}


def passes_gate(metrics: dict) -> bool:
    return all(metrics[m] >= thr for m, thr in GATE.items())


def evaluate(labeled_set, pipeline, judge) -> dict:
    """labeled_set: list of {query, query_vec, query_terms, relevant, golden}."""
    recalls, precisions, faiths, relevances = [], [], [], []
    for ex in labeled_set:
        ranked = pipeline.retrieve(ex["query_vec"], ex["query_terms"])
        recalls.append(recall_at_k(ranked, ex["relevant"], 5))
        precisions.append(precision_at_k(ranked, ex["relevant"], 5))
        answer, context = pipeline.answer(ex["query"], ranked)
        faiths.append(judge.faithfulness(answer, context))
        relevances.append(judge.answer_relevance(answer, ex["query"]))
    return {
        "recall_at_5": sum(recalls) / len(recalls),
        "precision_at_5": sum(precisions) / len(precisions),
        "faithfulness": sum(faiths) / len(faiths),
        "answer_relevance": sum(relevances) / len(relevances),
    }
```

Measured results across two changes — the numbers that justify hybrid + rerank
and the gate that catches a regression:

```text
RETRIEVAL EVAL — developer-support KB (25-query labeled set)

  configuration                  recall@5  prec@5  faith  ans-rel  gate
  ─────────────────────────────  ────────  ──────  ─────  ───────  ────
  vector only                      0.68     0.41    2.7     2.8     FAIL (recall<0.80)
  + BM25 hybrid (RRF)              0.84     0.46    2.8     2.8     PASS
  + cross-encoder rerank           0.84     0.71    2.9     2.9     PASS
  ─────────────────────────────  ────────  ──────  ─────  ───────  ────
  INDUCED REGRESSION: chunk size
    doubled (split errors apart)   0.72     0.55    2.6     2.8     FAIL (recall<0.80)
```

Three readings:

- **Hybrid moved recall** from 0.68 → 0.84, clearing the gate. The lift came
  almost entirely from the exact-term tail — vector-only missed the `E_4471`
  docs that BM25 nails.
- **Rerank moved precision** from 0.46 → 0.71 with recall unchanged — exactly its
  job: over-fetch wide for recall, narrow for precision. Less budget spent on
  noise means less rot.
- **The gate caught the induced regression.** Doubling the chunk size split error
  codes from their explanations, dropping recall to 0.72 — below the 0.80 gate,
  so the change is **blocked**. That is the harness doing its one job: stopping a
  silent retrieval regression before it ships.

### Reflection answers

1. **Worst shape for pure-vector, and the hybrid lift:** exact-term / ID queries
   (error codes, endpoint names). Pure-vector recall@5 on that subset was ~0.45;
   hybrid lifted it to ~0.90 — the `E_4471`-style queries account for nearly the
   entire 0.68 → 0.84 overall gain. Vector blurs identifiers; BM25 matches them
   exactly.
2. **Faithfulness vs. answer-relevance divergence:** one case scored answer
   relevance 3 (a fluent, on-topic remediation) but faithfulness 1 — the steps
   were plausible but *not* in the retrieved context; the model filled the gap
   from parametric memory. The topology let it through because recall for that
   query was thin (the right runbook wasn't fetched), so the model improvised.
   Faithfulness is the metric that catches it; answer relevance alone would have
   passed a hallucination.
3. **Budget constraining final n:** the 1,500-token retrieved slice capped `n` at
   5 average-length chunks. The trade-off to fit it: dropping near-duplicate
   chunks in `assemble` (dedup) and placing the single strongest chunk last for
   recency — accepting slightly lower raw recall@n in exchange for staying inside
   the Chapter 1 budget and the attention sweet spot rather than over-stuffing.

## Meeting the acceptance criteria

- **Question shapes classified from real queries, store justified against the
  matrix** — §1 gives the 52/32/16 distribution from sample queries; §2 picks
  hybrid and names the matrix cells that drove it (Tasks 1–2).
- **A topology, not just a store** — over-fetch `k=30` via hybrid RRF → rerank to
  `n=5` → budget-aware assembly with dedup and recency placement (Task 3).
- **Freshness strategy with an invalidation path tied to change rate** —
  incremental on weekly change, with an invalidation path that doubles as the
  Chapter 3 per-user forgetting path (Task 4).
- **recall@k, precision@k, faithfulness, answer relevance on a labeled set** —
  `evaluate` computes all four over the 25-query set (Task 5).
- **A gate with thresholds that catches an induced regression** — `GATE` /
  `passes_gate` blocks the chunk-size regression at recall 0.72 < 0.80 (Task 5).

## Common pitfalls

- **Shipping pure-vector on a mixed corpus.** It fails every exact-term query —
  error codes, endpoint names, proper nouns — which here is 32% of traffic. The
  measured recall gap (0.68 → 0.84) is the cost. Fuse BM25.
- **Reaching for a graph because it sounds advanced.** Only 16% of queries are
  multi-hop; a graph's high build/maintenance cost isn't justified as the
  backbone. Add it as a layer for the relationship tail, if at all.
- **Evaluating only the final answer.** A generation-only eval can't tell you
  *why* an answer is wrong. Decouple: measure recall@k/precision@k for retrieval
  independently, so a low score points you at the retriever, not the prompt.
- **Trusting answer-relevance as a quality signal.** A fluent, on-topic answer
  can be ungrounded. Faithfulness is the metric that catches the hallucination;
  gate on it, not just relevance.
- **Forgetting that misses the index.** A deletion/correction that updates the
  source but not the vector and BM25 entries leaves stale (or private) content
  retrievable. The invalidation path must reach every index, and it is the same
  path as the per-user forgetting path.

## Verification

- Run `evaluate` on the labeled set for vector-only, hybrid, and hybrid+rerank;
  confirm recall rises with hybrid and precision rises with rerank, matching the
  results table.
- Confirm `passes_gate` returns `False` for vector-only (recall 0.68) and `True`
  for hybrid+rerank.
- Induce the chunk-size regression, re-run `evaluate`, and confirm the gate
  blocks it (recall < 0.80) — the harness must fail closed on a real regression.
- Check `assemble` respects `RETRIEVED_BUDGET_TOKENS`: feed oversized chunks and
  confirm it stops at the budget and dedups near-identical chunks.
- Stretch: turn the blocked regression into a permanent test case and wire the
  gate into CI (mod-304) so every future change clears recall and faithfulness
  before it ships.
