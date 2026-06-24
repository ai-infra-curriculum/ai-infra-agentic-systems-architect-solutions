# mod-307-cost-latency-architecture/exercise-03-cost-latency-quality-budgets — Solution

This is the reference solution for the module capstone: turning the cost model
(exercise-01) and levers (exercise-02) into a **governed reference architecture**.
It sets a cost–latency–quality budget per surface, builds the combined SLA table on
p95/p99, wires the control plane (meter, budget guard, kill switch) with concrete
caps, and writes three ADRs defending the material trades. All dollar, latency, and
quality figures are **illustrative-pending-verification** (Chapter 3/4 teaching
figures applied to the exercise-01 baseline).

## Approach

Chapter 3's thesis is that a cost model in a spreadsheet is a *forecast*; a cost
model wired into the architecture as enforced budgets, SLAs, and kill switches is a
*control plane*. Chapter 4's thesis is that cost, latency, and quality form a
triangle you cannot maximise at once, so each surface must **fix two corners and
name the third as the free variable** — a budget that does not say which dimension
floats is three wishes, not a budget.

The solution governs the three surfaces of the competitive-intelligence agent —
**interactive brief**, **watchlist refresh**, **deep-dive analysis** — each at a
different point in the triangle, using the exercise-02 levers to hit each budget,
and records where a budget could *not* be met by the cheapest viable design. The
control plane is the same meter/guard/kill-switch the budget-guard component
enforces, with hard caps (`N` workers, `T` turns) chosen from the exercise-01
sensitivity finding that loop turns is the steepest cost axis.

## Reference solution

### 1. Budgets per surface

Each surface fixes two corners and floats the third, justified by value and user
tolerance (not by what the system happens to do).

| Surface | Fixed corner A | Fixed corner B | Floating corner | Justification |
| --- | --- | --- | --- | --- |
| **Interactive brief** | cost ≤ $0.30/task | latency p95 < 8s | **quality** | High volume, user is waiting; pay for speed and cap spend, take the best quality inside the box. Routed+cached design lands at ≈$0.24 (exercise-02). |
| **Watchlist refresh** | cost ≤ $0.12/task | quality ≥ worker eval floor | **latency** | Nobody waits overnight; spend latency freely (24h window) to minimise cost on a bulk lane. Batch+route+cache lands at ≈$0.1185. |
| **Deep-dive analysis** | quality maximal | latency relaxed (minutes OK) | **cost** | Low volume, high stakes; the value of a correct deep report dwarfs the token cost, so cost floats and quality is maximised on the big-model lane. |

The interactive budget ($0.30) sits comfortably above the exercise-02 routed+cached
figure ($0.2370) — the cheapest viable design fits the box with headroom. The
watchlist budget ($0.12) is set right at the batched figure ($0.1185) — it fits,
but only just, which is the subject of ADR-002 and ADR-003.

### 2. Resolve the trade and pick the levers

| Surface | Design point | Levers (exercise-02) | Result vs. budget |
| --- | --- | --- | --- |
| Interactive brief | small workers + cached prefix, capped fan-out (≤5), capped loops (≤4) | route + cache | $0.2370 ≤ $0.30 — **fits** |
| Watchlist refresh | small workers + cached prefix + batch lane | route + cache + batch | $0.1185 ≤ $0.12 — **fits, barely** |
| Deep-dive analysis | big model, deep loops (T up to 8), refinement pass | none (deliberately) | cost floats; quality maximised |

**Where a budget could not be met by the cheapest viable design.** If the watchlist
quality floor were raised so the worker had to run on the big model, the per-brief
cost would jump to ≈$0.46 batched — **over the $0.12 ceiling by ~4×.** The honest
move (Chapter 4) is to relax the floating corner's neighbour: the watchlist's
quality floor is *not* firm at the deep-dive level — a "good enough" nightly refresh
is acceptable, so the worker stays on the small model and the budget holds. That
resolution is recorded in ADR-003.

### 3. SLA table

Combined cost + latency SLA, targeted on **p95/p99** (not the mean — a $0.04 mean
can hide a $0.40 p95 from tasks that fan out wide and loop deep), with the enforcer
named per row.

