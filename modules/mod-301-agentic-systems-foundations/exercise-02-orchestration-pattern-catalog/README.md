# mod-301-agentic-systems-foundations/exercise-02-orchestration-pattern-catalog — Solution

## Approach

This is a documentation deliverable, so the model artifact *is* the catalog: five
complete pattern entries, ten task mappings, two composed diagrams with the
workflow/agent boundary marked, and one tradeoff table. The analytical spine
running through all of it is the single line from Chapter 2 — *which patterns have
fixed control flow and which are model-driven* — because that boundary is what
turns "predictable and cheap" into "variable and capped."

Two judgment calls shape the reference answer:

- **Each entry leads with its bounding control.** Chapter 2's recurring note is
  that every pattern needs *one* cap or gate to stay safe (chains need inter-step
  gates, orchestrator-workers needs a fan-out cap, evaluator-optimizer needs an
  iteration cap). I treat that as a first-class field, not an afterthought, because
  the bounding control is the difference between a pattern and an incident.
- **Composition is the real skill.** Single-pattern mapping is easy; the exercise's
  value is in tasks 4 and 7, which are genuine compositions (a router or
  orchestrator on the outside, a chain or sectioning inside a branch). I draw those
  two explicitly and mark exactly where control flow stops being fixed-path.

## Reference solution

### Catalog entry — Prompt chaining

```text
PATTERN ENTRY

  Name:             Prompt chaining
  Diagram:          in -> [LLM 1] -> (gate) -> [LLM 2] -> [LLM 3] -> out
                                       |
                                       +-- fail -> stop / fix
  Fits:             tasks that split into a fixed sequence of subtasks
                    (draft -> translate -> tighten)
  Relative cost:    low (linear in steps)
  Predictability:   high (each step replays deterministically)
  Bounding control: a programmatic gate between steps; without it a chain is
                    just one long, un-checkable prompt
  Primary failure:  an early-step error propagates silently to the end
  Do NOT use for:   open-ended tasks whose step count depends on intermediate
                    results (that is orchestrator-workers or an agent)
```

### Catalog entry — Routing

```text
PATTERN ENTRY

  Name:             Routing
  Diagram:                       +-> [handler A]
                    in -> [route] -> [handler B] -> out
                                   +-> [handler C]
  Fits:             distinct input categories that each deserve different
                    handling (refund / technical / billing)
  Relative cost:    low (one classify call + one handler)
  Predictability:   high (the route set is enumerable)
  Bounding control: a default / fallback route for unclassifiable input, so a
                    miss degrades gracefully instead of crashing
  Primary failure:  misclassification sends input to the wrong specialized path
  Do NOT use for:   categories you cannot enumerate in advance (if you cannot
                    write the routing table, you need model-driven decomposition)
```

### Catalog entry — Parallelization

```text
PATTERN ENTRY

  Name:             Parallelization (sectioning + voting)
  Diagram:                 +-> [LLM] -+
                    in --> +-> [LLM] -+-> (aggregate) -> out
                           +-> [LLM] -+
  Fits:             independent subtasks run together (sectioning), or the same
                    task run N times for confidence/coverage (voting)
  Relative cost:    medium (more tokens; latency ~ slowest branch, not the sum)
  Predictability:   high (branch set is fixed; aggregation rule is code)
  Bounding control: a fixed branch count + a defined aggregation rule (majority,
                    "flag if any branch says unsafe", weighted merge)
  Primary failure:  aggregation hides a dissenting-but-correct branch
  Do NOT use for:   subtasks that depend on each other's output (that is a chain,
                    not parallel branches)
```

### Catalog entry — Orchestrator-workers

```text
PATTERN ENTRY

  Name:             Orchestrator-workers
  Diagram:          task -> [orchestrator: decompose (dynamic)]
                                 |        |        |
                                 v        v        v
                             [worker] [worker] [worker]
                                 |        |        |
                                 +--------+--------+
                                          v
                              [orchestrator: synthesize] -> result
  Fits:             complex tasks whose subtasks are unknown until runtime
                    (multi-file code edits; open-ended research threads)
  Relative cost:    high (token spend multiplies with worker count)
  Predictability:   medium (the decomposition step is model-driven)
  Bounding control: a FAN-OUT CAP (max workers) + per-worker budget, or cost is
                    unbounded by construction
  Primary failure:  bad decomposition spawns redundant or off-target workers
  Do NOT use for:   tasks whose subtasks ARE predictable — that is parallel
                    sectioning at a fraction of the cost
```

### Catalog entry — Evaluator-optimizer

