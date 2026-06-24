# mod-305-observability-tracing/exercise-03-observability-platform-evaluation — Solution

## Approach

The deliverable is a **defensible build-vs-buy decision**, not a feature-checklist
bake-off. An architect derives requirements from the *specific* system, weights
them, runs a *real* proof-of-concept on their own traces, scores from the POC
(not the marketing page), and writes an ADR that names what it gives up. The four
artifacts, in order:

1. **`REQUIREMENTS.md`** — the system in three sentences, then weighted criteria.
   The weights *are* the architecture: a healthcare agent weights self-host at 5;
   a hackathon prototype weights it at 1.
2. **The POC** — ingest the exercise-01/02 traces into **two** of {LangSmith,
   Langfuse, Arize Phoenix}, confirm GenAI-convention hierarchy fidelity, run one
   LLM-judge eval joined by `trace_id`, exercise the deploy comparison from
   Chapter 3, and curate one regression dataset. Record worked / painful /
   impossible per platform.
3. **The scoring matrix** — criteria × platforms, every cell a POC-derived 1–5,
   weighted totals computed; the recommendation follows from the numbers.
4. **`ADR-observability-platform.md`** — context, options, decision, the
   trade-offs given up, and the explicit revisit condition.

The chosen system-under-evaluation for this reference: a **regulated-data research
agent** (orchestrator + workers + RAG), prompts carry user PII, projected mid-six-
figure trace volume. That sensitivity is what drives the weights — and it makes
self-host and OTel-native ingestion decisive. A different system would re-weight
and could legitimately flip the recommendation; that is the point of the exercise.

## Reference solution

### Requirements, weighted to this system (`REQUIREMENTS.md`)

```text
SYSTEM UNDER EVALUATION (3 sentences)
  An orchestrator-worker research agent with a RAG retrieval step, instrumented
  to the OTel GenAI conventions (exercise-01), emitting faithfulness/quality
  scores joined by trace_id (exercise-02). Prompts and retrieved context carry
  customer PII, so trace data is regulated and a third-party SaaS that egresses
  it is a hard problem. Projected volume is ~500k traces/month within a year,
  growing — so cost-at-scale and self-host economics are first-order, not later.

WEIGHTED CRITERIA (weight 1–5, justified by THIS system)
  OTel-native ingestion        5  instrumentation must stay portable; a
                                  proprietary-SDK-only backend re-locks us in.
  self-host / data residency   5  PII in prompts; "ship to third-party SaaS"
                                  may be non-starter → must run in our VPC.
  eval integration             4  the quality plane (faithfulness/drift) lives
                                  or dies on join-by-trace_id eval support.
  cost model at scale          4  500k→multi-M traces/month; per-seat/per-trace
                                  pricing can swing annual cost 10x.
  datasets / experiments       3  regression sets + prompt-diff turn obs into an
                                  improvement loop, not a read-only dashboard.
  drift / embedding analysis   3  Chapter-2 drift is a named requirement for a
                                  RAG agent whose retrieval index changes.
  framework coupling           2  OTel-agnostic; we don't want a bet on one
                                  orchestration framework.
  operational maturity         3  RBAC/SSO/retention controls matter for
                                  regulated data; alerting + dashboards-as-code.
```

### The proof-of-concept (two platforms: Langfuse + Arize Phoenix)

Both self-host trivially and are OTel-native, which is exactly what this
regulated system needs — so they are the two carried to a real POC. LangSmith is
scored from its free tier and docs as the third column for contrast, but the
deciding two were run on our own traces.

```bash
# Phoenix: local OTLP collector + UI in one container.
docker run -p 6006:6006 -p 4318:4318 arizephoenix/phoenix:latest
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318 python agent.py   # exercise-01

# Langfuse: self-host via compose, OTLP endpoint on /api/public/otel.
docker compose -f langfuse/docker-compose.yml up -d
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:3000/api/public/otel \
  OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic <pk:sk base64>" \
  python agent.py
```

POC findings, per platform — *worked / painful / impossible*:

```text
PHOENIX
  worked   : OTLP/OpenInference ingest renders invoke_agent → chat/execute_tool
             hierarchy faithfully; retry.attempt + ERROR spans show in-context.
             Best-in-class drift/embedding clustering on the faithfulness signal.
             Local self-host is a single container — zero data egress.
  painful  : dataset/experiment + prompt management thinner than LangSmith;
             production-scale story leans on Arize's commercial platform.
  impossible: nothing blocking for our POC scope.

LANGFUSE
  worked   : OTel-native ingest; GenAI spans render with correct nesting and
             attributes. Eval runner joins LLM-judge faithfulness by trace_id;
             group-by service.version reproduces the Chapter-3 regression view.
             Self-host (MIT core) keeps PII in our VPC; dataset curation + prompt
             versioning are first-class. Cost control at scale is the strength.
  painful  : eval-tooling depth slightly behind LangSmith; some convention
             attributes needed a mapping check on ingest.
  impossible: nothing blocking.

LANGSMITH (free tier / docs — not the deciding POC for this system)
  worked   : deepest eval + dataset + prompt tooling; polished deploy-compare.
  painful  : OTel ingest supported alongside a native SDK — verify convention
             coverage; primarily SaaS.
  impossible (for THIS system): in-VPC self-host only on enterprise tier → the
             PII residency constraint makes default SaaS a non-starter.
```

### The scoring matrix (POC-derived)

Cells are 1–5 from the POC above, not the marketing page. Weighted total =
Σ(weight × score).

```text
criterion (weight)             | Langfuse | Phoenix | LangSmith
-------------------------------|----------|---------|----------
OTel-native ingest (5)         |    5     |    5    |    4
self-host / residency (5)      |    5     |    5    |    2
eval integration (4)           |    4     |    5    |    5
cost at scale (4)              |    5     |    5    |    3
datasets/experiments (3)       |    4     |    4    |    5
drift / embedding (3)          |    3     |    5    |    3
framework coupling (2)         |    5     |    5    |    4
ops maturity (3)               |    4     |    3    |    5
-------------------------------|----------|---------|----------
weighted total                 |   129    |   136   |   109
```

Weighted-total arithmetic (Σ weight × score):

```text
Langfuse  : 5·5 +5·5 +4·4 +4·5 +3·4 +3·3 +2·5 +3·4
          = 25+25+16+20+12+ 9+10+12                       = 129
Phoenix   : 5·5 +5·5 +4·5 +4·5 +3·4 +3·5 +2·5 +3·3
          = 25+25+20+20+12+15+10+ 9                       = 136
LangSmith : 5·4 +5·2 +4·5 +4·3 +3·5 +3·3 +2·4 +3·5
          = 20+10+20+12+15+ 9+ 8+15                       = 109
```

The numbers recommend **Phoenix** for this regulated, drift-sensitive system —
it ties Langfuse on the two weight-5 criteria (OTel-native, self-host) and pulls
ahead on eval depth and the drift/embedding analysis a RAG agent needs.
LangSmith's residency penalty (weight-5 × score-2) is what sinks it *for this
system*, despite the best tooling.

### The ADR (`ADR-observability-platform.md`)

```text
# ADR — Observability platform for the regulated research agent

Status: accepted    Date: 2026-06    Decision owner: systems architect

## Context
A regulated-data orchestrator-worker RAG agent, OTel GenAI-instrumented, with
faithfulness/drift scores joined by trace_id. Prompts carry PII; projected
~500k traces/month and rising. We must choose where traces land, weighing data
residency, OTel portability, eval depth, drift analysis, and cost at scale.

## Options considered
- Langfuse (OSS, self-host)   — weighted 129
- Arize Phoenix (OSS, self-host) — weighted 136  ← chosen
- LangSmith (SaaS)            — weighted 109 (residency penalty)
- Build (OTel Collector → ClickHouse → Grafana + homegrown eval/dataset) —
  the tracing half is tractable; the eval/dataset/drift PRODUCT is not.

## Decision
Adopt Arize Phoenix, self-hosted in our VPC, as the primary observability +
eval backend. Keep instrumentation OTel-native so the backend stays swappable.

## Consequences — what we give up
- We accept Phoenix's thinner prompt-management / dataset-experiment tooling
  versus LangSmith; we backfill prompt versioning in our own repo + CI.
- We own the self-host operations (upgrades, storage, retention) — the cost of
  keeping PII in-VPC. Langfuse would have been a near-tie and is the fallback
  if Phoenix's production-scale path proves heavier than its commercial tier.
- We forgo LangSmith's polish to satisfy the residency constraint.

## Revisit condition
Re-run this decision if (a) trace volume exceeds ~2M/month — re-model self-host
storage + the Arize commercial tier against Langfuse's cost curve; or (b) the
data-residency constraint is lifted (no PII in prompts) — which removes the
weight-5 self-host penalty and re-opens LangSmith; or (c) we standardize on one
orchestration framework with a deeply-coupled native backend.
```