| SLO | Target (illustrative) | Enforced by |
| --- | --- | --- |
| Interactive latency | p95 < 8s, p99 < 15s | capped fan-out (≤5), sync lane, streaming |
| Watchlist latency | results within 24h, no interactive SLA | batch lane |
| Deep-dive latency | p95 < 5 min (analyst-tolerant) | big-model sync lane, no batch |
| Cost per task — interactive | < $0.24 mean, < $0.30 p95 | router + cache + per-task cap |
| Cost per task — watchlist | < $0.12 p95 | router + cache + batch lane |
| Cost per tenant / day | < $40.00 | tenant meter + per-tenant kill switch |
| Cache hit rate | > 80% of worker-prefix tokens read from cache | cache-first prompt layout + TTL tuning |

The per-tenant ceiling ($40/day) is set above the exercise-01 p95 tenant
(≈$48.72/day at the *baseline* cost, but ≈$12.65/day at the routed+cached cost) so
the optimised heavy-tail account stays inside the ceiling while the un-optimised one
would trip it — the kill switch is the backstop, the levers are the primary control.

### 4. Wire the control plane

```text
                         ┌──────────────────────────────────────────┐
                         │              CONTROL PLANE               │
                         │  budget guard · token meter · kill switch│
                         └───────────────┬──────────────────────────┘
                                         │ enforces caps, reads meter
   request ─▶ ┌──────────┐   ┌───────────▼───────────┐   ┌──────────────┐
              │  router  │──▶│      orchestrator      │──▶│  synthesize  │─▶ result
              │ (big/sm) │   │ bounded fan-out (N≤5)  │   │ (big model)  │
              └──────────┘   └───────────┬───────────┘   └──────────────┘
                                  fan-out │  (each capped at T≤4 turns)
                          ┌───────────────┼───────────────┐
                          ▼               ▼               ▼
                    ┌───────────┐   ┌───────────┐   ┌───────────┐
                    │ worker    │   │ worker    │   │ worker    │
                    │ small +   │   │ small +   │   │ small +   │  ← shared
                    │ cached    │   │ cached    │   │ cached    │    cached prefix
                    └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
                          └──────── meter every call ─────┘
                                         │
                         ┌───────────────▼──────────────┐
                         │  ASYNC / BATCH LANE (50% off) │  ← watchlist refresh,
                         │  nightly refresh · eval runs  │     eval runs
                         └──────────────────────────────┘
```

**Three enforcement mechanisms, with concrete values:**

1. **Hard caps (prevent).** Bounded fan-out `N = 5` workers/task (one per
   sub-question); capped loops `T = 4` turns/worker for interactive and watchlist,
   `T = 8` for deep-dive; max output tokens 2,000/call. The turn cap is the primary
   cost control because exercise-01 showed loop turns is the steepest cost axis.
2. **Graceful degradation (adapt).** At `degrade_at = 0.75` of the per-task budget,
   the orchestrator skips the optional synthesis refinement pass and drops any
   remaining worker turn, returning the best partial brief flagged
   `truncated for budget`. Useful-under-pressure beats brittle.
3. **Kill switches (stop).** Per-tenant: halt the tenant when daily spend hits the
   $40/day ceiling (contains blast radius to one account — a misconfigured client
   hammering the API). Global: halt all spend if system run-rate exceeds 1.5× the
   forecast daily ($730.80 baseline → trip at ≈$1,096/day) — the last line against a
   price-change-inverts-margin incident. Both read the same meter.

**Budget guard adapted to these numbers:**

```python
"""Budget guard for the competitive-intelligence agent. Per-task and per-tenant
ceilings enforced against a running token meter. Trips degradation, then a kill
switch. Ceilings are illustrative-pending-verification."""
from dataclasses import dataclass, field


@dataclass
class Budget:
    task_usd: float           # per-task ceiling
    tenant_daily_usd: float   # per-tenant/day ceiling
    degrade_at: float = 0.75  # fraction of task budget that triggers downshift


# Per-surface budgets (Task 1).
INTERACTIVE = Budget(task_usd=0.30, tenant_daily_usd=40.00)
WATCHLIST = Budget(task_usd=0.12, tenant_daily_usd=40.00)
DEEP_DIVE = Budget(task_usd=5.00, tenant_daily_usd=40.00)  # cost floats; ceiling is a guardrail


@dataclass
class Meter:
    spend_by_tenant: dict[str, float] = field(default_factory=dict)

    def record(self, tenant: str, call_cost: float) -> None:
        self.spend_by_tenant[tenant] = self.spend_by_tenant.get(tenant, 0.0) + call_cost


class BudgetExceeded(Exception):
    """Raised to trip the kill switch — caller halts the task/tenant."""


def guard(meter: Meter, b: Budget, tenant: str, task_spend: float) -> str:
    """Return 'proceed', 'degrade', or raise BudgetExceeded to kill."""
    if meter.spend_by_tenant.get(tenant, 0.0) >= b.tenant_daily_usd:
        raise BudgetExceeded(f"tenant {tenant} hit daily ceiling")
    if task_spend >= b.task_usd:
        raise BudgetExceeded(f"task for {tenant} hit per-task ceiling")
    if task_spend >= b.task_usd * b.degrade_at:
        return "degrade"   # skip refinement pass / drop remaining worker turn
    return "proceed"
```

