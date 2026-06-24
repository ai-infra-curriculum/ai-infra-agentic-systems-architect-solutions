# mod-307-cost-latency-architecture/exercise-02-caching-and-routing-architecture — Solution

This is the reference solution for the caching-and-routing architecture. It takes
the exercise-01 baseline ($0.9135/brief, illustrative-pending-verification) and
cuts it by designing the three Chapter 2 levers — prefix caching, model routing,
and batch processing — as architecture, with each lever's precondition stated and
its multiplier quantified against the baseline. All prices and multipliers are
**illustrative-pending-verification** (Chapter 2 teaching figures).

## Approach

Chapter 2 is explicit about order of operations: **route first, cache second,
batch last.** The reasoning is that routing scales every downstream number (it
changes the per-token price every other lever then discounts), caching only pays
where a prefix is reused before its TTL, and batching only applies to
latency-tolerant lanes. The solution follows that order and refuses to apply a
lever where its precondition fails — a cold cache that only pays write premiums or
a "batchable" job a user is blocked on is a regression, not a saving.

The single most important framing carried from exercise-01: **the worker loops are
92.9% of cost.** So both routing and caching are aimed squarely at the workers.
Decomposition and synthesis need the orchestrator's reasoning and stay on the big
model; the five extraction/distillation workers do not — they read pages and emit
structured findings, which the small model clears. That is the whole game.

## Reference solution

### 1. Lever inventory — where each one applies

One row per call type, with the precondition that gates the lever.

| Call | cache? | route-to | batch? | Precondition justification |
| --- | --- | --- | --- | --- |
| Orchestrator decompose | no | big | no | One call per brief, no shared prefix reuse; reasoning-heavy, cannot downshift; user waits |
| Worker (extraction loop) | **yes** | **small** | no (interactive) | Shares a stable preamble across 4 turns × 5 siblings (cache fires); reads/distils pages — clears small-model eval gate; user waits on the interactive brief |
| Orchestrator synthesize | no | big | no | One call per brief; writes the final brief, reasoning-heavy, cannot downshift; user waits |
| Watchlist nightly worker | yes | small | **yes** | Same worker shape, but the nightly re-run is latency-tolerant (24h window) → batch lane |
| Eval / regression run | n/a | small | **yes** | Canonical batch workload; no user waits on eval traffic |

No lever is applied where its precondition fails: the orchestrator calls are not
cached (no reuse), not downshifted (reasoning), and not batched (interactive); the
interactive worker is cached and routed but **not** batched because a user is
waiting on it.

### 2. Cache layout and multiplier

**Cache-first layout.** The worker prompt is ordered so the entire stable span
sits at the front and the volatile per-turn content comes after it:

```text
[ cacheable prefix ]                          ← marked cache_control: ephemeral
  system prompt           ~1,500 tok
  tool schema             ~2,000 tok
  shared assignment frame ~  500 tok
                          ─────────
                          ~4,000 tok stable
[ volatile suffix ]                           ← NOT cached; changes every turn
  this worker's sub-question
  retrieved pages so far
  prior tool outputs
```

The cache matches on a prefix, so one early-changing token invalidates everything
downstream — the per-worker sub-question must come *after* the cached span, not
woven into the system prompt.

**Multiplier on the baseline.** The 4,000-token prefix is re-sent on every worker
turn and across every sibling: `reuses = 4 turns × 5 workers = 20`. At the small
model's input price ($0.80/M, post-routing):

```text
cache write factor = 1.25×   cache read factor = 0.10×   (illustrative)

uncached prefix cost = 4,000 × 20 / 1e6 × $0.80          = $0.0640
cached prefix cost   = (4,000×1.25 + 4,000×0.10×19) / 1e6 × $0.80
                     = (5,000 + 7,600) / 1e6 × $0.80      = $0.0101
saving on cached input tokens                            = $0.0539  (≈ 84%)
```

**Optional confirmation.** The cache check is cheap because the usage object
reports the split directly. A cache you did not confirm is a slide, not a saving:

```python
import anthropic

client = anthropic.Anthropic()
SHARED_PREAMBLE = "..."  # the ~4k-token stable worker prefix

for i in range(2):  # call 1 writes the entry, call 2 should read it
    r = client.messages.create(
        model="claude-3-5-haiku-latest",
        max_tokens=64,
        system=[{"type": "text", "text": SHARED_PREAMBLE,
                 "cache_control": {"type": "ephemeral"}}],
        messages=[{"role": "user", "content": f"ping {i}"}],
    )
    u = r.usage
    print(i, "write:", u.cache_creation_input_tokens,
          "read:", u.cache_read_input_tokens)

# Expected: i=0 -> write ≈ 4000, read 0 ; i=1 -> write 0, read ≈ 4000.
```

### 3. Routing table

**Static (by role), not dynamic.** The input stream is *not* genuinely mixed —
every brief decomposes into the same five sub-question roles, so a classifier would
pay an overhead on every request to re-derive a routing decision the architecture
already knows. Reach for dynamic routing only when role-assignment leaves money on
the table; here it does not.

| Role / call | Assigned model | Eval gate before downshift is allowed |
| --- | --- | --- |
| Orchestrator decompose | big | none — stays on big (reasoning) |
| Extraction worker | **small** | distillation eval set must clear the per-call quality bar (faithfulness + coverage ≥ baseline) on a real test set before the downshift ships |
| Orchestrator synthesize | big | none — stays on big (reasoning) |

Every downshift is tied to an eval gate (the mod-304 hook): you only move the
worker to the small model *after* it clears the worker's quality bar on a held-out
test set, so quality does not silently regress on the long tail.

**Routing multiplier.** Re-pricing with the five workers downshifted to the small
model (decompose and synthesize unchanged):

```text
baseline task (all big)            = $0.9135
routed task (workers → small)      = $0.2909
routing cut vs. baseline           = 68.2%
```

Routing alone is the dominant lever for this system, because the dominant cost term
(the workers) is exactly the part that downshifts cleanly.

### 4. Batch lane

The interactive brief is **not** batchable — a user waits on it. The latency-
tolerant work lives adjacent to the interactive surface:

| Workload | Latency budget that makes batching safe | Multiplier |
| --- | --- | --- |
| Watchlist nightly refresh (~500 companies/night) | "Results by morning is acceptable — no analyst waits overnight; the 24h window covers it" | 0.50× on the lane |
| Eval / regression runs | "Eval traffic blocks no user by definition" | 0.50× on the lane |

**Batch multiplier on the watchlist lane.** Per brief, routed + cached =
≈$0.2370. Batched at 50% off:

```text
per-brief (routed + cached, sync)  = $0.2370
per-brief (routed + cached, batch) = $0.1185
500 briefs/night sync              = $118.49
500 briefs/night batch             = $59.24   (saves ~$59/night on that lane)
```

### 5. Stack the levers and diagram

Applying the levers in order — **route → cache → batch** — against the
exercise-01 baseline:

```text
                                   per-task $    cut vs. baseline
  baseline (all big, no cache)      0.9135          —
  + route workers → small           0.2909        −68.2%
  + cache shared worker prefix      0.2370        −74.1%   (cache adds −5.9 pts)
  ───────────────────────────────────────────────────────
  interactive design point          0.2370        −74.1% before/after

  batch (watchlist lane only)       0.1185        −87.0%   (not the interactive lane)
```

Per-lever contribution to the interactive brief: routing does the heavy lifting
(−68.2%), caching adds another ~6 points (−74.1% stacked); batching is a separate
lane (it does not touch the interactive number — a user waits on that brief).

**Routing-architecture diagram.**

```text
                         ┌────────────────────┐
        request ────────▶│  router (static    │   by role: decompose/synth → big
                         │  by role)          │           workers → small
                         └─────────┬──────────┘
                  hard / reasoning │ easy / bulk
                          ┌────────┴─────────┐
                          ▼                  ▼
                  ┌───────────────┐   ┌───────────────┐
                  │  BIG model    │   │  SMALL model  │
                  │ decompose,    │   │ 5 extraction  │
                  │ synthesize    │   │ workers       │
                  └──────┬────────┘   └───────┬───────┘
                         │  fan-out (≤5)      │
                         ▼ share ONE cached prefix (4k tok)
                  ┌──────────────────────────────────┐
                  │  [cached preamble] + [volatile]   │  ← cache-first layout
                  │   system+tools+frame  per-turn    │
                  └──────────────┬───────────────────┘
                         meter every call (in/out/cache_read)
                                 ▼
                  ┌──────────────────────────────────┐
                  │  ASYNC / BATCH LANE (~50% off)    │  ← watchlist nightly,
                  │  watchlist refresh · eval runs    │     eval runs only
                  └──────────────────────────────────┘
```

