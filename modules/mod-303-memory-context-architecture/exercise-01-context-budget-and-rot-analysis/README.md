# mod-303-memory-context-architecture/exercise-01 — Solution

## Approach

This is a measurement-and-design exercise, so the deliverable is evidence, not a
clever loop. The worked solution instruments a small reference agent — a
research loop that retrieves documents and answers questions over a growing
window — and produces five artifacts the exercise asks for:

1. Per-turn telemetry with the window broken into the Chapter 1 line items and a
   stale fraction.
2. A filled budget worksheet with an eviction rule per line item, compared
   against measured reality.
3. A quality-vs-window-length curve that locates the rot peak.
4. A distractor probe that tests lost-in-the-middle at three depths.
5. A rot budget whose every threshold traces back to one of the measurements.

The agent and model are deliberately under-specified in the source exercise
(any provider, any 10+ turn loop). The reference run below targets a 200K-token
window, uses an LLM judge with a 1–3 rubric for the quality signal, and counts
tokens with the provider tokenizer. The *shape* of the result — a non-monotonic
quality curve and a depth-dependent retrieval miss — is what generalizes;
re-derive the exact numbers for your own model and task.

The discipline that makes this an architect artifact: every number in the rot
budget at the end points back to a row in the telemetry or a cell in the
distractor table. A threshold you cannot trace to a measurement is a guess, and
guesses do not survive a design review.

## Reference solution

### 1. Instrument the window

The instrumentation is a single dataclass appended as JSON lines, one record per
turn. The only non-trivial part is attributing every token in the prompt to
exactly one component so the slices sum to `tokens_in_window` — overlap or gaps
make the budget comparison meaningless.

```python
# instrument.py — per-turn window telemetry
import json
from dataclasses import dataclass, asdict


@dataclass
class TurnRecord:
    turn_index: int
    tokens_in_window: int
    tok_system_tools: int      # stable: system prompt + tool defs
    tok_working_set: int       # task framing, current plan
    tok_retrieved: int         # chunks pulled this turn (JIT)
    tok_history: int           # prior turns' messages + tool outputs
    tok_subagent: int          # distilled sub-agent returns
    stale_fraction: float      # history older than 5 turns / tokens_in_window
    quality: float | None      # filled post-hoc by judge or success check

    def check_sums(self) -> None:
        parts = (
            self.tok_system_tools
            + self.tok_working_set
            + self.tok_retrieved
            + self.tok_history
            + self.tok_subagent
        )
        # Every prompt token is attributed to exactly one component.
        assert parts == self.tokens_in_window, (
            f"turn {self.turn_index}: components {parts} "
            f"!= window {self.tokens_in_window}"
        )


def log_turn(rec: TurnRecord, path: str = "run.jsonl") -> None:
    rec.check_sums()
    with open(path, "a", encoding="utf-8") as f:
        f.write(json.dumps(asdict(rec)) + "\n")
```

The `stale_fraction` is computed by tagging each history message with the turn
it was created on, then summing tokens whose age (`current_turn - created_turn`)
exceeds 5:

```python
def stale_fraction(history: list[dict], current_turn: int, window_tokens: int) -> float:
    stale = sum(
        msg["tokens"]
        for msg in history
        if current_turn - msg["created_turn"] > 5
    )
    return stale / max(1, window_tokens)
```

A representative slice of the resulting `run.jsonl` from the reference 40-turn
run (token counts rounded):

```text
turn  window  sys+tool  working  retrieved  history  subagent  stale%  quality
────  ──────  ────────  ───────  ─────────  ───────  ────────  ──────  ───────
  1     9100      7800     1300          0        0         0    0.00      3.0
  5    34200      7800     1500       8200    14900      1800    0.18      3.0
 10    71600      7800     1600       9100    49300      3800    0.41      2.7
 20   128400      7800     1700       9400   104700      4800    0.63      2.0
 40   181900      7800     1900      10100   157400      4700    0.79      1.3
```

Two facts jump off this table before any plotting: `history` is the line item
that grows without bound (it is 86% of the window by turn 40), and the stale
fraction climbs in lockstep. The retrieved and sub-agent slices stay roughly
flat — they are already governed (drop-after-use, distilled returns). History is
the line item to attack first.

### 2. The budget worksheet

The designed budget, filled for this agent against a 200K window at a 65%
steady-state target (130K tokens). The right-most column — the eviction rule per
line item — is the part that turns a pie chart into an architecture.