### 5. The ADRs

Three ADRs cover the material trades: the interactive routing decision, the
watchlist batch decision, and the budget that could not be met as stated.

```text
# ADR-001: Route interactive-brief workers to the small model

## Status
Accepted

## Context
Exercise-01 showed worker loops are 92.9% of the $0.9135 baseline per-task cost.
The interactive surface fixes cost (≤$0.30) and latency (p95<8s) and floats
quality. The all-big-model design is $0.9135 — 3× over the cost box.

## Decision
Interactive position: cost ≤ $0.30 and latency p95 < 8s FIXED; quality FLOATS.
Route the 5 extraction workers to the small model; keep decompose/synthesize on
the big model. Lands at $0.2370/task (exercise-02), inside the box with headroom.

## Levers & preconditions
Routing (precondition: small model clears the worker eval gate) + caching
(precondition: 4k prefix reused 20× before TTL — it is, within one brief).

## Evidence
- Cost-model output: routed+cached = $0.2370 ≤ $0.30 ceiling (exercise-01/02).
- Eval: the worker distillation eval set clears its faithfulness+coverage bar on
  the small model on a real test set (mod-304) before the downshift shipped.

## Accepted trade-off
p95 quality dips on the long tail of multi-hop sub-questions in exchange for a
74% cost cut. Deep-dive questions route to the big-model lane instead.

## Consequences
Hard caps N≤5, T≤4; degrade-at-0.75 skips the refinement pass; the worker eval
gate must be re-run before any prompt change that could move the quality floor.
```

```text
# ADR-002: Run the watchlist refresh on the async batch lane

## Status
Accepted

## Context
The nightly watchlist re-runs briefs for every watched company — a bulk,
latency-tolerant workload. Running it on the synchronous interactive lane would
pay full price for work nobody waits on.

## Decision
Watchlist position: cost ≤ $0.12 and quality ≥ worker eval floor FIXED; latency
FLOATS (24h window). Route + cache + batch (50% off) → $0.1185/brief.

## Levers & preconditions
Routing + caching (as ADR-001) + batching (precondition: latency tolerance — the
24h window holds because no analyst waits overnight).

## Evidence
- Cost-model output: routed+cached+batched = $0.1185 ≤ $0.12 ceiling.
- Eval: same worker eval gate as ADR-001; the small model clears the
  "good-enough" nightly floor.

## Accepted trade-off
Results are not available until the morning batch completes; a same-day watchlist
change is not reflected until the next nightly run. Acceptable for a watchlist.

## Consequences
Batch lane SLO "within 24h"; eval runs share the lane; if the batch window is
missed, the runbook re-queues, it does not fall back to the sync lane (that would
bust the cost ceiling).
```

```text
# ADR-003: Watchlist quality floor held at "good enough", not deep-dive parity

## Status
Accepted

## Context
A request surfaced to raise the watchlist quality floor to deep-dive parity (big
model). That forces the worker onto the big model, pushing the batched per-brief
cost to ≈$0.46 — ~4× over the $0.12 ceiling. The budget cannot be met as stated.

## Decision
Hold the watchlist quality corner at the small-model "good-enough" floor; do NOT
raise it to deep-dive parity. The budget owner relaxed the quality neighbour
rather than the cost ceiling (Chapter 4's honest move).

## Levers & preconditions
None added — the resolution is a budget decision, not a lever. The escape hatch
is the existing deep-dive surface for any company that genuinely needs parity.

## Evidence
- Cost-model output: big-model batched watchlist = ≈$0.46 > $0.12 — does not fit.
- The deep-dive surface already exists for high-stakes, quality-maximal requests.

## Accepted trade-off
The nightly refresh will occasionally miss nuance a big-model run would catch;
analysts escalate those companies to the deep-dive surface on demand. We accept
a quality ceiling on the bulk lane to keep its cost ceiling firm.

## Consequences
Watchlist eval bar is the "good-enough" floor, not the deep-dive bar; a
documented escalation path to deep-dive; the $0.12 ceiling stays load-bearing.
```

