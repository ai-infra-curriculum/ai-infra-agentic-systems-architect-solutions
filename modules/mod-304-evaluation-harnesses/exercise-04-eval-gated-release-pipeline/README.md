# mod-304-evaluation-harnesses/exercise-04-eval-gated-release-pipeline — Solution

## Approach

The exercise composes the previous three into a **deployment gate** that makes the
"should we ship?" decision mechanical and reproducible — relative gates, hard gates,
shadow, canary with a window, and automated rollback that feeds the label flywheel.
The discipline from Chapter 4 is that promotion criteria are *code/config*, not a
judgment call in a meeting: the same inputs always produce the same decision.

Design choices that drive the reference implementation:

- **One versioned dataset, candidate vs. baseline, k runs per case.** The offline gate
  never uses an absolute threshold; it diffs the candidate against the current
  production baseline on the same dataset version, and runs each case `k` times so a
  flaky case cannot randomly red/green the build.
- **Two gate types with different semantics.** A **relative gate** blocks if any axis
  regresses more than ε versus baseline. A **hard gate** blocks regardless of baseline
  on a safety violation, an unconfirmed destructive call, or a schema-invalid rate
  above a tiny floor. A hard gate never trades off against an accuracy gain — a build
  that's better on accuracy but adds one destructive-tool violation does not ship.
- **Shadow and canary are reference-free.** They score replayed inputs with no labels,
  using the same harness: judge-sampled axes, trajectory predicates, schema-validity
  rate, and cost/latency. Canary compares over a **window** of N runs, not a single
  point, and arms automated rollback throughout.
- **The flywheel is wired, not described.** A rolled-back failing case is appended to
  the offline dataset (the in-memory `Dataset` here; a JSON file in production), so the
  same bug becomes a permanent regression test and cannot ship twice.

The two scorers used by the gate are stubbed deterministically so the pipeline is fully
runnable: a `baseline` build that behaves well, a **regressing** candidate that skips
retrieval and emits an unconfirmed refund (trips both a relative regression and the
hard gate), and a **clean** candidate that promotes through every stage. The stubs
expose the exact metric vector the real exercise-01/02/03 scorers produce, so swapping
them in is a constructor change.

## Reference solution

### The deployment-gate diagram (`pipeline.md`)

```text
  change (prompt / tool / model)
        │
        ▼
  ┌──────────────────────────────────────────────────────────────┐
  │ OFFLINE GATE   (CI, dataset@version, k runs/case)            │
  │   scorers: trajectory+final (ex-01), tool-call (ex-02),      │
  │            calibrated judge (ex-03)                          │
  │   RELATIVE: no axis drops > eps vs baseline                  │
  │   HARD:     safety_violations==0, unconfirmed_destructive==0,│
  │             schema_invalid_rate <= floor                     │
  └───────────┬──────────────────────────────────────┬───────────┘
   relative   │ pass                          FAIL    │ block
   or hard ───┤                              ◀────────┘ (fix & re-run)
   regress?   ▼
  ┌────────────────────────┐
  │ SHADOW / ONLINE EVAL   │  replay held-out inputs through candidate,
  │  reference-free signals│  no user impact; block on hard gate or
  │  (judge sample, preds, │  metric regression beyond threshold
  │   schema rate, cost)   │
  └───────────┬────────────┘
              │ pass
              ▼
  ┌────────────────────────┐     hard gate / metric regression
  │ CANARY  (5% slice)     │────────  over the window  ────────▶ AUTO-ROLLBACK
  │  candidate vs baseline │                                         │
  │  over a WINDOW of N    │                                         │
  └───────────┬────────────┘                                         │
              │ stable over window                                   │
              ▼                                                      ▼
  ┌────────────────────────┐                         ┌──────────────────────────┐
  │ FULL ROLLOUT           │                         │ route failing case BACK   │
  │  candidate -> baseline │                         │ INTO offline dataset      │
  │  continuous online ────┼──── flagged failures ──▶│ (label flywheel)          │
  │  eval continues        │                         └──────────────────────────┘
  └────────────────────────┘

  PROMOTION CRITERIA (coded in gates.py, not prose):
    merge   -> offline : green relative gate AND zero hard-gate hits
    offline -> shadow  : offline pass AND shadow clean over min sample
    shadow  -> canary  : shadow clean
    canary  -> full    : window-stable within threshold of baseline, rollback armed
    any     -> rollback: one hard-gate hit OR metric regression past threshold
```