```text
CONTEXT BUDGET WORKSHEET
Agent: research loop (retrieve → answer, 10–40 turns)
Model window: 200,000 tokens
Steady-state target: ~65% = 130,000 tokens (the budget)

Component             Strategy            Budget    Cache?  Eviction rule
-------------------- ------------------- --------- ------- -----------------------------
System + instructions stable, front       5,200    yes     never
Tool definitions      stable, front       2,600    yes     never
Working set (task)    pre-loaded         13,000    no      replace per task/phase
Structured notes      note-taking (ref)   3,900    no      summary only; detail in store
Retrieved content     JIT retrieval      17,000    no      drop after use (same turn)
Conversation history  compaction         26,000    partial summarize when slice exceeded
Sub-agent returns     isolation           6,500    no      distilled summary only (cap 800)
Headroom (reasoning)  reserved           55,800    n/a     never fill
-------------------- ------------------- --------- ------- -----------------------------
                                        130,000

Triggers to specify:
  - Compaction fires when:   history slice > 26,000 tokens
  - Sub-agent spawned when:  a sub-task needs > 8,000 tokens of its own exploration
  - JIT vs. pre-load:        pre-load if needed on > 60% of turns, else reference
```

Budget vs. measured reality — the comparison the exercise asks for:

```text
Line item             Budgeted   Measured @ turn 40   Verdict
--------------------- ---------- -------------------- -----------------------------
System + tools          7,800              7,800      on budget (stable, as designed)
Working set            13,000              1,900      under (task framing is small)
Retrieved              17,000             10,100      under (k is conservative)
History                26,000            157,400      BLOWN — 6x over, uncapped
Sub-agent               6,500              4,700      on budget
Headroom               55,800             18,100      consumed by history overrun
```

The reading is unambiguous: **history blew the budget by 6x and ate the
headroom.** Nothing else is close. The architecture's first move is a compaction
trigger on the history slice (built in exercise-03), not a bigger window, not a
fancier reranker. The budget told us where to spend effort.

### 3. Finding the rot

Plot the `quality` column against `tokens_in_window` across the 5/10/20/40-turn
runs. The curve is the artifact:

```text
  judge
  score
  3.0 ●──●
      │    ╲
  2.5 │     ●           quality PEAKS early and FALLS — this is rot,
      │      ╲          not a plateau
  2.0 │       ●
      │        ╲
  1.5 │         ╲
      │          ●
  1.0 └───┬───┬───┬───┬───▶ tokens in window
         9K  34K 72K 128K 182K
              ▲
         peak ≈ 30–35K tokens (~17% of the 200K window)
```

Quality holds at 3.0 through ~34K tokens, then declines monotonically as the
window fills — a textbook rot curve, peaks-and-falls, not flat-and-fine. The
empirical occupancy ceiling is **~35K tokens, roughly 17% of the hard limit.**

This is the single most important — and most counterintuitive — number in the
exercise: quality peaked at 17% occupancy on a 200K-token model. The 83% of the
window below the hard cap is not free capacity; past ~35K it is actively
degrading the agent. "Fill the window" is exactly backwards.

### 4. The distractor probe

One true fact and two plausible-but-wrong distractors placed in a long
(~120K-token) filler window. The needle depth sweeps 10% / 50% / 90%; we record
whether the agent returns the true value, a distractor, or nothing. Twenty
trials per depth.

```text
Distractor probe — needle "launch code is 4471"
  distractors present: "backup code is 9920", "old launch code was 1130"

  needle depth    true (4471)   distractor   nothing
  ────────────    ───────────   ──────────   ───────
   10% (start)        19/20         1/20        0/20
   50% (middle)        8/20         9/20        3/20   ◀── lost in the middle
   90% (end)          18/20         2/20        0/20
```

The middle row is the finding: at 50% depth the agent returns the correct value
barely 40% of the time and returns a *distractor* nearly as often. At the edges
it is near-perfect. **Lost-in-the-middle is confirmed for this model, and the
distractor variant sharpens it** — the failure is not "couldn't find it" (only 3
nothing) but "found the wrong, plausible thing" (9 distractors). That is exactly
the production failure mode: citing a stale-but-related value. A clean needle
test without distractors would have under-stated the risk.

### 5. The rot budget

Every threshold below traces to a measurement above — the citation is in
parentheses.