```text
PATTERN ENTRY

  Name:             Evaluator-optimizer
  Diagram:          in -> [generator] -> candidate -> [evaluator]
                              ^                            |
                              +--------- feedback ---------+
                                                           | pass / cap
                                                           v
                                                          out
  Fits:             tasks with clear, writable criteria where iteration
                    measurably helps (literary translation, search refinement)
  Relative cost:    variable (scales with iteration count)
  Predictability:   medium (loop length is data-dependent)
  Bounding control: a HARD ITERATION CAP; without it the loop can run forever
  Primary failure:  the evaluator and generator oscillate without converging
  Do NOT use for:   tasks where you cannot articulate the rubric (the evaluator
                    then has nothing to check against — use a single good call)
```

### Ten task mappings

```text
TASK MAPPING 1
  Task:        Translate a document, then summarize it, then format as a brief.
  Pattern(s):  Prompt chaining
  Boundary:    none — fully fixed-path
  Rationale:   three fixed subtasks in a known order; classic chain.

TASK MAPPING 2
  Task:        Route inbound tickets to refund / technical / billing handlers.
  Pattern(s):  Routing
  Boundary:    none — the route set is enumerable
  Rationale:   distinct known categories, each to a specialized handler.

TASK MAPPING 3
  Task:        Review a code change for security, style, performance at once.
  Pattern(s):  Parallelization (sectioning)
  Boundary:    none — three fixed independent branches
  Rationale:   independent checks; concurrency + a clean context per check.

TASK MAPPING 4
  Task:        Answer an open-ended research question by deciding which sources.
  Pattern(s):  Orchestrator-workers (composition; see Composed System A)
  Boundary:    at the orchestrator's decomposition step — model-driven
  Rationale:   subtasks (which sources, which threads) are unknown until runtime.

TASK MAPPING 5
  Task:        Generate a tagline, critique vs. brand guidelines, revise to pass.
  Pattern(s):  Evaluator-optimizer
  Boundary:    inside the loop — data-dependent iteration count (capped)
  Rationale:   clear written criteria (brand guidelines) + iteration helps.

TASK MAPPING 6
  Task:        Classify an email's language, then run the matching responder.
  Pattern(s):  Routing
  Boundary:    none — fixed language set -> fixed responders
  Rationale:   classify-then-dispatch over a known set; textbook routing.

TASK MAPPING 7
  Task:        Make a coordinated edit across an unknown set of files in a repo.
  Pattern(s):  Orchestrator-workers (composition; see Composed System B)
  Boundary:    at decomposition — the file set is discovered at runtime
  Rationale:   you cannot list the files in advance; the lead must decide them.

TASK MAPPING 8
  Task:        Grade a free-text answer 3x and take the majority safety verdict.
  Pattern(s):  Parallelization (voting)
  Boundary:    none — fixed 3 runs + majority aggregation
  Rationale:   same task repeated for confidence; aggregation is code.

TASK MAPPING 9
  Task:        Draft a contract clause, then check it for prohibited terms.
  Pattern(s):  Prompt chaining (with a hard gate)
  Boundary:    none — draft -> gate; the gate can block release
  Rationale:   fixed two-step with a validation gate; the gate is the value.

TASK MAPPING 10
  Task:        Personalize an onboarding sequence from a fixed CRM schema.
  Pattern(s):  Prompt chaining
  Boundary:    none — fixed schema -> fixed sequence of templated calls
  Rationale:   bounded inputs + fixed order; personalization is template-fill.
```

### Composed System A — research question (task 4)

```text
COMPOSED SYSTEM A — "answer an open-ended research question"

  in (question)
    |
    v
  [ router ]  <-- FIXED control flow (is this a known FAQ or open research?)
    |
    +-- known FAQ ----> [ retrieval + single LLM call ] --------+
    |                                                            |
    +-- open research -> ===== model-driven boundary BELOW =====  |
                          [ orchestrator: decide which sources ]  |
                                 |          |          |          |
                                 v          v          v          |
                            [worker:    [worker:   [worker:       |
                             web]        docs]      internal]     |
                                 |          |          |          |
                                 +----------+----------+          |
                                            v                     |
                          [ orchestrator: synthesize ] -----------+
                                            |
                                            v
                                          answer

  FIXED-PATH zone:    the outer router + the FAQ branch (enumerable).
  MODEL-DRIVEN zone:  everything below the marked line in the research branch —
                      the orchestrator decides the subtasks at runtime.
  Bounding controls:  router fallback route; orchestrator FAN-OUT CAP + per-worker
                      token budget.
  Highest-risk boundary: the orchestrator's decomposition (unbounded cost if the
                      fan-out cap is missing).
```

### Composed System B — coordinated repo edit (task 7)

