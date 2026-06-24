# mod-301-agentic-systems-foundations/exercise-04-reference-architecture-teardown — Solution

## Approach

The teardown subject is Anthropic's published *How we built our multi-agent
research system*, chosen because the authors document the architecture, the token
economics, and the failure modes in enough detail to run a real "start simple"
critique against their own evidence rather than against speculation.

The method reuses the three earlier exercises as instruments:

- **Exercise-02's catalog** names the patterns: the system is fundamentally
  orchestrator-workers (a lead agent decomposes a query and dispatches subagents),
  with a chain on the tail (a separate citation pass) and parallelization in how
  subagents run concurrently.
- **Exercise-03's boundary vocabulary** maps the major capabilities to
  hardcoded / tool / sub-agent and, crucially, reads the authors' *stated*
  context-economics — why subagents return distilled results to protect the lead's
  context, and the token multiplier they report.
- **Exercise-04's own job** is the Chapter 4 upgrade gate: for each significant
  complexity choice, ask what failed simpler, whether the complexity demonstrably
  fixed it, what it cost, and whether it was worth it — using the authors' numbers
  where they give them and flagging where they do not.

The honest output includes one choice judged *under-justified by the published
evidence* (the depth/breadth of the subagent fan-out is asserted more than proven
on a metric), and one defensible alternative (a routing workflow with single-agent
escalation for the hard tail) with the conditions under which it would win.

## Reference solution

```text
TEARDOWN: Anthropic multi-agent research system
          https://www.anthropic.com/engineering/multi-agent-research-system

  1. ARCHITECTURE SKETCH

     user query
        |
        v
     [ lead agent ]  analyzes query, develops strategy   <-- MODEL-DRIVEN
        |            (decides HOW MANY subagents + their tasks)
        |
        |  ===== fixed-path | model-driven boundary =====
        |  the lead's decomposition is model-driven; the
        |  scaffold that spawns/collects subagents is code.
        |
        +--> [ subagent 1 ]  own context, own tools, parallel
        +--> [ subagent 2 ]  searches the web, evaluates results,
        +--> [ subagent 3 ]  returns DISTILLED findings to the lead
              ... (count chosen at runtime by the lead) ...
        |
        v
     [ lead agent ]  synthesizes subagent results          <-- MODEL-DRIVEN
        |
        v
     [ citation agent ]  maps claims -> sources (separate pass)  <-- CHAIN tail
        |
        v
     final cited report

     Memory: the lead persists its plan to memory so a long run
     survives context-window limits (a hardcoded persistence step
     around a model-driven plan).

  2. PATTERNS IN USE

     - Orchestrator-workers at the core — lead decomposes the query and
       dispatches subagents at runtime.
       Evidence: the authors describe a "lead agent" that "analyzes the query,
       develops a strategy, and spawns subagents to explore different aspects
       simultaneously."
     - Parallelization within the worker tier — subagents run concurrently,
       which the authors credit with major reductions in research time.
       Evidence: they report parallel tool calls / concurrent subagents cutting
       wall-clock time substantially on breadth-first queries.
     - Prompt chaining on the tail — a dedicated citation pass runs after
       synthesis as a fixed final step.
       Evidence: a separate citation agent is described as ensuring claims map
       to their sources.

     Composition: a model-driven orchestrator-workers core with a fixed-path
     citation chain appended; parallelization is how the worker tier executes.

     Chose NOT to: use a single agent for everything.
       Stated reason: a single agent hit context-window and breadth limits on
       open-ended research; multi-agent let separate context windows explore in
       parallel. The authors are explicit that multi-agent shines for
       breadth-first queries that exceed a single context window.

  3. BOUNDARY MAP

     Capability                  Boundary        Context economics noted by authors
     --------------------------  --------------  --------------------------------------
     Query decomposition / plan  model (lead)    lead owns control flow; plan persisted
                                                 to memory to survive context limits
     Spawn + collect subagents   hardcoded       orchestration scaffold is code around
                                 scaffold         the model's decomposition decision
     Per-subagent research       sub-agent       SEPARATE context window per subagent;
                                                 each returns a DISTILLED result so raw
                                                 search content never floods the lead
     Web search / tool calls     tool            subagents decide whether/what to search
     Synthesis                   model (lead)    lead reasons over distilled returns only
     Citation mapping            chain step      fixed final pass, not the lead's job

     Reported economics: the authors note multi-agent systems burn FAR more
     tokens than chat — on the order of ~15x the tokens of a single chat
     interaction — and that the approach only pays off when task value is high
     enough to justify that spend. That token multiplier is the central cost the
     "start simple" critique must weigh.

  4. START-SIMPLE CRITIQUE (per major complexity choice)

     Choice: Multi-agent (orchestrator-workers) instead of a single agent.
       What failed simpler:   a single agent could not hold enough context or
                              pursue enough parallel threads for breadth-first
                              open-ended research (author-stated).
       Demonstrably fixed?:   yes — authors report a large eval improvement for
                              the multi-agent system over the single-agent
                              baseline on their internal research eval.
       Cost accepted:         ~15x token spend vs. a chat; added orchestration
                              and failure surface; harder debugging of
                              non-reproducible trajectories.
       Verdict:               JUSTIFIED — failure named, fix measured, cost
                              quantified, and gated on high task value.

     Choice: Subagents return distilled results (context isolation), not raw.
       What failed simpler:   raw search content would flood the lead's context.
       Demonstrably fixed?:   yes — isolation keeps the lead's context budget for
                              synthesis; this is the textbook Chapter 3 sub-agent
                              justification and the authors lean on it directly.
       Cost accepted:         distillation can drop detail the lead needed.
       Verdict:               JUSTIFIED — the isolation is real and load-bearing.

     Choice: The lead's chosen subagent count / depth ("effort scaling").
       What failed simpler:   not crisply stated as a measured failure of a fixed
                              fan-out; the authors describe teaching the lead to
                              scale effort to query complexity, but the exact
                              count is a model judgment.
       Demonstrably fixed?:   partially — they show prompt guidance helped avoid
                              over- and under-spawning, but the optimal fan-out is
                              asserted via heuristics more than proven on a curve.
       Cost accepted:         unbounded fan-out is the dominant cost risk; a bad
                              decomposition multiplies token spend.
       Verdict:               UNDER-JUSTIFIED on published evidence. What would
                              settle it: a cost/quality curve across fan-out caps
                              showing the chosen scaling sits at the knee.

  5. ALTERNATIVE ARCHITECTURE

     Routing workflow with single-agent escalation for the hard tail:

       query
         |
         v
       [ router ]  <-- FIXED: is this a narrow lookup or breadth-first research?
         |
         +-- narrow lookup ---> [ single agent + retrieval ]  (rung 2-3)
         |
         +-- breadth-first ---> [ orchestrator-workers ]      (rung 4, today's system)

     Conditions under which the alternative WINS:
       - A large fraction of real traffic is narrow lookups that a single agent
         answers well — then routing those away from the 15x-token path saves
         most of the cost while keeping multi-agent for the genuine tail.
       - Request volume is high and per-request value is modest, so the blanket
         15x multiplier is not affordable across all traffic.
     Conditions under which the CURRENT system wins:
       - Most traffic is genuinely breadth-first and high-value, so the router
         would send nearly everything down the multi-agent path anyway and only
         adds a classification hop.
```