### Builds and scorer stubs (`builds.py`)

```python
"""Deterministic build stubs. Each returns the per-axis metric vector the real
ex-01/02/03 scorers produce. Swap these for live scorers via the same shape."""
from __future__ import annotations

from dataclasses import dataclass


@dataclass(frozen=True)
class Metrics:
    accuracy: float
    citation: float
    retrieved_before_answering: float   # trajectory predicate pass-rate
    schema_invalid_rate: float
    safety_violations: int              # unconfirmed destructive calls etc.
    cost_per_task: float

    def axes(self) -> dict[str, float]:
        # Axes the RELATIVE gate compares (higher is better).
        return {
            "accuracy": self.accuracy,
            "citation": self.citation,
            "retrieved_before_answering": self.retrieved_before_answering,
        }


# Production baseline: healthy on every axis.
BASELINE = Metrics(
    accuracy=0.91, citation=0.88, retrieved_before_answering=1.0,
    schema_invalid_rate=0.005, safety_violations=0, cost_per_task=0.012,
)

# Regressing candidate: a prompt change skipped retrieval and a tool change
# emits an unconfirmed refund -> relative regression AND a hard-gate hit.
REGRESSING = Metrics(
    accuracy=0.84, citation=0.72, retrieved_before_answering=0.40,
    schema_invalid_rate=0.02, safety_violations=3, cost_per_task=0.018,
)

# Clean candidate: a genuine accuracy improvement, no regressions, no violations.
CLEAN = Metrics(
    accuracy=0.93, citation=0.89, retrieved_before_answering=1.0,
    schema_invalid_rate=0.004, safety_violations=0, cost_per_task=0.011,
)


def run_suite(build: Metrics, dataset: list, *, k: int = 3) -> Metrics:
    """Stub: a real run executes `build` over `dataset` k times and aggregates.
    The stub is deterministic, so k does not change the result here — but the
    signature and k-runs contract match the production scorer."""
    _ = (dataset, k)
    return build
```

### Gates, shadow, canary, flywheel (`gates.py`)

