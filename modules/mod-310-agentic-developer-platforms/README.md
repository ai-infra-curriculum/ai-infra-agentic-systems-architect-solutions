# mod-310-agentic-developer-platforms — Solutions

Reference solutions for the Agentic Systems Architect module **mod-310 —
Agentic Developer Platforms** (~L48). These are **architecture** exercises, so
each solution is a worked **design artifact** — a cross-platform extension-mapping
table with a portability/lock-in register, a scoped-access and trust-boundary
design, a toolchain integration contract, an adoption/DX plan — rather than
runnable code. Config and query snippets appear where an exercise calls for them.

The module's central discipline is **de-biased by construction**: the generic
extension model (agent / skill / tool / hook-lifecycle, with MCP as the
external-access mechanism) is host-agnostic, and host platforms — Claude Code,
Cursor, GitHub Copilot, LangGraph Platform, and a custom in-house host — are
treated as **interchangeable peers**, never with any one as canonical. Likewise,
software development is **one domain binding among others**: every design is shown
to re-bind unchanged to a non-developer domain (ops, support, analytics).

## Index

- [exercise-01-extension-model-mapping-across-platforms](exercise-01-extension-model-mapping-across-platforms/README.md)
  — Place five capabilities on the generic extension model by *kind*, then map the
  architecture across **two-plus host platforms as peers** (LangGraph, a custom
  host, Claude Code), grade portability per capability (MCP High, hook Low), build
  a lock-in/exit-cost register, and defend a deliberate split against a rejected
  single-host alternative.
- [exercise-02-secure-tool-use-and-context-hydration](exercise-02-secure-tool-use-and-context-hydration/README.md)
  — A trust-boundary design (model is *not* a boundary) for a support agent: a
  tool/privilege register scoped by target and sorted on the reversibility ladder
  with the irreversible tier gated, a four-layer budgeted hydration plan with a
  concrete distillation, and a no-fabrication safe-degradation table — re-bound to
  the SDLC agent to prove domain-independence.
- [exercise-03-workflow-toolchain-integration](exercise-03-workflow-toolchain-integration/README.md)
  — A vendor-neutral integration contract: a per-integration REST/GraphQL/event
  selection table (reactive trigger an event, never a poll), one shaped GraphQL
  read collapsing a ~13-call N+1 cascade to one round-trip, and a webhook receiver
  that survives retries, duplicates, and out-of-order delivery.
- [exercise-04-platform-adoption-and-dx-plan](exercise-04-platform-adoption-and-dx-plan/README.md)
  — An adoption plan diagnosed against the two decisive moments (activation,
  trust-under-failure) by root cause: one-step bundle activation with a real
  metric and a refused vanity metric, legibility + reversibility-by-default +
  frictionless reporting, skills that pay twice, and an eval-gated feedback loop.
- [exercise-05-non-sdlc-platform-case](exercise-05-non-sdlc-platform-case/README.md)
  — The generalization proof: restate the five-element pattern domain-free, re-bind
  it to ops/on-call (and analytics in the stretch), and show a changed-vs-constant
  table that reads **"same"** down the architecture column — verdict: re-bind, not
  rebuild.

## Using these solutions

Each exercise README follows the same shape: **Approach** (the reasoning and design
choices), **Reference solution** (the worked artifacts — mapping tables,
trust-boundary diagrams, registers, integration contracts, adoption plans, with
config/query snippets where called for), **Meeting the acceptance criteria**
(mapped to the learning exercise), **Common pitfalls**, and **Verification** (how
to check a submission, including the `NOTES.md` reflection prompts). Read the
learning module's exercise brief first, attempt the artifact, then compare against
the reference — the value is in the judgment (which extension point a capability
*is*, where the trust boundary sits, which interface an access shape demands, how
adoption is engineered), and in the de-biasing discipline: platforms are peers,
and SDLC is one domain among many.
