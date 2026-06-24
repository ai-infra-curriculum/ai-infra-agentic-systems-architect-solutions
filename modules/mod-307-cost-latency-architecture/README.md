# mod-307-cost-latency-architecture — Solutions

Reference solutions for the Agentic Systems Architect module on cost and latency
architecture. The three exercises build on each other against one running system —
a competitive-intelligence fan-out agent — so the numbers carry forward: exercise-01
prices the baseline, exercise-02 cuts it with levers, exercise-03 governs it.

All dollar, token, latency, and quality figures in these solutions are
**illustrative-pending-verification** — they use the Chapter teaching tables, not a
live provider pricing page. The method is durable; the numbers are not.

## Exercise index

- [exercise-01-token-economics-cost-model](exercise-01-token-economics-cost-model/README.md)
  — price every model call one brief triggers, decompose the cost by phase (workers
  dominate at ~93%), roll up to per-tenant and run-rate (mean vs. p95 tenant), run a
  worker × turns sensitivity sweep, and apply the value-threshold gate. Baseline:
  ≈$0.9135/brief.
- [exercise-02-caching-and-routing-architecture](exercise-02-caching-and-routing-architecture/README.md)
  — design the three levers (route → cache → batch) against the baseline, with each
  precondition stated and multiplier quantified, plus a routing-architecture diagram.
  Interactive design point: ≈$0.2370/brief (−74.1%).
- [exercise-03-cost-latency-quality-budgets](exercise-03-cost-latency-quality-budgets/README.md)
  — the capstone: per-surface cost–latency–quality budgets, a p95/p99 SLA table, the
  governed reference architecture (meter, budget guard, kill switch, hard caps), and
  three ADRs defending the trades.
