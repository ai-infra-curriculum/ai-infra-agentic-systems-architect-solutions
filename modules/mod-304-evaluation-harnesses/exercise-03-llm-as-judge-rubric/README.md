# mod-304-evaluation-harnesses/exercise-03-llm-as-judge-rubric — Solution

## Approach

The exercise asks you to treat the judge as a **measurement instrument** — author a
four-axis rubric, then *calibrate it against human labels* and prove per-axis
agreement before declaring any axis gateable. The deliverable is not a clever prompt;
it is a judge you have measured and shown to agree well enough to gate on, plus an
honest verdict on which axes you cannot yet trust.

Design choices that drive the reference implementation:

- **Four separate axes, each on a 0/1/2 anchored scale.** Accuracy, completeness,
  citation faithfulness, tool efficiency. No blended "overall quality 1–10" — a vector
  you can threshold and diagnose, not a scalar you cannot act on.
- **Evidence before score, in the output schema.** The judge must quote the specific
  claim/citation/trace fact *before* it assigns a level, and the JSON schema orders
  the `evidence` field ahead of `level`. This makes scores auditable and measurably
  more reliable, because the model commits to a reason before a number.
- **The judge is pinned and reproducible.** Fixed model + version string, temperature
  0, JSON-schema-constrained output. The version string is recorded with every run, so
  a silent model upgrade can never invalidate a historical comparison unnoticed.
- **Calibration is the centerpiece, and it is runnable offline.** A `RecordedJudge`
  replays pinned judge outputs against a hand-labeled gold set, so the agreement
  mathematics — exact agreement, mean absolute error, Cohen's quadratic kappa, plus a
  human–human ceiling — runs without an API key. The same `score()` interface is what a
  live judge implements; only the constructor changes.

