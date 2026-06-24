# mod-301-agentic-systems-foundations/exercise-01-workflows-vs-agents-decision-matrix — Solution

## Approach

The exercise asks for a *reusable decision instrument*, not five gut calls. So the
work is in two layers: first build the matrix (criteria, weights, scoring scale,
decision rule, tie-break) and freeze it; then run all five candidates through the
*frozen* instrument without retuning per case. The discipline that makes this a
real architect tool is exactly that frozen-weights constraint — if the weights move
to make a case come out "right," the instrument is just rationalization.

Design choices that drive the reference matrix below:

- **Eight criteria**: the six seeded in Chapter 1 plus two I can defend —
  *cost-of-being-wrong asymmetry* (a regulated or financial wrong answer is far
  more expensive than a low-stakes one, pushing toward auditable workflows) and
  *blast radius of model-chosen actions* (the more irreversible side effects the
  system can trigger, the more we want fixed paths fencing the judgment). Both come
  straight from Chapter 1's "the cost of autonomy is not just tokens" section.
- **Signed scoring, `-2 … +2`**, where negative means "this evidence points to
  workflow" and positive means "points to agent." This keeps the weighted sum
  directional: a negative total means workflow, positive means agent.
- **Asymmetric thresholds** to encode Chapter 1's explicit *bias toward workflow*:
  the band that resolves to "agent" starts at `+6`, while "workflow" starts at
  `-4`. The agent has to clear a higher bar than the workflow. The wide middle band
  is **hybrid** — a workflow that escalates to an agent for the hard tail.
- **Tie-break inside the band**: anything in the ambiguous middle that I cannot
  split on evidence collapses *down* to the simpler side (workflow, or a hybrid
  that defaults to its workflow path), never up.

The trap candidate is #3 (code-review bot). It *sounds* agentic — "a bot that
reasons about code" — but it is three independent fixed-shape checks run in
parallel and aggregated. The matrix correctly scores it strongly negative.

## Reference solution

### The frozen instrument

```text
DECISION INSTRUMENT — weights & rule (frozen across all candidates)

  Criterion                                   Weight   Score scale
  ------------------------------------------  ------   ------------------------
  C1 Steps enumerable in advance?              3       -2 fully .. +2 not at all
  C2 Path shape stable across runs?            3       -2 identical .. +2 varies
  C3 Predictable cost/latency required?        2       -2 hard req .. +2 tolerant
  C4 Deterministic replay required?            2       -2 mandated .. +2 not needed
  C5 Needs mid-task course-correction?         3       -2 never .. +2 essential
  C6 Input space bounded & known?              2       -2 bounded .. +2 open-ended
  C7 Cost-of-being-wrong asymmetry (added)     2       -2 high stakes .. +2 low
  C8 Blast radius of model-chosen actions      2       -2 large/irrevers .. +2 none
  ------------------------------------------  ------   ------------------------

  Weighted score = sum( weight_i * score_i )      range: -38 .. +38

  Decision rule:  total <= -4   -> workflow
                  -3 .. +5       -> hybrid (workflow w/ agent escalation)
                  total >= +6    -> agent

  Tie-break:  within the hybrid band, if evidence does not clearly favor the
              agent path, default to the workflow path of the hybrid (and if the
              hybrid's agent fraction is empty, it is simply a workflow).
              Rationale: bias toward predictability (Chapter 1).
```

C7 and C8 carry *negative-leaning* defaults on purpose: most production systems
have some stakes and some side effects, so these two criteria act as a standing
drag toward the workflow side that the genuinely-agentic criteria (C1, C2, C5, C6)
must overcome. That is Chapter 4's "burden of proof sits on complexity" rule,
encoded numerically.

### Candidate 1 — Weekly compliance report from three fixed sources

```text
DECISION MATRIX — Weekly compliance report (3 fixed sources)

  Criterion                              Weight   Score   Weighted
  -------------------------------------  ------   -----   --------
  C1 Steps enumerable in advance?         3        -2       -6
  C2 Path shape stable across runs?       3        -2       -6
  C3 Predictable cost/latency required?   2        -2       -4
  C4 Deterministic replay required?       2        -2       -4
  C5 Needs mid-task course-correction?    3        -2       -6
  C6 Input space bounded & known?         2        -2       -4
  C7 Cost-of-being-wrong asymmetry        2        -2       -4
  C8 Blast radius of model actions        2        -2       -4
  -------------------------------------  ------   -----   --------
  WEIGHTED TOTAL                                            -38

  RECOMMENDATION: workflow (prompt chaining)
  Escalation trigger: none
  One-line rationale: fixed sources + fixed format + regulated audit trail
                      (C1/C2/C4) make every agentic criterion negative.
```

### Candidate 2 — Customer-support assistant investigating across systems

