# mod-305-observability-tracing/exercise-02-quality-and-drift-signals — Solution

## Approach

This exercise builds the **quality plane** that APM cannot produce, on top of the
instrumented agent from exercise-01. The work is in four moves, in order:

1. **Design the signal taxonomy as an artifact first** (`SIGNALS.md`). The
   taxonomy is the architecture — it decides what each signal measures, how it's
   computed, where it attaches, and whether it runs sync or async. Code follows.
2. **Score faithfulness asynchronously**, against the run's output *and* its
   retrieved context, and **join the score back to the trace by `trace_id`** as a
   backend score. Async-on-a-sample is the production-correct shape because inline
   scoring doubles latency and cost.
3. **Detect quality drift across two deploys** by comparing the *distribution* of
   faithfulness grouped by `service.version`, not by eyeballing one trace —
   because every run differs, a regression is only legible as a step-change in a
   distribution.
4. **Write the drift alert rule** with a justified threshold that is neither
   jumpy nor numb.

The load-bearing decisions: a single judge is fallible, so we never trust it as
ground truth — we trust it as a *signal* that discriminates faithful from
unfaithful in aggregate; and the join key is always `trace_id`, never an
in-band mutation of the agent loop, so evaluation stays decoupled from execution.

## Reference solution

### The signal taxonomy artifact (`SIGNALS.md`)

```text
SIGNAL TAXONOMY — research/RAG agent
Every signal here is structurally absent from APM's golden signals
(latency / traffic / errors / saturation), because each needs the OUTPUT,
the CONTEXT, or a JUDGMENT that a metric pipeline does not carry.

────────────────────────────────────────────────────────────────────────
1. FAITHFULNESS (groundedness)
   measures   : are the output's claims supported by the retrieved context?
   computed   : LLM-judge over (output AND retrieved context) → 0..1
                (fraction of claims the evidence supports)
   attaches   : separate score joined by trace_id  (name="faithfulness")
   sync/async : ASYNC, post-hoc on a sample (and 100% of errors)
   why not APM: APM has no retrieved context and makes no judgment call;
                a fluent hallucination is a fast 200 it scores as success.
────────────────────────────────────────────────────────────────────────
2. OUTPUT QUALITY
   measures   : did the answer satisfy the request (complete/relevant/cited)?
   computed   : LLM-judge rubric over the output (+ the question) → 0..1
   attaches   : trace-level attribute (rolls up the whole trajectory)
   sync/async : ASYNC, sampled
   why not APM: quality is graded and subjective; APM has no slot for a 0.7.
────────────────────────────────────────────────────────────────────────
3. SAFETY (PII / toxicity / jailbreak)
   measures   : did the run leak PII or emit disallowed content?
   computed   : deterministic regex/classifier guardrail over the output
                → boolean outcome (+ category)
   attaches   : trace-level attribute  (safety.pii_leak = true|false)
   sync/async : SYNC is acceptable (cheap, deterministic) but recorded async
                here for uniformity; a leak blocks/flags regardless.
   why not APM: a leak is a 200; APM has no vocabulary for "right but unsafe".
────────────────────────────────────────────────────────────────────────
4. TASK SUCCESS
   measures   : over the whole trajectory, did it accomplish the user's goal?
   computed   : LLM-judge or deterministic check over the FULL span tree
                (e.g. did the booking land on the requested date?) → 0/1
   attaches   : trace-level attribute  (task.success)
   sync/async : ASYNC; trajectory-scoped, needs the completed trace.
   why not APM: APM aggregates per-request and discards the sequence; success
                emerges only from the ordered spans.
────────────────────────────────────────────────────────────────────────

JOIN MODEL
  All four join to the trace by trace_id as backend scores/attributes.
  None of them mutate the agent loop. Evaluation is decoupled from execution.

SAMPLING (which traces get the expensive judge)
  100% of errors + guardrail trips, 100% of low-quality/high-cost runs,
  a small fixed fraction of the rest  → tail-biased, not uniform.
```

### Faithfulness scoring, joined by `trace_id`

`faithfulness.py` — an LLM-judge with a faithfulness rubric, run **after** the
trace completes and joined back by `trace_id`. The judge call is abstracted so
the same wiring works against the mod-304 rubric harness or a direct provider
call; a `RubricJudge` shim makes the solution runnable offline.

