# mod-302-multi-agent-orchestration/exercise-01 — Solution

## Approach

The exercise asks for a topology *and* a defended budget — not a diagram in
isolation. The discipline that makes it an architect's artifact is that the token
estimate has to come out of the topology, and the topology has to be sized so the
estimate clears the value bar. So the work runs in one direction: decide whether
fan-out is even earned, draw the shape that the workload's independent breadth
implies, then prove the `~15x` multiplier buys something the brief actually wants.

Three design choices drive the reference solution below:

- **Fan-out earns its keep here for one reason: independent breadth.** Five
  providers, researched on the same five axes, with no provider's findings feeding
  another's. That is the textbook fan-out shape from Chapter 1 — N independent
  sub-tasks that each overflow a shared single-agent context if you tried to run
  them in one window (five providers × five axes × incident search is dozens of
  tool calls). The justification is breadth and context isolation, not raw speed,
  though parallel latency is a real secondary win.
- **Flat, not hierarchical.** One decomposition layer expresses the task
  completely: provider is the natural unit of independence. A second layer
  (a sub-orchestrator per provider fanning out to one worker per axis) would
  compound the multiplier (Chapter 1: hierarchy nests a ~15x system inside a
  worker slot) to buy nothing — the per-axis work inside one provider is small and
  sequential enough for a single worker to hold. Hierarchy is rejected explicitly,
  not skipped silently.
- **The distilled return is the whole budget.** A provider worker that hands back
  its raw transcript (search results, page dumps, scratch reasoning) would force
  the orchestrator's synthesis context to carry five full research sessions. The
  worker instead returns a fixed structured summary — five scored fields plus a
  short incident note — so orchestrator context grows by *summaries × 5*, not
  *transcripts × 5*. That ratio, not N, is what keeps the multiplier in band.

The honest counter-case, surfaced rather than hidden: at N = 5 this is squarely
worth it, but the workload sits near the *low* edge where fan-out pays. Drop it to
two providers on three axes and a single sequential agent wins — the boilerplate
duplication (five copies of framing + tool schemas) stops being amortized across
enough independent work to justify the `~15x`.

## Reference solution

### Decision: multi-agent, bounded fan-out

Multi-agent is justified. The workload decomposes into five **independent** units
(one per provider) that share no data dependency — the Postgres-A research never
needs the Postgres-B findings — and the combined breadth (5 providers × 5 axes +
per-provider incident search) overflows a single clean context. That independent
breadth is the specific property from Chapter 1 that earns fan-out; it is not
specialization (the workers are near-identical) and only secondarily context
overflow. A sequential chain would serialize five unrelated research sessions for
no dependency reason and throw away the parallel-latency win.

### Topology

```text
TOPOLOGY: flat orchestrator-worker      FAN-OUT CAP N = 5 (one per provider)

                ┌──────────────┐
   task ───────▶│ orchestrator │  decompose: 1 provider-research task per provider
                └──────┬───────┘
        ┌────────┬─────┼─────┬────────┐
        ▼        ▼     ▼     ▼        ▼
   ┌────────┐┌────────┐┌─ ─┐┌────────┐┌────────┐
   │provider││provider││...││provider││provider│   isolated context each;
   │   #1   ││   #2   ││   ││   #4   ││   #5   │   identical instruction shape
   └───┬────┘└───┬────┘└─┬─┘└───┬────┘└───┬────┘
       │ distilled return (5 scored fields + incident note) per worker
       └────────┴─────┬─────┴────────┴────────┘
                      ▼
                ┌──────────────┐
                │ orchestrator │  synthesize ──▶ comparative brief
                └──────────────┘   (build the cross-provider comparison table)
```

Each worker owns exactly one provider and researches all five axes for it; the
orchestrator owns only decompose + synthesize. The cross-provider comparison — the
actual value of the brief — is built once, at synthesis, from the five distilled
returns. No worker sees another worker's output, and no worker sees the
conversation.

### Worker assignment table