```text
DECISION MATRIX — Support assistant (cross-system investigation)

  Criterion                              Weight   Score   Weighted
  -------------------------------------  ------   -----   --------
  C1 Steps enumerable in advance?         3        +1       +3
  C2 Path shape stable across runs?       3        +2       +6
  C3 Predictable cost/latency required?   2        +1       +2
  C4 Deterministic replay required?       2        +1       +2
  C5 Needs mid-task course-correction?    3        +2       +6
  C6 Input space bounded & known?         2        +1       +2
  C7 Cost-of-being-wrong asymmetry        2         0        0
  C8 Blast radius of model actions        2        -1       -2
  -------------------------------------  ------   -----   --------
  WEIGHTED TOTAL                                            +19

  RECOMMENDATION: agent (for the investigation phase)
  Escalation trigger: n/a (clears the +6 agent threshold outright)
  One-line rationale: the investigation path depends on what each lookup reveals
                      (C2/C5) — you cannot draw the graph in advance.
```

C8 is held at `-1`, not `0`: refunds and account mutations are real side effects.
The matrix says "agent for investigation," but the *acting* boundary (issuing a
refund) is fenced by a hardcoded human-approval gate — a Chapter 3 detail the
matrix flags via C8 rather than resolving on its own.

### Candidate 3 — Code-review bot (security + style + performance) — the trap

```text
DECISION MATRIX — Code-review bot (3 parallel checks)

  Criterion                              Weight   Score   Weighted
  -------------------------------------  ------   -----   --------
  C1 Steps enumerable in advance?         3        -2       -6
  C2 Path shape stable across runs?       3        -2       -6
  C3 Predictable cost/latency required?   2        -1       -2
  C4 Deterministic replay required?       2        -1       -2
  C5 Needs mid-task course-correction?    3        -2       -6
  C6 Input space bounded & known?         2        -1       -2
  C7 Cost-of-being-wrong asymmetry        2        -1       -2
  C8 Blast radius of model actions        2        -2       -4
  -------------------------------------  ------   -----   --------
  WEIGHTED TOTAL                                            -32

  RECOMMENDATION: workflow (parallelization / sectioning)
  Escalation trigger: none
  One-line rationale: three independent fixed-shape checks aggregated — the
                      "agent" framing is a trap (C1/C2/C5 all strongly negative).
```

This is the planted trap: a naive reader sees "AI reviews my code" and reaches for
an agent. The matrix sees three predetermined, independent checks (sectioning,
Chapter 2) and scores it firmly as a workflow. The model judgment lives *inside*
each parallel branch; the control flow around them is fixed.

### Candidate 4 — Onboarding email sequence personalized from a CRM record

```text
DECISION MATRIX — Onboarding email sequence (CRM personalization)

  Criterion                              Weight   Score   Weighted
  -------------------------------------  ------   -----   --------
  C1 Steps enumerable in advance?         3        -2       -6
  C2 Path shape stable across runs?       3        -2       -6
  C3 Predictable cost/latency required?   2        -1       -2
  C4 Deterministic replay required?       2        -1       -2
  C5 Needs mid-task course-correction?    3        -2       -6
  C6 Input space bounded & known?         2        -2       -4
  C7 Cost-of-being-wrong asymmetry        2        -1       -2
  C8 Blast radius of model actions        2        -1       -2
  -------------------------------------  ------   -----   --------
  WEIGHTED TOTAL                                            -30

  RECOMMENDATION: workflow (prompt chaining over a fixed CRM schema)
  Escalation trigger: none
  One-line rationale: fixed schema + fixed sequence (C1/C2/C6) — personalization
                      is template-fill, not runtime path selection.
```

### Candidate 5 — "AI research analyst" choosing which sources to consult

```text
DECISION MATRIX — AI research analyst (open-ended)

  Criterion                              Weight   Score   Weighted
  -------------------------------------  ------   -----   --------
  C1 Steps enumerable in advance?         3        +2       +6
  C2 Path shape stable across runs?       3        +2       +6
  C3 Predictable cost/latency required?   2        +1       +2
  C4 Deterministic replay required?       2        +1       +2
  C5 Needs mid-task course-correction?    3        +2       +6
  C6 Input space bounded & known?         2        +2       +4
  C7 Cost-of-being-wrong asymmetry        2         0        0
  C8 Blast radius of model actions        2        +1       +2
  -------------------------------------  ------   -----   --------
  WEIGHTED TOTAL                                            +28

  RECOMMENDATION: agent (orchestrator-workers in practice)
  Escalation trigger: n/a (clears +6 outright)
  One-line rationale: the analyst decides which sources to pull based on what it
                      finds (C1/C2/C5/C6 all strongly positive) — the defining
                      open-ended task.
```

### Stress-test