## Meeting the acceptance criteria

- **Each surface fixes two corners and names the floating one, with numeric
  ceilings justified by value/tolerance.** Interactive (cost+latency fixed, quality
  floats), watchlist (cost+quality fixed, latency floats), deep-dive (quality+
  latency fixed, cost floats) — all with numbers.
- **SLA table targets cost and latency on p95/p99, with a cache-hit and per-tenant
  SLO and a named enforcer per row.** The table targets p95/p99 latency and cost,
  includes cache hit rate (>80%) and per-tenant/day (<$40), each with an enforcer.
- **Reference architecture shows a control plane (meter, guard, kill switch) and
  structural caps with concrete N and T.** Diagram includes the control plane;
  N=5, T≤4 (interactive/watchlist) / T≤8 (deep-dive).
- **Three enforcement mechanisms specified: hard caps, graceful degradation, and
  per-tenant + global kill-switch triggers.** Caps (N/T/max-output), degrade at
  0.75 (skip refinement, drop turn), kill switches (per-tenant $40/day, global
  1.5× forecast).
- **Three ADRs in the template format, each naming the accepted trade plainly and
  citing cost-model and eval evidence.** ADR-001/002/003 each cite an exercise-01/02
  cost figure and an eval result and state the trade.
- **At least one ADR documents a budget that could not be met and the honest
  resolution.** ADR-003 documents the un-meetable watchlist-at-parity budget and
  the relax-the-quality-neighbour resolution.

## Common pitfalls

- **Writing a budget that does not name the floating corner.** "Fast, cheap, and
  high quality" is three wishes; under load the system silently picks one for you,
  usually badly. Every surface must say which dimension floats.
- **Forcing one global trade-off across all three surfaces.** Putting the deep-dive
  on the same small-model lane as the interactive brief under-delivers on the
  high-stakes report; putting the watchlist on the big-model sync lane overpays for
  work nobody waits on. Let each surface sit at its own point.
- **Targeting the mean instead of p95/p99.** A healthy mean cost-per-task hides the
  fan-out-wide, loop-deep tail that generates the surprise bill and the SLA breach.
  Budget and alert on the tail.
- **A budget guard with no meter behind it.** A ceiling the system cannot measure
  in real time is a wish. The guard, the kill switch, and every SLO must read the
  same live token meter (wired to the mod-305 observability stack) or none of it is
  enforceable.
- **Quietly busting a budget instead of recording the trade.** An over-budget
  design that ships without an ADR becomes a production cost incident with no paper
  trail — the exact failure this module exists to prevent. ADR-003 is the model:
  document the un-meetable budget and the honest resolution.

## Verification

```bash
# Exercise the budget guard against the per-surface ceilings.
python3 budget-guard.py
# Expected: 'proceed' below 0.75× budget; 'degrade' at/above it; BudgetExceeded at
#           the per-task or tenant-daily ceiling.
```

Checklist for a complete submission:

- [ ] `budgets.md` gives each surface two fixed corners + the floating one, with
      value/tolerance justification and numeric ceilings.
- [ ] `sla.md` targets cost and latency on p95/p99, includes a cache-hit and a
      per-tenant cost SLO, and names the enforcer of each row.
- [ ] `governed-architecture.{txt,png}` shows the control plane (meter, guard, kill
      switch) and structural caps with concrete N and T.
- [ ] `budget-guard.py` is adapted to the per-surface ceilings and the three
      enforcement mechanisms (caps, degradation, kill switches) are specified.
- [ ] `adr/ADR-001…003.md` follow the template, name the accepted trade plainly,
      and cite cost-model and eval evidence; at least one documents an un-meetable
      budget.
- [ ] Every cost, latency, and quality figure is labelled
      illustrative-pending-verification.
- [ ] `NOTES.md` answers the three reflection prompts (hardest surface to fit, where
      cost and latency SLOs conflict, whether the interactive ADR survives an
      incident review).