```python
"""Async faithfulness scoring joined to traces by trace_id."""
from __future__ import annotations

import asyncio
import re
from dataclasses import dataclass, field


@dataclass
class Verdict:
    score: float            # 0..1 fraction of claims supported
    supported: list[str] = field(default_factory=list)
    unsupported: list[str] = field(default_factory=list)


class RubricJudge:
    """Offline stand-in for an LLM-judge: a claim is 'supported' if its key
    tokens appear in the evidence. Swap for the mod-304 harness in prod."""

    async def __call__(self, *, answer: str, evidence: str) -> Verdict:
        await asyncio.sleep(0)
        claims = [c.strip() for c in re.split(r"[.\n]", answer) if c.strip()]
        ev = evidence.lower()
        supported, unsupported = [], []
        for claim in claims:
            tokens = [t for t in re.findall(r"[a-z0-9]+", claim.lower()) if len(t) > 3]
            hit = sum(1 for t in tokens if t in ev)
            (supported if tokens and hit / len(tokens) >= 0.5 else unsupported).append(
                claim
            )
        total = len(supported) + len(unsupported)
        return Verdict(
            score=(len(supported) / total) if total else 1.0,
            supported=supported,
            unsupported=unsupported,
        )


async def score_faithfulness(
    judge, trace_id: str, output: str, context: list[str]
) -> float:
    """LLM-judge: fraction of output claims supported by retrieved context. 0..1."""
    verdict = await judge(answer=output, evidence="\n".join(context))
    return verdict.score


class InMemoryBackend:
    """Stands in for the observability backend's score/feedback API."""

    def __init__(self) -> None:
        self.scores: dict[tuple[str, str], float] = {}

    async def create_score(self, *, trace_id: str, name: str, value: float) -> None:
        await asyncio.sleep(0)
        self.scores[(trace_id, name)] = value


async def attach_score(backend, trace_id: str, name: str, value: float) -> None:
    """Join an async eval result back to its trace by trace_id."""
    await backend.create_score(trace_id=trace_id, name=name, value=value)


async def evaluate_run(judge, backend, run: dict) -> float:
    """Post-hoc: read output+context off a completed run, score, join by id."""
    score = await score_faithfulness(
        judge, run["trace_id"], run["output"], run["context"]
    )
    await attach_score(backend, run["trace_id"], "faithfulness", score)
    return score
```

Demonstrating that the signal **discriminates**:

```python
async def demo_discrimination() -> None:
    judge, backend = RubricJudge(), InMemoryBackend()
    context = [
        "Q2 revenue was 4.2 million dollars, up 12 percent.",
        "Operating margin improved to 18 percent.",
    ]
    faithful = {
        "trace_id": "T-faithful",
        "context": context,
        "output": "Revenue reached 4.2 million dollars. Margin improved to 18 percent.",
    }
    unfaithful = {
        "trace_id": "T-hallucinated",
        "context": context,
        # injected claim absent from the context:
        "output": "Revenue reached 4.2 million dollars. The company acquired a rival.",
    }
    high = await evaluate_run(judge, backend, faithful)
    low = await evaluate_run(judge, backend, unfaithful)
    assert high > low, (high, low)
    print(f"faithful={high:.2f}  unfaithful={low:.2f}")
    print("backend scores:", backend.scores)


if __name__ == "__main__":
    asyncio.run(demo_discrimination())
```

The faithful run scores `1.00`; the injected-hallucination run scores `0.67`
(two of three claims supported, the acquisition claim unsupported). The scores
land in the backend keyed by `trace_id`, exactly as an async eval result joins
back to its trace.

### Quality drift across deploys

`drift.py` — run a batch of N≥20 inputs under `service.version = A`, change *one
thing* that should degrade quality, tag it `B`, and compare distributions
grouped by version.

```python
"""Deploy-segmented drift comparison: faithfulness distribution by version."""
from __future__ import annotations

import asyncio
import statistics

from faithfulness import InMemoryBackend, RubricJudge, evaluate_run

CONTEXT = [
    "The library opens at 9am on weekdays.",
    "Returns are due within 21 days.",
    "Late fees are 25 cents per day.",
]
QUESTIONS = [f"q{i}: when is X due / open / charged?" for i in range(20)]


def answer_good(_q: str) -> str:
    """Deploy A: grounded answers drawn from the context."""
    return "It opens at 9am. Returns are due within 21 days. Fees are 25 cents."


def answer_degraded(_q: str) -> str:
    """Deploy B: a 'worse prompt' that injects an ungrounded claim."""
    base = "It opens at 9am. Returns are due within 21 days."
    return base + " Members get free overnight delivery."  # not in context


async def run_batch(judge, backend, version: str, answer_fn) -> list[float]:
    scores = []
    for i, q in enumerate(QUESTIONS):
        run = {
            "trace_id": f"{version}-{i}",
            "context": CONTEXT,
            "output": answer_fn(q),
            "service.version": version,
        }
        scores.append(await evaluate_run(judge, backend, run))
    return scores


def compare_by_version(scores: dict[str, list[float]]) -> None:
    print("faithfulness distribution, grouped by deploy")
    for version, vals in scores.items():
        mean = statistics.mean(vals)
        bar = "█" * round(mean * 20)
        print(f"{version}: mean={mean:.3f}  n={len(vals)}  {bar}")


async def main() -> None:
    judge, backend = RubricJudge(), InMemoryBackend()
    a = await run_batch(judge, backend, "v2026.05", answer_good)
    b = await run_batch(judge, backend, "v2026.06", answer_degraded)
    compare_by_version({"v2026.05": a, "v2026.06": b})
    delta = statistics.mean(a) - statistics.mean(b)
    print(f"regression delta = {delta:.3f}")


if __name__ == "__main__":
    asyncio.run(main())
```

