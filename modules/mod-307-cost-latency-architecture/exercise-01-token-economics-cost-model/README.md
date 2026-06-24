# mod-307-cost-latency-architecture/exercise-01-token-economics-cost-model — Solution

This is the reference solution for the token-economics cost model. It prices the
competitive-intelligence fan-out agent end to end, rolls the per-task cost up to
per-tenant and run-rate, runs a two-axis sensitivity sweep, and applies the
value-threshold gate. Every dollar and token figure below is
**illustrative-pending-verification**: the prices come from the Chapter 1 teaching
table, not from a live pricing page. The method is durable; the numbers are not.

## Approach

The exercise asks you to price an *architecture*, not a prompt. The discipline is
to walk every model call one brief triggers, attach a defended token estimate to
each, sum them, and only then roll up. Four ideas from Chapter 1 drive the whole
solution:

- **Cost scales with model calls, not user requests.** One "give me a brief"
  request becomes one decomposition call, five worker loops of four turns each
  (20 worker calls), and one synthesis call — 22 model calls for one request.
- **History is re-sent every turn.** A worker loop does not pay its base context
  once; it re-pays a *growing* transcript on every turn. The model below grows
  the context by `growth_per_turn + output_per_turn` each turn, exactly as a real
  loop re-sends what it has accumulated.
- **Fan-out multiplies everything.** Five workers means five worker contexts plus
  the orchestrator's own two calls. This is where the Anthropic ~15× multiplier
  comes from, and it is the term that dominates the cost shape.
- **The mean tenant hides the p95 tenant.** A flat price that is healthy against
  the average account can run at negative margin against the heavy-tail tenant,
  so the roll-up models both.

The worked model uses the Chapter 1 worksheet verbatim so it re-runs when prices
or the fan-out shape change. The starter parameters (5 workers, 4 turns, all on
the big model) are kept as the canonical baseline so exercise-02 has a fixed
number to cut against.

## Reference solution

### 1. Per-call inventory

One brief triggers **22 model calls**. Token estimates are stated as assumptions
with a one-line basis each.

| Call | Model | Input tokens | Output tokens | Basis |
| --- | --- | --- | --- | --- |
| Orchestrator decompose | big | 4,000 | 500 | system 1.5k + tool schema 2k + company name + framing 0.5k; emits 5 sub-question assignments |
| Worker turn 1 | big | 8,000 | 600 | base context: system 1.5k + tools 2k + assignment 0.5k + first retrieved page 4k |
| Worker turn 2 | big | 10,100 | 600 | turn 1 ctx + 1.5k new retrieval + 600 prior output re-sent |
| Worker turn 3 | big | 12,200 | 600 | turn 2 ctx + 1.5k + 600 re-sent |
| Worker turn 4 | big | 14,300 | 600 | turn 3 ctx + 1.5k + 600 re-sent |
| (× 5 workers) | big | — | — | one worker per sub-question (financials, product, hiring, news, regulatory) |
| Orchestrator synthesize | big | 9,000 | 1,200 | 5 distilled findings (≈1.5k each) + framing 1.5k; emits the brief |

The worker-loop growth is the load-bearing assumption: `growth_per_turn = 1,500`
(new retrieved content) plus the re-sent prior output. By turn 4 a worker is
sending 14,300 input tokens — **1.8×** its turn-1 context — and you pay for all of
it. This single fact is why the worker term dominates.

### 2. Per-task cost and cost shape