## Meeting the acceptance criteria

- **`REQUIREMENTS.md` states the system and assigns justified weights** — the
  three-sentence topology/sensitivity/scale statement drives weight-5 on
  self-host and OTel-native ingestion, explicitly because prompts carry PII and
  volume is projected high.
- **The POC was real** — exercise-01/02 traces ingested into **two** self-hosted,
  OTel-native platforms (Langfuse, Phoenix) via local containers, with per-
  platform notes on hierarchy fidelity, eval join-by-`trace_id`, the
  `service.version` deploy comparison, and dataset curation.
- **The matrix uses POC-derived scores and computes weighted totals** — every
  cell traces to a POC finding; totals (Langfuse 129, Phoenix 136, LangSmith 109)
  are shown with the arithmetic, and the recommendation follows the numbers.
- **The ADR names the decision, the trade-offs given up, and a concrete revisit
  condition** — it accepts thinner prompt tooling + self-host ops cost, names
  Langfuse as the fallback, and sets quantified revisit triggers (volume > 2M/mo;
  residency lifted; framework standardization).

## Common pitfalls

- **Scoring from the marketing page, not the POC.** Demos lie; your own traces
  don't. A platform that *demos* best often *scores* worse once you ingest real
  GenAI-convention spans and check whether it flattens `invoke_agent`/
  `execute_tool` instead of nesting them. Score the cell from what the POC did.
- **Unweighted (or uniformly weighted) criteria.** A flat matrix is a feature
  checklist in disguise. The weights tied to *this* system's sensitivity and
  scale are the architecture; without them the recommendation isn't defensible.
- **Skipping the eval-join and deploy-comparison in the POC.** Ingesting traces
  is the easy half. The decisive question is whether the platform runs an
  LLM-judge on captured traces and joins by `trace_id`, and whether you can group
  a quality signal by `service.version` — exercise those or the POC proves nothing
  about the quality plane.
- **An ADR with no trade-off and no revisit condition.** "We chose X because it's
  best" is not an ADR. An honest one names what you give up (here: prompt tooling,
  self-host ops) and the explicit condition that flips the decision.
- **Forgetting that the recommendation is system-specific.** The same matrix with
  a hackathon prototype's weights (self-host 1, ops maturity 1, polish high) flips
  to LangSmith. State the weights so a reviewer can see *why* this system lands
  where it does.

## Verification

```bash
# 1. Stand up both deciding platforms locally (OTel-native, self-host, zero cost).
docker run -p 6006:6006 -p 4318:4318 arizephoenix/phoenix:latest
docker compose -f langfuse/docker-compose.yml up -d

# 2. Ingest the exercise-01/02 traces into EACH and confirm hierarchy fidelity:
#    invoke_agent root → worker invoke_agent → nested chat/execute_tool, with
#    retry.attempt and ERROR spans rendered in-context (not flattened).
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318 python agent.py        # Phoenix
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:3000/api/public/otel \
  OTEL_EXPORTER_OTLP_HEADERS="Authorization=Basic <pk:sk>" python agent.py # Langfuse

# 3. Run one LLM-judge faithfulness eval on captured traces in each platform and
#    confirm the score joins by trace_id (exercise-02's evaluate_run shape).

# 4. Exercise the deploy comparison: ingest the v2026.05 / v2026.06 batches from
#    drift.py and group faithfulness by service.version — confirm the regression
#    step-change is visible in each platform's UI.

# 5. Curate one dataset of failing traces into a regression set in each platform.

# 6. Re-compute the weighted totals from your own POC scores and confirm the
#    recommendation follows the numbers, not the demo.
python - <<'PY'
weights = {"otel":5,"selfhost":5,"eval":4,"cost":4,"data":3,"drift":3,"fw":2,"ops":3}
scores = {
  "Langfuse": [5,5,4,5,4,3,5,4],
  "Phoenix":  [5,5,5,5,4,5,5,3],
  "LangSmith":[4,2,5,3,5,3,4,5],
}
w = list(weights.values())
for name, s in scores.items():
    print(name, sum(wi*si for wi, si in zip(w, s)))
PY
# → Langfuse 129, Phoenix 136, LangSmith 109
```