Output (the seeded regression shows as an attributable step-change):

```text
faithfulness distribution, grouped by deploy
v2026.05: mean=1.000  n=20  ████████████████████
v2026.06: mean=0.667  n=20  █████████████
regression delta = 0.333
```

Because the only thing that changed between batches is the deploy-tagged answer
function (the "worse prompt"), the drop is attributable to `v2026.06`, not
run-to-run noise — every A run scores identically, so the step is the deploy.

### The drift alert rule

```text
RULE
  Alert when the 7-day rolling mean faithfulness for the CURRENT
  service.version drops more than 0.05 below the prior version's 7-day mean,
  AND the current window has n >= 50 scored runs.

  fire if:  mean_faithfulness(current_version, 7d) <
            mean_faithfulness(prior_version, 7d) - 0.05
            and n_current >= 50

THRESHOLD JUSTIFICATION
  0.05 is ~1 in 20 runs flipping faithful→unfaithful — a real, user-visible
  shift, well outside the run-to-run jitter we observe in a stable deploy
  (typically < 0.02 on a 7-day window). Smaller than 0.05 and the alert is
  JUMPY (fires on noise; a single bad prompt edit reverts it). Larger and
  it is NUMB (a steady 0.04/week slide never trips it until quality has
  cratered). The n >= 50 floor keeps a quiet weekend from firing on 3 runs.
```

## Meeting the acceptance criteria

- **`SIGNALS.md` defines ≥4 non-APM signals** — faithfulness, output quality,
  safety, task success — each with measure / computation / attachment /
  sync-vs-async, and each states why APM's golden signals can't produce it.
- **Faithfulness is scored asynchronously** against the run's output *and
  retrieved context* (`score_faithfulness` reads both off the completed run), and
  the score is **joined back by `trace_id`** via `attach_score` →
  `backend.create_score(trace_id=...)` — never inline in the agent loop.
- **The signal discriminates** — `demo_discrimination` shows a faithful run at
  `1.00` and an injected-hallucination run at `0.67`.
- **Distribution by `service.version` shows the seeded regression** as an
  attributable step-change (`v2026.05` 1.000 → `v2026.06` 0.667), not noise,
  because the only variable is the deploy tag.
- **A drift alert rule is written** with a justified 0.05 threshold and an
  `n >= 50` floor, argued as neither jumpy nor numb.

## Common pitfalls

- **Scoring inline in the agent loop.** Calling the judge synchronously inside
  the run doubles latency and adds an LLM call's cost to every request. Score
  async, on a sample, joined by `trace_id` — the agent loop must not wait on the
  quality plane.
- **Judging the output without the retrieved context.** Faithfulness is
  *groundedness*; scoring the answer alone reduces it to a fluency check that a
  confident hallucination passes. The judge must see the output *and* the
  evidence the trace captured.
- **Trusting a single judge as ground truth.** One judge misses injected claims
  (the reflection question is exactly this). Treat the score as a discriminating
  signal in aggregate; for high-stakes gating, ensemble judges or add a
  deterministic check — never promote a single 0/1 to "truth".
- **Comparing two individual traces instead of distributions.** Because every run
  differs, eyeballing one A trace against one B trace shows noise, not the
  regression. Group a quality signal by `service.version` and compare means over
  a window.
- **Un-versioned runs.** Without `service.version` (and `prompt.version`/model)
  on every run, a regression has no cause to attribute — you see the drop and can
  blame nothing. The deploy tag is what makes drift legible.

## Verification

```bash
# 1. Deps: only the stdlib is needed for the offline judge; install the OTel
#    SDK + mod-304 rubric harness when wiring the real judge.
python --version   # 3.10+

# 2. Show the signal discriminates faithful from hallucinated, scores joined.
python faithfulness.py
#   → faithful=1.00  unfaithful=0.67
#   → backend scores keyed by trace_id

# 3. Show the deploy-segmented drift regression as a step-change.
python drift.py
#   → v2026.05 mean ~1.000, v2026.06 mean ~0.667, regression delta ~0.33

# 4. Confirm the join is by trace_id (no agent-loop mutation): every score is
#    addressed by (trace_id, signal_name).
python -c "import asyncio, faithfulness as f; \
b=f.InMemoryBackend(); j=f.RubricJudge(); \
asyncio.run(f.evaluate_run(j,b,{'trace_id':'T1','output':'opens at 9am','context':['it opens at 9am']})); \
print(b.scores)"   # → {('T1','faithfulness'): 1.0}

# 5. (Stretch) tail-biased sampling: route 100% of errors/low-quality runs and a
#    small fraction of the rest to the judge; confirm no failed run is skipped.
```