```text
ROT BUDGET (research loop)
  Occupancy ceiling:  context never exceeds 20% of window (~40K tokens) at
                      steady state.
                      → traces to: quality peak at ~35K / 17% (Task 3 curve);
                        20% leaves a small margin above the measured peak.
  Staleness ceiling:  no more than 35% of the window is history older than
                      5 turns.
                      → traces to: stale_fraction hit 0.41 at turn 10 where
                        quality first dipped to 2.7 (Task 1 telemetry).
  Critical-fact rule: the goal + freshest facts live in the last ~8K tokens.
                      → traces to: 90%-depth recall 18/20 vs. 50%-depth 8/20
                        (Task 4 probe) — the end edge is the safe zone.
  Trigger — occupancy crossed:  compact history to its budgeted slice (26K).
  Trigger — staleness crossed:  re-anchor from structured notes, drop old turns.
  Trigger — long run (> 15 turns): hard reset working context from the store.
                      → traces to: quality fell below 2.0 by turn 20 (Task 1).
```

### Reflection answers

1. **Largest budget vs. measured divergence:** history (26K budgeted, 157K
   measured — 6x). It is uncapped, so it is the line item to attack first; every
   other slice is already within or under budget.
2. **Peak occupancy and headroom:** quality peaked at ~17% occupancy, leaving
   ~83% of the hard limit unused *by design*. "Fill the window" is wrong because
   the function is non-monotonic — past the peak, more tokens dilute attention
   and the agent gets worse, not just slower and more expensive.
3. **Most-implicated rot source:** the distractor probe most implicates
   **distraction** — at mid-depth the agent returned a plausible wrong value
   (9/20) far more than it returned nothing (3/20), so noise is out-competing
   signal rather than the fact simply being absent.

## Meeting the acceptance criteria

- **Per-turn telemetry with Chapter 1 components + stale fraction** —
  `TurnRecord` logs all five line items and `stale_fraction`, with a
  `check_sums()` assertion that the components exactly partition the window
  (Task 1 table).
- **Filled budget worksheet with an eviction rule per line item, and where
  measured diverged** — the worksheet names an eviction rule for all eight rows;
  the budget-vs-measured table flags history as 6x over (Task 2).
- **Quality-vs-window-length plot, with rot occupancy** — the curve peaks at
  ~35K / 17% and falls (Task 3).
- **Distractor probe showing depth effect** — 50%-depth recall collapses to
  8/20 with 9/20 distractor hits; edges near-perfect — lost-in-the-middle
  confirmed with data (Task 4).
- **Rot budget whose every threshold traces to a measurement** — each ceiling
  and trigger cites the specific Task it derives from (Task 5).

## Common pitfalls

- **Component slices that don't sum to the window.** If `tok_history` and
  `tok_retrieved` double-count overlapping tokens, or filler is unattributed,
  the budget comparison is fiction. The `check_sums()` assertion catches this at
  log time — keep it.
- **Stopping at a clean needle test.** A needle-in-a-haystack with no
  distractors hides the dangerous failure. The agent that aces a clean probe and
  returns a distractor 45% of the time at mid-depth is the one that cites a stale
  tool output in production. Always run the distractor variant.
- **Reading a plateau as "fine."** A flat quality line at low scores is not
  success — it may mean the task is too easy to surface rot, or the judge rubric
  is too coarse. Push the run longer (40+ turns) until the curve either rises,
  plateaus *at a high score*, or peaks-and-falls.
- **Inventing rot-budget numbers.** A ceiling of "80% occupancy" that traces to
  nothing is exactly the guess this exercise exists to eliminate. If you can't
  cite the measurement, you haven't measured enough — extend the run.
- **Confusing capacity with budget.** The 200K hard limit is not the target. The
  peak at 17% is. Designing to the cap re-creates the rot you just measured.

## Verification

- Run the agent at 5, 10, 20, and 40 turns; confirm `run.jsonl` has one record
  per turn and that `check_sums()` passes on every record (no attribution gaps).
- Plot `quality` vs. `tokens_in_window`; confirm the curve is non-monotonic
  (peaks-and-falls) and read off the peak occupancy.
- Run the distractor probe 20 trials × 3 depths; confirm mid-depth recall is
  measurably below edge recall (or, if your model refutes lost-in-the-middle,
  that the curve is flat across depth — either is a valid, evidenced result).
- Re-read the rot budget and check that each threshold has a parenthetical
  citation to a Task; any threshold without one fails the exercise's own
  acceptance bar.
- Stretch check: overlay the quality curve with compaction on (exercise-03) and
  confirm the peak moves right — the diagnosis-to-mitigation loop closes.