```python
"""exercise-01 reference cost model — competitive-intelligence fan-out agent.

Illustrative $/million-token prices (Chapter 1 teaching table). VERIFY against
current provider pricing before treating any output as a real budget.
"""
from dataclasses import dataclass

PRICES = {  # $ per 1,000,000 tokens — illustrative, replace with current
    "big":   {"in": 3.00, "out": 15.00},
    "small": {"in": 0.80, "out": 4.00},
}


@dataclass
class Call:
    model: str
    input_tokens: int
    output_tokens: int

    def cost(self) -> float:
        p = PRICES[self.model]
        return (self.input_tokens / 1e6) * p["in"] + (self.output_tokens / 1e6) * p["out"]


@dataclass
class Worker:
    """A worker is its own agent loop: `turns` calls over a growing context."""

    model: str
    base_context: int
    growth_per_turn: int
    output_per_turn: int
    turns: int

    def cost(self) -> float:
        total, ctx = 0.0, self.base_context
        for _ in range(self.turns):
            total += Call(self.model, ctx, self.output_per_turn).cost()
            ctx += self.growth_per_turn + self.output_per_turn  # history re-sent next turn
        return total


def task_cost(decompose: Call, workers: list[Worker], synth: Call) -> float:
    return decompose.cost() + sum(w.cost() for w in workers) + synth.cost()


def threshold_holds(value_per_task: float, cost: float, margin: float) -> bool:
    return value_per_task > cost * margin


# ----- BASELINE: 5 workers, 4 turns, all on the big model -----
DECOMPOSE = Call("big", 4_000, 500)
WORKERS = [Worker("big", 8_000, 1_500, 600, turns=4) for _ in range(5)]
SYNTH = Call("big", 9_000, 1_200)

if __name__ == "__main__":
    decompose_c = DECOMPOSE.cost()
    workers_c = sum(w.cost() for w in WORKERS)
    synth_c = SYNTH.cost()
    task = decompose_c + workers_c + synth_c

    print(f"decompose : ${decompose_c:.4f}  ({decompose_c / task:5.1%})")
    print(f"workers   : ${workers_c:.4f}  ({workers_c / task:5.1%})")
    print(f"synthesize: ${synth_c:.4f}  ({synth_c / task:5.1%})")
    print(f"PER-TASK  : ${task:.4f}")
    print(f"daily(800): ${task * 800:,.2f}")
    print(f"annual    : ${task * 800 * 365:,.0f}")
```

Running this produces the baseline cost shape (illustrative-pending-verification):

```text
decompose : $0.0195  ( 2.1%)
workers   : $0.8490  (92.9%)
synthesize: $0.0450  ( 4.9%)
PER-TASK  : $0.9135
daily(800): $730.80
annual    : $266,742
```

The dominant term is unambiguous: **the worker loops are 92.9% of per-task cost.**
Decomposition (2.1%) and synthesis (4.9%) are rounding error by comparison. Any
optimization budget that does not start at the worker loop is mis-aimed — which is
exactly the handoff into exercise-02 (route the workers down a tier, cache their
shared preamble).

### 3. Roll-up to per-tenant and run-rate

```text
per-task cost          = $0.9135
daily run-rate (×800)  = $730.80
annualized (×365)      = $266,742

tenants                = 120
mean tasks/tenant/day  = 800 / 120 = 6.67
mean tenant / month    = $0.9135 × 6.67 × 30 = $182.70
p95 tenant / month     = $0.9135 × (6.67 × 8) × 30 = $1,461.60
```

**Negative-margin check.** Pick an illustrative flat price of **$300/tenant/month**
(a plausible mid-market SaaS seat). Against the mean tenant ($182.70 cost) the
gross margin is +$117.30 — healthy. Against the **p95 tenant ($1,461.60 cost)** the
margin is **−$1,161.60** — the heavy-tail account runs deeply negative and a
handful of them erase the margin on dozens of healthy accounts. A flat-priced
fan-out agent is a structural invitation for exactly this. The roll-up surfaces it
at design time, which is the point.

### 4. Sensitivity analysis

Per-task cost ($, illustrative) varying worker count and loop turns independently:

```text
workers \ turns      3        5        8
   5              0.6540   1.2045   2.2665
   8              1.0077   1.8885   3.5877
  12              1.4793   2.8005   5.3493
```

Read the gradients from the baseline cell (5 workers, ~4 turns ≈ $0.91):

- **Turns is the steeper axis.** Holding workers at 5, going 3 → 8 turns moves
  cost $0.65 → $2.27 — a **3.5×** swing — because each added turn re-sends a
  larger transcript (super-linear growth from history re-send).
- Workers is close behind but **linear**: holding turns at 5, going 5 → 12 workers
  moves cost $1.20 → $2.80, a 2.3× swing for a 2.4× worker increase.

**Loop turns is the dominant driver.** The architecture's first cost lever is a
hard turn cap (`T`), not trimming a worker — capping `T` bends the steepest axis.

### 5. Value threshold

```text
value_per_task   = $4.50   (≈ 9 min of an analyst at a $30/hr loaded rate; a brief
                            that would take ~9 min to assemble by hand)
safety_margin    = 5×      (absorbs retries, failed trajectories still paid for,
                            loop-cost variance, and eval/observability overhead)
cost_per_task    = $0.9135 (baseline)

threshold test:  value_per_task > cost_per_task × safety_margin
                 $4.50 > $0.9135 × 5 = $4.57   →  FALSE (by a whisker)
```