**Least-confident output: Candidate 2.** Its total (`+19`) clears the agent
threshold, but several scores sat near the middle (C1 at `+1`, C3/C4/C6 at `+1`).
A team that bounds the investigation to a *known* set of systems and a *known*
escalation order could legitimately re-score C1/C2 toward zero, dropping the total
into the hybrid band — a workflow that escalates to an agent only on the hard tail.
The matrix gives the right *default* (agent) but is honest that this candidate is
the one most sensitive to how tightly the problem is scoped. That sensitivity is
the signal to write an ADR (exercise-03) rather than silently pick a side.

**Adversarial candidate — looks like an agent, is a workflow:**

```text
DECISION MATRIX — "Autonomous invoice-processing agent" (adversarial)

  Description: vendor invoices arrive by email; the system extracts fields,
  validates them against the PO, and either posts the invoice or routes
  exceptions to a human. Marketed internally as an "agent."

  Criterion                              Weight   Score   Weighted
  -------------------------------------  ------   -----   --------
  C1 Steps enumerable in advance?         3        -2       -6
  C2 Path shape stable across runs?       3        -2       -6
  C3 Predictable cost/latency required?   2        -2       -4
  C4 Deterministic replay required?       2        -2       -4
  C5 Needs mid-task course-correction?    3        -2       -6
  C6 Input space bounded & known?         2        -1       -2
  C7 Cost-of-being-wrong asymmetry        2        -2       -4
  C8 Blast radius of model actions        2        -2       -4
  -------------------------------------  ------   -----   --------
  WEIGHTED TOTAL                                            -36

  Naive call: "agent" (it's autonomous!).  Matrix call: workflow (extract ->
  validate -> route), with model judgment confined to extraction. The matrix
  gets it right; no weight surgery needed.
```

The adversary's only genuinely-uncertain criterion is C6 (a new vendor format can
appear), scored `-1` to reflect that. Everything else is fixed: the *name* is
agentic, the *architecture* is a validated chain. The matrix is not fooled.

## Meeting the acceptance criteria

- **Weighted, scored criteria with an explicit rule and stated tie-break** — the
  frozen instrument lists eight weighted criteria, a signed `-2 … +2` scale,
  asymmetric thresholds, and a tie-break that collapses ambiguity toward workflow.
- **All five candidates scored with per-criterion values and a recommendation** —
  each has a full filled matrix, a weighted total, a recommendation, and an
  escalation-trigger line.
- **A trap correctly identified** — Candidate 3 (code-review bot): naive reader
  says agent, matrix says workflow (sectioning), reasoning written out. The
  adversarial invoice candidate is a second, deliberately constructed trap.
- **Every recommendation tied to specific criteria, not vibes** — each rationale
  cites the criteria (e.g. "C1/C2/C5") that drove it.
- **Same weights across all candidates** — the weight column is identical in every
  table; only the scores change. The instrument is reusable, not retrofitted.

## Common pitfalls

- **Retuning weights per candidate to get the "right" answer.** The instant the
  weights move per case, the matrix stops being an instrument and becomes a story
  you tell after deciding. If a candidate comes out wrong, fix the *scores* (your
  reading of the evidence) or accept the matrix's verdict — do not edit the rule.
- **Symmetric thresholds.** A `±4` symmetric rule loses Chapter 1's bias toward
  workflow. The agent band must start *farther* from zero than the workflow band,
  or the instrument quietly over-recommends autonomy.
- **Scoring the marketing name instead of the control flow.** "Autonomous invoice
  agent" scores like a workflow because the *path* is fixed. Always score who owns
  control flow, never the label on the slide.
- **Forgetting blast radius and cost-of-being-wrong.** Two systems with identical
  path shape can deserve different boundaries if one can issue refunds and the
  other writes a draft. C7/C8 are what stop the matrix from treating a financial
  actuator like a text generator.
- **Treating hybrid as a cop-out.** Hybrid is a real recommendation with a *named
  escalation trigger*. A hybrid with no stated trigger is just indecision; always
  write the condition under which the workflow hands off to the agent.

## Verification

A completed submission is correct when:

- The weight column is identical across all five candidate tables plus the
  adversarial one (scan the tables; the weights never change).
- Each weighted total equals the sum of `weight * score` for its table (re-add by
  hand or with a calculator — the totals above all check out).
- Every total maps to its stated recommendation under the frozen rule
  (`<= -4` workflow, `-3..+5` hybrid, `>= +6` agent).
- Candidate 3 resolves to a workflow despite its agentic framing, with the trap
  reasoning written out — this is the load-bearing check that the instrument
  resists hype.
- Each recommendation's rationale names at least one criterion, and no rationale
  appeals to "it feels agentic."
- `NOTES.md` answers the three reflection prompts: the single hardest-working
  criterion (here C2, path-shape stability, does the most separation across the
  five), where the tie-break flipped a call, and what input change would collapse
  each hybrid into a pure workflow.