```python
"""Coded promotion criteria: relative + hard offline gate, shadow check,
windowed canary, automated rollback, and the label flywheel."""
from __future__ import annotations

from dataclasses import dataclass, field

from builds import Metrics, run_suite

EPS = 0.02                 # relative-regression tolerance per axis
SCHEMA_INVALID_FLOOR = 0.01
CANARY_WINDOW = 20
CANARY_METRIC_THRESHOLD = 0.03  # max tolerated canary-vs-baseline accuracy drop


@dataclass
class Dataset:
    version: str
    cases: list = field(default_factory=list)

    def add_case(self, case: dict) -> None:
        self.cases.append(case)


@dataclass
class GateResult:
    passed: bool
    summary: dict
    blocking: list[str]


# --------------------------------------------------------------------------- #
# Offline gate: relative (vs baseline) + hard (absolute floors)
# --------------------------------------------------------------------------- #
def offline_gate(
    candidate: Metrics, baseline: Metrics, dataset: Dataset, *, k: int = 3,
    eps: float = EPS,
) -> GateResult:
    cand = run_suite(candidate, dataset.cases, k=k)
    base = run_suite(baseline, dataset.cases, k=k)
    blocking, summary = [], {}

    for axis, cand_val in cand.axes().items():
        base_val = base.axes()[axis]
        delta = cand_val - base_val
        summary[axis] = {"candidate": cand_val, "baseline": base_val,
                         "delta": round(delta, 3)}
        if delta < -eps:                                  # RELATIVE gate
            blocking.append(f"{axis} regressed {delta:+.3f} (> {eps} drop)")

    # HARD gates — absolute, independent of baseline.
    if cand.safety_violations > 0:
        blocking.append(
            f"safety: {cand.safety_violations} unconfirmed destructive call(s) "
            "(hard gate)"
        )
    if cand.schema_invalid_rate > SCHEMA_INVALID_FLOOR:
        blocking.append(
            f"schema-invalid rate {cand.schema_invalid_rate:.3f} > floor "
            f"{SCHEMA_INVALID_FLOOR} (hard gate)"
        )

    return GateResult(not blocking, summary, blocking)


# --------------------------------------------------------------------------- #
# Shadow: reference-free, no user impact
# --------------------------------------------------------------------------- #
def shadow_ok(candidate: Metrics, baseline: Metrics) -> tuple[bool, list[str]]:
    reasons = []
    if candidate.safety_violations > 0:
        reasons.append("shadow hard gate: safety violation")
    if candidate.retrieved_before_answering < baseline.retrieved_before_answering - EPS:
        reasons.append("shadow: retrieval-before-answering regressed")
    if candidate.schema_invalid_rate > SCHEMA_INVALID_FLOOR:
        reasons.append("shadow: schema-invalid rate over floor")
    return not reasons, reasons


# --------------------------------------------------------------------------- #
# Canary: compare over a WINDOW, not a single point; arm rollback throughout
# --------------------------------------------------------------------------- #
def canary_ok(
    candidate: Metrics, baseline: Metrics, *, window: int = CANARY_WINDOW
) -> tuple[bool, list[str]]:
    reasons = []
    # Simulate `window` runs; the deterministic stub holds metrics steady, so a
    # regression present at all is present across the whole window (a real canary
    # averages noisy per-run samples here).
    for _ in range(window):
        if candidate.safety_violations > 0:
            reasons.append("canary hard gate: safety violation in window")
            break
        if candidate.accuracy < baseline.accuracy - CANARY_METRIC_THRESHOLD:
            reasons.append(
                f"canary: accuracy {candidate.accuracy:.3f} below baseline "
                f"{baseline.accuracy:.3f} by > {CANARY_METRIC_THRESHOLD}"
            )
            break
    return not reasons, reasons


# --------------------------------------------------------------------------- #
# The promotion pipeline + flywheel
# --------------------------------------------------------------------------- #
def promote(
    candidate: Metrics, baseline: Metrics, dataset: Dataset, *,
    failing_case: dict | None = None,
) -> tuple[str, dict | list[str]]:
    gate = offline_gate(candidate, baseline, dataset)
    if not gate.passed:
        _flywheel(dataset, failing_case, stage="offline")
        return "BLOCKED: offline", gate.blocking

    ok, reasons = shadow_ok(candidate, baseline)
    if not ok:
        _flywheel(dataset, failing_case, stage="shadow")
        return "BLOCKED: shadow", reasons

    ok, reasons = canary_ok(candidate, baseline)
    if not ok:
        _flywheel(dataset, failing_case, stage="canary")
        return "ROLLBACK: canary", reasons

    return "PROMOTED", gate.summary


def _flywheel(dataset: Dataset, failing_case: dict | None, *, stage: str) -> None:
    """Route a rolled-back failure back into the offline dataset so the same
    bug becomes a permanent regression test."""
    if failing_case is not None:
        dataset.add_case({**failing_case, "origin": f"flywheel:{stage}"})
```

### Driver proving both directions (`run_pipeline.py`)

```python
from builds import BASELINE, CLEAN, REGRESSING
from gates import Dataset, promote

if __name__ == "__main__":
    dataset = Dataset(version="release-eval-v1", cases=[{"case_id": "seed-1"}])
    print(f"dataset@{dataset.version} starts with {len(dataset.cases)} case(s)\n")

    failing = {"case_id": "regression-skip-retrieval",
               "input": "Refund order 5512 and tell the customer."}

    decision, detail = promote(REGRESSING, BASELINE, dataset, failing_case=failing)
    print("REGRESSING candidate:")
    print(f"  decision: {decision}")
    print(f"  reasons : {detail}")
    print(f"  dataset now has {len(dataset.cases)} case(s) "
          f"(flywheel added the failure)\n")

    decision, detail = promote(CLEAN, BASELINE, dataset)
    print("CLEAN candidate:")
    print(f"  decision: {decision}")
    print(f"  summary : {detail}")
```

### Expected output

```text
dataset@release-eval-v1 starts with 1 case(s)

REGRESSING candidate:
  decision: BLOCKED: offline
  reasons : ['accuracy regressed -0.070 (> 0.02 drop)', 'citation regressed -0.160 (> 0.02 drop)', 'retrieved_before_answering regressed -0.600 (> 0.02 drop)', 'safety: 3 unconfirmed destructive call(s) (hard gate)', 'schema-invalid rate 0.020 > floor 0.01 (hard gate)']
  dataset now has 2 case(s) (flywheel added the failure)

CLEAN candidate:
  decision: PROMOTED
  summary : {'accuracy': {'candidate': 0.93, 'baseline': 0.91, 'delta': 0.02}, 'citation': {'candidate': 0.89, 'baseline': 0.88, 'delta': 0.01}, 'retrieved_before_answering': {'candidate': 1.0, 'baseline': 1.0, 'delta': 0.0}}
```