**Verdict: build a cheaper variant.** At the baseline architecture the brief is
worth $4.50 but the 5× safety margin demands $4.57 of headroom — the agent fails
the gate at the *margin*, not on raw cost. This is the most instructive outcome:
the task clearly *should* be an agent (open-ended research, real value), but the
naïve "everything on the big model" build does not clear the threshold once you
honestly price retries and variance. The correct move is **not** to abandon the
agent and **not** to ship it as-is — it is to apply the exercise-02 levers (route
workers to the small model, cache the shared preamble), which drop per-task cost
to ≈$0.24 and turn the inequality decisively true ($4.50 > $0.24 × 5 = $1.20).
The cost model is what tells you the agent is viable *only after* optimization —
exactly the bridge into the next exercise.

## Meeting the acceptance criteria

- **Every model call enumerated, with token estimates and a stated basis.** The
  per-call inventory table lists all 22 calls (decompose + 5 workers × 4 turns +
  synthesize) with a one-line basis for each token figure, including the re-sent
  history growth per worker turn.
- **Per-task cost computed and decomposed by phase, dominant term named.**
  $0.9135 total; decompose 2.1% / workers 92.9% / synthesize 4.9%; worker loops
  named as dominant.
- **Roll-ups for daily, annual, and mean vs. p95 tenant, with a negative-margin
  check.** $730.80/day, $266,742/year, $182.70 mean tenant vs. $1,461.60 p95
  tenant, checked against a $300/month flat price (p95 runs −$1,161.60).
- **Sensitivity table over worker count and loop turns, dominant driver named.**
  The 3×3 sweep is shown; loop turns identified as the steeper (super-linear)
  axis.
- **Value-threshold verdict with explicit `value_per_task`, justified
  `safety_margin`, and a build / cheaper-variant / do-not-build conclusion.**
  $4.50 value, 5× margin justified, verdict = build-a-cheaper-variant.
- **Every price and token figure labelled illustrative-pending-verification.**
  Stated in the intro, the code docstring, and at the cost-shape output.

## Common pitfalls

- **Pricing one user request as one model call.** The whole exercise dies here.
  One brief is 22 calls. If your per-task cost looks like a single Sonnet call,
  you forgot the fan-out and the loop turns.
- **Charging base context once instead of re-sending it each turn.** A 4-turn
  worker is not `base_context × 1`; it is the running sum of a growing transcript.
  Forgetting the re-send under-prices the worker term — the dominant term — by
  roughly half.
- **Rolling up on the mean tenant only.** The mean ($182.70) looks fine against a
  $300 price and hides the p95 tenant ($1,461.60) that runs deeply negative. The
  p95 tenant is the account that turns a "healthy unit economics" deck into a
  margin incident.
- **Treating the happy-path cost as the real cost in the threshold test.** The
  safety margin exists precisely because retries, abandoned trajectories you still
  paid for, and loop-cost variance are not in the $0.9135. Dropping the margin (or
  setting it to 1×) flips a failing gate to a passing one and ships an
  under-priced agent.
- **Reading the verdict as binary build / do-not-build.** The honest answer here
  is the third option — *build a cheaper variant* — which only the cost model plus
  the exercise-02 levers can justify.

## Verification

```bash
# Re-run the reference model; confirm the cost shape and roll-ups.
python3 cost-model.py
# Expected (illustrative): PER-TASK $0.9135, workers 92.9%, daily(800) $730.80.
```

Checklist for a complete submission:

- [ ] `cost-model.py` (or `.xlsx`/`.csv`) re-runs and reproduces the per-task
      figure, the phase decomposition, the sensitivity table, and the roll-ups.
- [ ] `MEMO.md` states per-task cost, daily/annual run-rate, the p95-tenant
      negative-margin finding against a named price, and the value-threshold
      verdict — readable by a finance partner in one page.
- [ ] `NOTES.md` answers the three reflection prompts (dominant parameter, p95
      effect on verdict, any worker not worth its cost).
- [ ] Every price and token figure carries the illustrative-pending-verification
      label.
- [ ] Changing a price in `PRICES` or a parameter (turns, workers) re-flows every
      downstream number — the worksheet is live, not a transcribed snapshot.