Every worker runs the same self-contained instruction with the provider name bound
in. The assignment is non-overlapping by construction (provider is the partition
key) and self-contained (the worker is handed the provider name, the five axes, and
the return schema — it never needs the orchestrator's conversation).

| worker_role | instruction summary | tools needed | expected return shape |
| --- | --- | --- | --- |
| provider-researcher (×5, one per provider) | "Research provider `<X>` on exactly these five axes: pricing model, autoscaling behavior, backup/restore guarantees, region coverage, recent-incident summary (last 12 months). Cite a source URL per axis. Do not compare to other providers." | web search, page fetch, provider status-page/docs reader | fixed JSON: `{provider, pricing, autoscaling, backup_restore, regions, recent_incident, sources[]}` — each axis a 1–3 sentence finding + URL |

The "do not compare to other providers" clause is load-bearing: it keeps each
worker's scope to its own provider so returns stay non-overlapping and the
comparison stays a single orchestrator responsibility.

### Token-economics estimate

```text
TOKEN ENVELOPE

est ≈ orchestrator_overhead                       (decompose call: ~1 unit)
    + N × (shared_context + worker_work + distilled_return)
    + synthesis_pass                              (build comparison: ~1.5 units)

  N = 5
  shared_context   ≈ 1 unit   (framing + 5 axes + tool schemas, re-paid per worker)
  worker_work      ≈ 4 units  (5-axis research + incident search, multi-turn)
  distilled_return ≈ 0.3 unit (5 fields + short note, NOT the transcript)

  distilled_return / worker_work = 0.3 / 4 ≈ 0.08      (target ≪ 1 — met)

  est ≈ 1 + 5 × (1 + 4 + 0.3) + 1.5
      ≈ 1 + 26.5 + 1.5  ≈ 29 units

  single-chat baseline for the same five-provider sweep ≈ 2 units of useful work
  estimated multiplier vs single chat ≈ 29 / 2 ≈ 14–15x

  verdict: JUSTIFIED — the brief's value is breadth + freshness across 5 providers
           delivered in roughly one-worker wall-clock, not 5x serial time.
```

The distilled-return ratio is the figure that defends the band. If workers returned
**full transcripts** instead (`distilled_return ≈ 4 units`, equal to the work), the
orchestrator's synthesis context would carry `5 × 4 = 20` units of raw material
instead of `5 × 0.3 = 1.5`, and the synthesis pass itself would balloon to re-read
all of it — pushing the estimate toward `~48 units` and a `~24x` multiplier. The
distillation is a roughly **40% cost reduction** on this workload and the difference
between "in band" and "runaway."

### Bounds and degradation

```text
| Decision           | Choice                          | Why                          |
| ------------------ | ------------------------------- | ---------------------------- |
| Multi-agent?       | yes, flat fan-out               | 5 independent provider units |
| Topology shape     | orchestrator + N workers (flat) | one layer expresses the task |
| Fan-out cap N      | 5 (== provider count; hard cap) | bounded by the workload      |
| Return distillation| structured summary, ratio ≈0.08 | dominant lever on cost       |
| Failure behavior   | degrade + flag (see below)      | partial failure is normal    |
```

- **Fan-out cap:** N = 5, hard. The provider list is the cap — there is no path
  that spawns a sixth worker. A request to "also cover the next three providers"
  re-runs decompose with N = 8, surfaced in the spec, never an implicit expansion.
- **Per-worker turn cap:** 8 turns. Five axes plus an incident search is ~5–7 tool
  calls; 8 leaves one turn of slack. A worker that hits the cap returns what it has
  with the unfinished axes marked `unavailable`, it does not loop.
- **2-of-5 failure:** the orchestrator synthesizes from the 3 returns it has and
  ships a brief that **names the gap explicitly** — "covered 3 of 5 providers;
  research for `<X>` and `<Y>` failed and is omitted" — and never fabricates the two
  missing providers' rows. A confident five-row table that silently invented two
  rows is the bug this rule exists to prevent (Chapter 4, partial failure).

## Meeting the acceptance criteria

- **Multi-agent-vs-single decision stated and tied to a property** — "yes, because
  five *independent* provider units (independent breadth) overflow a single clean
  context," with the chain alternative rejected for serializing unrelated work.
- **Topology diagram labels N, roles, and distilled-return flow** — N = 5 is on the
  diagram, the worker role is `provider-researcher`, and the distilled return
  (5 fields + incident note) is the labeled edge flowing back to synthesis.
- **Assignment rows self-contained and non-overlapping** — one row, instantiated
  five times with the provider bound in; partition key is provider, and the
  "do not compare" clause enforces non-overlap. Each worker gets axes + schema and
  never needs the conversation.
- **Token estimate shows the distilled-return ratio and a defended band** — ratio
  ≈ 0.08, multiplier `~14–15x`, with the full-transcript counterfactual (`~24x`)
  shown to prove the ratio is doing the work.
- **Fan-out cap, turn cap, and 2-of-5 behavior all specified** — N = 5 hard,
  8 turns/worker, and degrade-and-flag with an explicit no-fabrication rule for the
  missing providers.

## Common pitfalls

- **Fanning out dependent work.** If you let one worker's findings feed another's
  (e.g., "research the cheapest provider's backup story in more depth"), you have
  invented a dependency and fan-out now re-establishes context across the seam —
  that is a chain, not a fan-out, and it wastes the multiplier. Keep the five
  provider units strictly independent; do any cross-provider reasoning at synthesis.
- **Letting workers return transcripts.** The single most common cost blowup. A
  worker that hands back its raw research session pushes the multiplier from `~15x`
  toward `~24x+` and makes synthesis re-read everything. Mandate the structured
  return schema and treat an un-distilled worker as a defect.
- **Reaching for hierarchy.** A sub-orchestrator per provider feels organized but
  nests a ~15x system inside each of five worker slots for a per-provider task that
  is small and mostly sequential. The compounded cost buys nothing here. Hierarchy
  is for decompositions a single layer genuinely cannot express.
- **Unbounded N.** "One worker per provider" is safe only because the provider list
  is fixed at five. A decomposition keyed off an open-ended set (every provider the
  model can name) is a cost incident; the cap must be an explicit architectural
  number, not whatever the model decides to spawn.
- **A failure path that fabricates.** When 2 of 5 workers fail, the tempting output
  is a tidy five-row table. Filling the two missing rows with plausible-sounding
  guesses is the worst outcome — worse than the honest 3-row brief. The degrade rung
  must flag the gap, never paper over it.

## Verification

A completed submission is correct when:

- The topology diagram labels N, names the worker role, and shows a *distilled*
  return edge (a summary shape, not "returns its findings") flowing into a separate
  synthesis step.
- The assignment table's rows are self-contained (worker gets provider + axes +
  return schema, never "the conversation") and non-overlapping (a single partition
  key — provider — with an explicit no-cross-comparison clause).
- The token envelope shows an actual `distilled_return / worker_work` ratio that is
  `≪ 1`, and the multiplier is defended against the full-transcript counterfactual,
  not just asserted.
- All three bounds are concrete numbers: a hard N, a numeric per-worker turn cap,
  and a 2-of-5 behavior that degrades-and-flags with a stated no-fabrication rule.
- `NOTES.md` answers the three prompts: the workload size at which a single agent
  wins (roughly ≤ 2 providers / 3 axes, where boilerplate stops amortizing), the
  full-transcript context blowup (the `~24x` figure and why synthesis compounds it),
  and where hierarchy would pay (it would not on *this* workload — say so, and name
  the larger decomposition where it would).