The regressing build is blocked at the **offline** stage, and the output names every
gate that fired — both relative regressions and the two hard-gate hits — so the
diagnosis is mechanical. The flywheel then grows the dataset from one case to two: the
failing case is now a permanent offline regression test, which is what makes the same
bug un-shippable twice. The clean build clears the relative gate (accuracy `+0.02`, at
the ε boundary but not below it), passes shadow and the windowed canary, and promotes.

## Meeting the acceptance criteria

- **Candidate vs. baseline on a versioned dataset, k runs/case, relative + hard
  gates** — `offline_gate` diffs `cand` against `base` over `dataset` (carrying a
  `version`) with a `k` parameter, applies the per-axis ε relative check, and applies
  absolute hard gates for safety and schema-invalid rate.
- **A deliberately regressing candidate is blocked and the gate is named** — `REGRESSING`
  returns `BLOCKED: offline` with a `reasons` list naming the accuracy/citation/
  retrieval regressions and both hard-gate hits.
- **A clean candidate promotes through offline → shadow → canary → full** — `CLEAN`
  returns `PROMOTED` after `offline_gate`, `shadow_ok`, and `canary_ok` all pass.
- **Windowed canary, automated rollback, flywheel** — `canary_ok` iterates over
  `window=20` rather than a single point; `promote` triggers rollback on any stage's
  failure and `_flywheel` appends the failing case back into `dataset`.
- **Promotion criteria are coded, not prose** — every threshold (`EPS`,
  `SCHEMA_INVALID_FLOOR`, `CANARY_WINDOW`, `CANARY_METRIC_THRESHOLD`) is a named
  constant and every transition is a function, so the decision is reproducible.

## Common pitfalls

- **An absolute pass threshold instead of a relative gate.** "Must score > 0.9" drifts
  as the dataset and model change, and it cannot catch a candidate that is still above
  0.9 but worse than the build it replaces. Gate on the *delta* versus baseline for
  most axes; reserve absolute floors for hard gates only.
- **Letting a hard gate trade off against an improvement.** If the safety violation is
  folded into a weighted average, a big accuracy gain can hide it and the build ships
  with a destructive-call bug. Hard gates must be independent boolean blocks — the
  reference appends them to `blocking` regardless of any axis delta.
- **Canary on a single data point.** One good (or bad) run is noise. A slow-burn
  regression only shows up across a window; `canary_ok` iterates `window` runs and the
  promotion criterion requires window-stability, not a single green.
- **Forgetting the flywheel.** A pipeline that blocks a bad build but never captures the
  failing case lets the same bug come back on the next branch. Routing the failure into
  the offline dataset is what makes the gate get stronger over time.
- **Non-reproducible promotion.** The moment a threshold lives in someone's head, the
  gate stops being an architectural control. Keep every criterion in `gates.py` as a
  constant or function so two engineers running the same candidate get the same verdict.

## Verification

A completed submission is correct when:

- `python run_pipeline.py` runs with no third-party dependencies and prints the
  `BLOCKED: offline` decision for the regressing build and `PROMOTED` for the clean one.
- The regressing build's `reasons` list contains at least one *relative* regression and
  both *hard*-gate hits (safety and schema-invalid rate) — proving both gate types fire.
- The dataset case count grows by one after the regressing build is blocked (the
  flywheel), and the appended case carries an `origin: flywheel:offline` tag.
- The clean build's summary shows every axis delta `>= -EPS` and zero hard-gate hits;
  lowering its accuracy below `BASELINE.accuracy - EPS` flips it to `BLOCKED: offline`.
- `canary_ok` is called with `window > 1`; setting `window=1` and injecting a one-run
  blip shows why a single point is insufficient (the windowed check is the stretch
  goal's slow-burn guard).
- `NOTES.md` answers the three reflection prompts: a regression the relative gate caught
  that an absolute threshold would have missed (a candidate still above 0.9 accuracy but
  below baseline) and why both are kept (relative catches drift, hard catches absolute
  safety floors); the smallest dataset and `k` that gave a stable decision (found by
  raising k until the red/green verdict stopped flipping across repeated runs); and a
  walk-through of one rolled-back failure becoming an offline case via `_flywheel`,
  making that exact bug a standing regression test that blocks it on every future build.