## Meeting the acceptance criteria

- **Every call tagged with applicable levers and preconditions; no lever applied
  where its precondition fails.** The Task-1 table tags each call cache/route/batch
  with a precondition; orchestrator calls are explicitly left un-cached,
  un-downshifted, and un-batched, and the interactive worker is explicitly not
  batched.
- **Cache layout is cache-first, names the exact cacheable span, reports the
  read-discount saving.** The 4,000-token system+tools+frame prefix is named, the
  volatile suffix is placed after it, and the saving is $0.0539 (≈84%) over 20
  reuses, with the optional `cache_read_input_tokens` confirmation snippet.
- **Routing table assigns the cheapest viable model per call and ties each
  downshift to an eval gate, with static-vs-dynamic justified.** Workers → small
  behind a distillation eval gate; static-by-role chosen because the stream is not
  genuinely mixed.
- **Batch lane specified with a stated latency budget per workload and a computed
  discount.** Watchlist (24h window) and eval runs, both 0.50× on the lane.
- **Stacked before/after reported against the exercise-01 baseline with per-lever
  contributions.** $0.9135 → $0.2370 interactive (−74.1%): route −68.2%, cache
  −5.9 pts; batch −87.0% on the watchlist lane.
- **Diagram shows router, tiers, shared cached prefix, meter, and sync/batch
  split.** All five elements are present.

## Common pitfalls

- **Caching before routing.** If you cache on the big model first and then
  downshift, you redo the cache math, and the headline saving is mis-attributed.
  Route first so caching discounts the already-cheaper per-token price.
- **Caching a prefix that does not survive its TTL.** The cache only pays when the
  prefix is reused before it goes cold. The worker loop (4 turns) and fan-out (5
  siblings) reuse it 20× within one brief — that fires. A once-per-day call with a
  unique prefix would pay the 1.25× write premium and never collect the read
  discount; do not cache it.
- **Weaving the volatile sub-question into the system prompt.** A cache matches on
  the prefix; one early-changing token invalidates the whole entry. The per-worker
  sub-question must sit *after* the cached span or the hit rate collapses to zero.
- **Downshifting without an eval gate.** A worker that looks fine in a demo on the
  small model can degrade on the long tail of multi-hop questions. The downshift is
  only a saving once the small model clears the worker's eval bar on a real test
  set; otherwise it is a silent quality regression.
- **Batching something a user waits on.** The interactive brief is not batchable.
  Putting it in the 50%-off lane to chase the discount trades a real SLA breach for
  a small cost win — the precondition (latency tolerance) fails.

## Verification

```bash
# Re-run the multiplier worksheet against the exercise-01 baseline.
python3 multipliers.py
# Expected (illustrative): routed $0.2909 (−68.2%), routed+cached $0.2370 (−74.1%),
#                          batched watchlist brief $0.1185.

# Optional: confirm the cache actually fires (two calls share the prefix).
python3 confirm_cache.py
# Expected: call 0 writes ~4000 tokens, call 1 reads ~4000 tokens.
```

Checklist for a complete submission:

- [ ] `levers.md` carries the tagged call inventory (with preconditions), the
      cache-first layout naming the exact span, the routing table with eval gates,
      and the batch-lane spec with latency budgets.
- [ ] `routing-architecture.{txt,png}` shows router, big/small tiers, shared cached
      prefix across siblings, the meter, and the sync/batch split.
- [ ] `multipliers.{py,xlsx}` reports per-lever and stacked before/after numbers
      against the exercise-01 baseline, and re-flows if the baseline changes.
- [ ] If the optional cache check was run, `cache_read_input_tokens` is reported.
- [ ] Every multiplier and price is labelled illustrative-pending-verification.
- [ ] `NOTES.md` answers the three reflection prompts (biggest lever under bursty
      traffic, a tempting-but-gated downshift, a precondition that failed).