```text
COMPOSED SYSTEM B — "coordinated edit across an unknown set of files"

  in (change request)
    |
    v
  [ validate request + load repo index ]   <-- FIXED (hardcoded, no model)
    |
    v
  ===== model-driven boundary BELOW =====
  [ orchestrator: discover affected files, decompose into edits ]
       |              |              |
       v              v              v
  [worker: edit  [worker: edit  [worker: edit
   file X]        file Y]        file Z]
       |              |              |
       +--------------+--------------+
                      v
  ===== back to FIXED control flow =====
  [ run test suite (gate) ] -- fail --> [ orchestrator: re-decompose, retry ]
                      |
                    pass
                      v
  [ open PR ]   <-- FIXED (hardcoded trigger)

  FIXED-PATH zone:    request validation, repo-index load, the test gate, PR open.
  MODEL-DRIVEN zone:  the orchestrator's file discovery + edit decomposition.
  Bounding controls:  FAN-OUT CAP on workers; a retry cap on the test-fail loop
                      (otherwise re-decompose can loop like evaluator-optimizer).
  Highest-risk boundary: file discovery — a bad decomposition edits the wrong
                      files; the deterministic test gate is what contains it.
```

### Tradeoff table

```text
PATTERN              CONTROL FLOW              REL. COST   PREDICTABILITY   BOUNDING CONTROL
-------------------  ------------------------  ---------  ---------------  ----------------------
Prompt chaining      fixed                     low        high             inter-step gate
Routing              fixed                     low        high             default/fallback route
Parallelization      fixed                     medium     high             fixed branch count +
                                                                           aggregation rule
Orchestrator-workers model-driven decompose    high       medium           fan-out cap +
                                                                           per-worker budget
Evaluator-optimizer  looping, capped           variable   medium           hard iteration cap
```

The line splits exactly where Chapter 2 says it does: chaining, routing, and
parallelization are pure workflows (every path enumerable); orchestrator-workers
and evaluator-optimizer introduce model-driven control flow and the variance —
and the mandatory cap — that comes with it.

## Meeting the acceptance criteria

- **All five patterns have a complete entry with a diagram and a bounding
  control** — each `PATTERN ENTRY` carries a diagram, fit, cost, predictability,
  an explicit `Bounding control` field, a primary failure, and a "do NOT use for"
  anti-example.
- **All ten tasks mapped with a one-or-two-sentence rationale tied to shape** —
  every `TASK MAPPING` cites the task shape (fixed sequence, distinct categories,
  unknown-until-runtime), not "it seemed agentic."
- **At least two tasks are compositions, drawn as such** — tasks 4 and 7 are drawn
  as Composed Systems A and B.
- **The composed diagrams mark the fixed-path → model-driven boundary** — both
  diagrams carry an explicit `model-driven boundary BELOW` marker and label the
  fixed-path and model-driven zones.
- **The tradeoff table is consistent with Chapter 2** — the first three patterns
  are high-predictability workflows; orchestrator-workers and evaluator-optimizer
  carry the cost and variance.

## Common pitfalls

- **Over-patterning a task that a chain or route already covers.** Tasks 1, 9, and
  10 tempt readers toward orchestrator-workers because they "feel agentic," but
  their steps are fixed and enumerable — a chain is correct and an order of
  magnitude cheaper. Reach for the heaviest pattern only when the lighter one
  provably fails.
- **Omitting the bounding control.** An orchestrator-workers entry without a
  fan-out cap, or an evaluator-optimizer without an iteration cap, is the catalog
  entry for a cost incident. The cap is mandatory, not optional polish.
- **Mislabeling sectioning as orchestrator-workers.** Tasks 3 and 8 have a *fixed*
  set of branches known in advance — that is parallelization, not dynamic
  decomposition. Orchestrator-workers is only for subtasks unknown until runtime.
- **Not marking the boundary in compositions.** A composed diagram that does not
  show where control flow turns model-driven hides exactly the risk the exercise
  is testing. Mark it on the diagram, not just in prose.
- **Confusing voting with sectioning.** Both are parallelization, but voting runs
  the *same* task N times for confidence (task 8) while sectioning splits *different*
  subtasks (task 3). The aggregation rule differs — majority vs. merge.

## Verification

A completed submission is correct when:

- Each of the five `PATTERN ENTRY` blocks has all seven fields filled, including a
  non-empty `Bounding control`, and a concrete "do NOT use for" anti-example.
- All ten tasks are mapped, and exactly the model-driven ones (4 and 5 internally;
  4 and 7 as compositions) name a runtime boundary; the rest say "none — fixed".
- Tasks 4 and 7 appear as drawn composed diagrams, each with an explicit
  fixed-path | model-driven boundary marker.
- The tradeoff table lists chaining/routing/parallelization as `high`
  predictability with `fixed` control flow, and the other two as `medium` with
  model-driven / looping control flow — matching Chapter 2's own table.
- `NOTES.md` answers the three prompts: which task was most tempting to
  over-pattern (here task 4 or 7 if you forget the FAQ fast-path), the single
  bounding control per pattern, and the highest-risk boundary in each composed
  system (the orchestrator's decomposition in both A and B).