The load-bearing demonstration is the **inspect-fix-remeasure** cycle on the citation
axis: the v1 rubric anchor for level 1 is ambiguous ("a citation is missing OR doesn't
support its claim"), the judge disagrees with humans on the borderline cases, and a
v2 anchor that splits the two failure modes lifts citation agreement to its human
ceiling. Tool efficiency stays below bar and is declared **not-yet-gateable** with a
deterministic fallback — the honest outcome the exercise is built around.

## Reference solution

### The pinned rubric and judge interface (`judge.py`)

```python
"""Four-axis LLM-as-judge rubric, pinned for reproducibility, plus a recorded
judge so calibration runs offline. A live judge implements the same score()."""
from __future__ import annotations

from dataclasses import dataclass

JUDGE_VERSION = "judge/anthropic-claude:2026-06-01@t0"  # model+version+temp, pinned

JUDGE_SYSTEM = (
    "You are a strict evaluation judge. For EACH axis: first quote the specific "
    "evidence (a claim, a citation pair, or a trace fact) that determines the "
    "level, THEN assign a level from the anchored scale. Use ONLY the anchored "
    "levels. Do not reward fluency or length. If evidence is missing, score "
    "conservatively (the lower level)."
)

# v2 rubric — citation level 1 split into two concrete failure modes after
# the calibration cycle below.
RUBRIC = {
    "accuracy": {
        2: "all claims correct and consistent with the retrieved evidence",
        1: "a minor error that does not change the conclusion",
        0: "a claim is wrong or contradicts the evidence",
    },
    "completeness": {
        2: "addresses every part the task asked for",
        1: "covers the main ask but misses one sub-part",
        0: "leaves a primary part of the task unanswered",
    },
    "citation": {
        2: "every claim needing support cites a source that actually backs it",
        1: "a claim needing support is uncited, but no citation is mismatched",
        0: "a citation is fabricated or points at a source that does not "
           "support its claim, OR citations are absent entirely",
    },
    "tool_efficiency": {
        2: "no redundant or unnecessary tool calls",
        1: "exactly one avoidable or redundant call",
        0: "substantial wasted tool use",
    },
}

AXES = list(RUBRIC)


@dataclass
class AxisScore:
    evidence: str   # quoted BEFORE the level — enforced by schema ordering
    level: int


# JSON schema the live judge is constrained to (evidence precedes level).
def output_schema() -> dict:
    axis_schema = {
        "type": "object",
        "properties": {
            "evidence": {"type": "string"},
            "level": {"type": "integer", "enum": [0, 1, 2]},
        },
        "required": ["evidence", "level"],
        "additionalProperties": False,
    }
    return {
        "type": "object",
        "properties": {axis: axis_schema for axis in AXES},
        "required": AXES,
        "additionalProperties": False,
    }


def build_judge_prompt(task: str, answer: str, evidence: str, trace: str) -> str:
    return (
        f"TASK:\n{task}\n\n"
        f"RETRIEVED EVIDENCE:\n{evidence}\n\n"
        f"AGENT ANSWER:\n{answer}\n\n"
        f"TOOL TRACE:\n{trace}\n\n"
        f"RUBRIC (use these exact levels):\n{RUBRIC}\n\n"
        "Return JSON: for every axis an object {evidence: str, level: int}. "
        "Quote the evidence before the level."
    )


class RecordedJudge:
    """Replays pinned judge outputs by case_id so calibration is reproducible
    offline. A live judge has the same score() signature and calls the model
    with build_judge_prompt + output_schema at temperature 0."""

    version = JUDGE_VERSION

    def __init__(self, recorded: dict[str, dict[str, AxisScore]]) -> None:
        self._recorded = recorded

    def score(self, case_id: str) -> dict[str, AxisScore]:
        return self._recorded[case_id]
```

### Calibration mathematics (`calibration.py`)

```python
"""Per-axis agreement metrics: exact agreement, MAE, and quadratic-weighted
Cohen's kappa (an ordinal rank measure). No averaging across axes."""
from __future__ import annotations

from judge import AXES


def _pairs(human: dict, judge: dict, axis: str) -> list[tuple[int, int]]:
    return [
        (human[cid][axis].level, judge[cid][axis].level)
        for cid in human
        if cid in judge
    ]


def exact_agreement(human: dict, judge: dict, axis: str) -> float:
    pairs = _pairs(human, judge, axis)
    return sum(h == j for h, j in pairs) / len(pairs)


def mean_abs_error(human: dict, judge: dict, axis: str) -> float:
    pairs = _pairs(human, judge, axis)
    return sum(abs(h - j) for h, j in pairs) / len(pairs)


def quadratic_kappa(human: dict, judge: dict, axis: str, k: int = 3) -> float:
    """Cohen's quadratic-weighted kappa over levels 0..k-1."""
    pairs = _pairs(human, judge, axis)
    n = len(pairs)
    obs = [[0] * k for _ in range(k)]
    hist_h, hist_j = [0] * k, [0] * k
    for h, j in pairs:
        obs[h][j] += 1
        hist_h[h] += 1
        hist_j[j] += 1

    num = den = 0.0
    for a in range(k):
        for b in range(k):
            w = ((a - b) ** 2) / ((k - 1) ** 2)
            expected = hist_h[a] * hist_j[b] / n
            num += w * obs[a][b]
            den += w * expected
    return 1.0 - num / den if den else 1.0


def per_axis_report(human: dict, judge: dict) -> dict[str, dict]:
    return {
        axis: {
            "exact": round(exact_agreement(human, judge, axis), 3),
            "mae": round(mean_abs_error(human, judge, axis), 3),
            "kappa": round(quadratic_kappa(human, judge, axis), 3),
            "n": len(_pairs(human, judge, axis)),
        }
        for axis in AXES
    }


def gateability(report: dict, ceilings: dict[str, float], bar: float) -> dict:
    """An axis is gateable if exact agreement clears the bar AND stays within
    a small margin of its human-human ceiling (you cannot beat the ceiling)."""
    verdict = {}
    for axis, stats in report.items():
        ceiling = ceilings.get(axis)
        clears_bar = stats["exact"] >= bar
        near_ceiling = ceiling is None or stats["exact"] >= ceiling - 0.10
        gateable = clears_bar and near_ceiling
        verdict[axis] = {
            "gateable": gateable,
            "exact": stats["exact"],
            "ceiling": ceiling,
            "fallback": None if gateable else "deterministic proxy / human review",
        }
    return verdict
```

### Gold set, recorded judge runs, and driver (`run_calibration.py`)

```python
"""A 12-case slice of a 30-80 case gold set (abridged for the reference; the
shape and the math are identical at full size). Human labels cover the score
range. judge_v1 has the ambiguous citation anchor; judge_v2 has the fix."""
from calibration import gateability, per_axis_report
from judge import AxisScore, RecordedJudge

A = AxisScore  # brevity


def axes(acc, comp, cit, eff):
    return {
        "accuracy": A("...", acc),
        "completeness": A("...", comp),
        "citation": A("...", cit),
        "tool_efficiency": A("...", eff),
    }


# Human gold labels (cover 0/1/2 on every axis).
HUMAN = {
    "g01": axes(2, 2, 2, 2), "g02": axes(2, 1, 2, 1), "g03": axes(1, 2, 1, 2),
    "g04": axes(0, 1, 0, 2), "g05": axes(2, 2, 1, 0), "g06": axes(1, 0, 2, 1),
    "g07": axes(2, 2, 2, 2), "g08": axes(0, 0, 0, 0), "g09": axes(2, 1, 1, 2),
    "g10": axes(1, 2, 2, 1), "g11": axes(2, 2, 0, 0), "g12": axes(1, 1, 1, 2),
}

# Second labeler on a 12-case subset -> human-human ceiling per axis.
HUMAN_2 = {
    "g01": axes(2, 2, 2, 2), "g02": axes(2, 1, 2, 0), "g03": axes(1, 2, 1, 2),
    "g04": axes(0, 1, 1, 2), "g05": axes(2, 2, 1, 1), "g06": axes(1, 0, 2, 1),
    "g07": axes(2, 2, 2, 2), "g08": axes(0, 1, 0, 0), "g09": axes(2, 1, 1, 2),
    "g10": axes(1, 2, 2, 0), "g11": axes(2, 2, 0, 1), "g12": axes(2, 1, 1, 2),
}

# v1 judge: citation anchor ambiguous -> disagrees on borderline citation cases
# (g03, g05, g09, g12) and on tool efficiency.
JUDGE_V1 = {
    "g01": axes(2, 2, 2, 2), "g02": axes(2, 1, 2, 2), "g03": axes(1, 2, 2, 2),
    "g04": axes(0, 1, 1, 2), "g05": axes(2, 2, 2, 1), "g06": axes(1, 0, 2, 2),
    "g07": axes(2, 2, 2, 2), "g08": axes(0, 0, 0, 1), "g09": axes(2, 1, 2, 2),
    "g10": axes(1, 2, 2, 2), "g11": axes(2, 2, 0, 1), "g12": axes(1, 1, 2, 2),
}

# v2 judge: citation anchor split -> citation agreement recovers; tool
# efficiency still noisy (the not-yet-gateable axis).
JUDGE_V2 = {
    "g01": axes(2, 2, 2, 2), "g02": axes(2, 1, 2, 2), "g03": axes(1, 2, 1, 2),
    "g04": axes(0, 1, 0, 2), "g05": axes(2, 2, 1, 1), "g06": axes(1, 0, 2, 2),
    "g07": axes(2, 2, 2, 2), "g08": axes(0, 0, 0, 1), "g09": axes(2, 1, 1, 2),
    "g10": axes(1, 2, 2, 2), "g11": axes(2, 2, 0, 0), "g12": axes(1, 1, 1, 2),
}


def show(title, report):
    print(title)
    for axis, s in report.items():
        print(f"  {axis:<16} exact={s['exact']:.2f} mae={s['mae']:.2f} "
              f"kappa={s['kappa']:.2f} n={s['n']}")


if __name__ == "__main__":
    ceilings = {
        axis: per_axis_report(HUMAN, HUMAN_2)[axis]["exact"]
        for axis in per_axis_report(HUMAN, HUMAN_2)
    }
    print(f"judge version: {RecordedJudge.version}\n")
    print("human-human ceiling (exact agreement):")
    for axis, c in ceilings.items():
        print(f"  {axis:<16} {c:.2f}")
    print()

    show("v1 judge vs human:", per_axis_report(HUMAN, JUDGE_V1))
    print()
    show("v2 judge vs human (after citation-anchor fix):",
         per_axis_report(HUMAN, JUDGE_V2))
    print()

    verdict = gateability(per_axis_report(HUMAN, JUDGE_V2), ceilings, bar=0.80)
    print("gateability (bar=0.80 exact, within 0.10 of ceiling):")
    for axis, v in verdict.items():
        tag = "GATEABLE" if v["gateable"] else f"NOT YET ({v['fallback']})"
        print(f"  {axis:<16} exact={v['exact']:.2f} ceiling={v['ceiling']:.2f} -> {tag}")
```

### Expected output

```text
judge version: judge/anthropic-claude:2026-06-01@t0

human-human ceiling (exact agreement):
  accuracy         0.92
  completeness     0.92
  citation         0.92
  tool_efficiency  0.67

v1 judge vs human:
  accuracy         exact=1.00 mae=0.00 kappa=1.00 n=12
  completeness     exact=1.00 mae=0.00 kappa=1.00 n=12
  citation         exact=0.58 mae=0.42 kappa=0.70 n=12
  tool_efficiency  exact=0.50 mae=0.50 kappa=0.56 n=12

v2 judge vs human (after citation-anchor fix):
  accuracy         exact=1.00 mae=0.00 kappa=1.00 n=12
  completeness     exact=1.00 mae=0.00 kappa=1.00 n=12
  citation         exact=1.00 mae=0.00 kappa=1.00 n=12
  tool_efficiency  exact=0.58 mae=0.42 kappa=0.67 n=12

gateability (bar=0.80 exact, within 0.10 of ceiling):
  accuracy         exact=1.00 ceiling=0.92 -> GATEABLE
  completeness     exact=1.00 ceiling=0.92 -> GATEABLE
  citation         exact=1.00 ceiling=0.92 -> GATEABLE
  tool_efficiency  exact=0.58 ceiling=0.67 -> NOT YET (deterministic proxy / human review)
```

The citation axis moves from `0.58` to `1.00` after splitting the level-1 anchor — it
now sits at or above its human ceiling and is gateable. Tool efficiency stays at
`0.58`, below both the bar and its own `0.67` ceiling, so it is declared
**not-yet-gateable**; the honest fallback is the deterministic redundancy count from
exercise-02 (which has a correct answer the judge was only approximating).

## Meeting the acceptance criteria

- **Four separate axes on a short anchored scale, evidence before score** — `RUBRIC`
  defines accuracy/completeness/citation/tool_efficiency, each at 0/1/2 with a concrete
  anchor per level; `output_schema` orders `evidence` ahead of `level`, and
  `JUDGE_SYSTEM` requires quoting evidence first.
- **Judge pinned and reproducible** — `JUDGE_VERSION` records model + version +
  temperature 0; the output is JSON-schema-constrained; the recorded judge replays the
  same outputs deterministically.
- **Per-axis agreement against a range-covering gold set with a human ceiling** —
  `HUMAN` covers 0/1/2 on every axis, `HUMAN_2` gives a second-labeler ceiling, and
  `per_axis_report` reports exact agreement, MAE, and quadratic kappa per axis with no
  cross-axis averaging.
- **Inspect-fix-remeasure, then per-axis gateability** — v1→v2 shows the citation
  anchor fix raising agreement from `0.58` to `1.00`; `gateability` labels three axes
  gateable and tool efficiency not-yet-gateable with a named fallback.

## Common pitfalls

- **Averaging across axes.** A mean "judge agreement 0.79" hides that citation was
  fixable and tool efficiency is not gateable. Report — and gate — per axis, always.
- **An all-perfect gold set.** If every gold case scores 2, you learn nothing about
  how the judge handles failures, and your agreement number is meaningless. Cover the
  range; include clear 0s and 1s (the gold set here deliberately does).
- **Gating without a ceiling.** Without measuring human–human agreement you cannot
  tell a "bad judge" from an "ambiguous rubric." Tool efficiency's `0.67` ceiling shows
  even humans disagree there — the judge is not the only problem, so the fix is a
  deterministic proxy, not more prompt-tuning.
- **Score before evidence.** If the schema lets the judge emit the number first, it
  rationalizes afterward and reliability drops. The `evidence`-then-`level` ordering is
  load-bearing, not cosmetic.
- **Treating a passing v1 as done.** The first rubric almost always has an ambiguous
  anchor. Skipping the inspect-fix-remeasure cycle ships a judge that disagrees with
  humans on exactly the borderline cases that matter most.

## Verification

A completed submission is correct when:

- `python run_calibration.py` runs (only the standard library is needed) and prints
  the ceiling block, the v1 and v2 per-axis reports, and the gateability verdict.
- The citation axis improves from v1 to v2 (here `0.58` → `1.00`) after the level-1
  anchor split — the inspect-fix-remeasure proof.
- Tool efficiency is labelled `NOT YET` with a stated fallback, and its judge agreement
  (`0.58`) is at or below its human–human ceiling (`0.67`) — you cannot beat the
  ceiling.
- Every gold case carries a label on all four axes, and at least one case scores 0 and
  at least one scores 2 on each axis (range coverage).
- Changing `JUDGE_VERSION` or the recorded outputs changes the printed version line and
  the metrics, demonstrating the judge is pinned and version-tracked.
- `NOTES.md` answers the three reflection prompts: the lowest-agreement axis (tool
  efficiency) and the edit that helped citation most (splitting the level-1 anchor into
  "uncited" vs. "mismatched"); a bias you detected and how (e.g., self-preference,
  mitigated by using a different model family for the judge than the agent under test);
  and whether the worst axis's gap is a judge limitation or rubric ambiguity (tool
  efficiency's human ceiling of `0.67` shows the ambiguity is in the *rubric/task*, not
  the model — distinguishable because humans disagree there too).
