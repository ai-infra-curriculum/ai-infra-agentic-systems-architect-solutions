# mod-303-memory-context-architecture — Solutions Index

Reference solutions for the Memory & Context Architecture module. Each exercise
treats context and memory as a first-class architectural concern: budgets,
placement diagrams, and decision matrices a team can build against — with the
failure modes named and the trade-offs made explicit.

Each linked solution is a worked artifact, not a one-paragraph answer key. Read
the paired learning exercise first; the solution overwrites the stub with an
`## Approach`, a `## Reference solution` (the artifact itself, code where it
earns its place), `## Meeting the acceptance criteria`, `## Common pitfalls`,
and `## Verification`.

## Exercises

- [exercise-01 — Context budget and rot analysis](exercise-01-context-budget-and-rot-analysis/README.md):
  per-turn window telemetry, a filled budget worksheet with an eviction rule per
  line item, a quality-vs-window-length curve that locates the rot, and a
  distractor probe that confirms (or refutes) lost-in-the-middle.
- [exercise-02 — Memory-tier placement design](exercise-02-memory-tier-placement-design/README.md):
  every piece of agent state placed into a tier with a schema-level scope key,
  the five boundary checks answered with specifics, a promotion/distillation
  flow, and a provable forgetting path.
- [exercise-03 — Compaction and note-taking strategy](exercise-03-compaction-and-notetaking-strategy/README.md):
  a fidelity contract, a token-threshold compaction trigger that persists the
  raw span, a typed task ledger, an automated fidelity check, and a re-anchoring
  demonstration.
- [exercise-04 — RAG architecture for agents](exercise-04-rag-architecture-for-agents/README.md):
  question-shape classification, a store chosen against the selection matrix,
  an over-fetch → rerank → budget-aware assembly topology, a freshness strategy
  with an invalidation path, and a retrieval-eval gate that catches an induced
  regression.

## How to use these

- The artifacts (budget worksheet, placement table, decision matrix) are the
  deliverable at this altitude. The code exists to *prove* the artifact holds —
  it is illustrative, provider-agnostic, and runnable with the standard library
  plus a token counter where noted.
- Numbers (occupancy ceilings, budget slices, gate thresholds) are defended,
  not invented: each traces to a measurement or a stated trade-off. Treat the
  values as a starting allocation to re-derive for your own workload.