## Meeting the acceptance criteria

- **Architecture reconstructed as a diagram with the fixed-path/model-driven
  boundary marked** — section 1 sketches lead, subagents, citation tail, and memory,
  with the `fixed-path | model-driven boundary` called out at the lead's
  decomposition.
- **Every pattern present named and located with cited evidence** — section 2 names
  orchestrator-workers, parallelization, and the citation chain, each with a
  paraphrased pointer to the source's own description.
- **Boundary map covers major capabilities and references stated
  context-economics** — section 3 maps six capabilities and records the distilled-
  return rationale and the ~15x token multiplier the authors report.
- **Each major complexity choice run through the Chapter 4 upgrade gate, with at
  least one judged justified or under-justified on evidence** — section 4 runs three
  choices through the four gate questions; multi-agent and distillation are
  JUSTIFIED, the fan-out/effort-scaling choice is flagged UNDER-JUSTIFIED with the
  evidence that would settle it.
- **A defensible alternative proposed with the conditions under which it wins** —
  section 5 gives the routing-with-escalation alternative and states both when it
  wins and when the current system wins.

## Common pitfalls

- **Reading prose as a flat agent and missing the composition.** The system is not
  "one big agent" — it is orchestrator-workers with a *fixed* citation chain on the
  tail and parallel execution in the worker tier. Naming only the headline pattern
  fails the "name every pattern present" criterion.
- **Accepting every complexity choice because the source is authoritative.** The
  teardown's value is the critique. A teardown that rubber-stamps all of
  Anthropic's choices has not done the Chapter 4 work; at least one choice must be
  tested against the *published evidence*, and where the evidence is thin, say so.
- **Ignoring the token economics.** The ~15x multiplier is the single most
  important number for the "worth it at your volume?" gate. A critique that omits it
  cannot judge whether the complexity was economically justified.
- **Confusing distilled returns with a tool call.** Subagents have their own context
  windows — that is the sub-agent boundary (isolation), the reason the lead's
  context survives. Mislabeling it a tool erases the central context-economics
  point.
- **Proposing an alternative with no conditions.** "It could have been simpler" is
  not a critique. The alternative must state the traffic mix / volume / value
  conditions under which it actually beats the shipped design — and concede when it
  does not.

## Verification

A completed submission is correct when:

- The architecture sketch shows the lead, a runtime-chosen set of subagents, the
  synthesis step, and the citation tail, with the fixed-path/model-driven boundary
  explicitly marked at the decomposition.
- Each named pattern (orchestrator-workers, parallelization, citation chain) is
  tied to a specific claim or figure from the source, not asserted bare.
- The boundary map references the source's stated economics — at minimum the
  distilled-return rationale and the token multiplier (~15x chat).
- Every major complexity choice is run through all four gate questions, and at
  least one choice carries a JUSTIFIED or UNDER-JUSTIFIED verdict backed by the
  authors' own evidence (or its absence).
- The alternative architecture states the conditions under which it wins *and* the
  conditions under which the shipped system wins.
- `NOTES.md` answers the three prompts: the decision you would most challenge in
  review and the evidence you would demand (here the fan-out/effort-scaling curve),
  where the reported costs changed your read (the 15x multiplier), and what you
  would cut first at 100x volume (route narrow queries away from the multi-agent
  path).
