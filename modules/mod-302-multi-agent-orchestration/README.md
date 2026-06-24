# mod-302-multi-agent-orchestration — Solutions

Reference solutions for the Multi-Agent Orchestration Architecture module (mod-302).
These are design and architecture exercises, so each solution is a worked **model
artifact** — a topology diagram with a defended token budget, a delegation control-flow
with answer ownership, an MCP/A2A integration spec, or a coordination-and-escalation
spec — rather than runnable code.

## Index

- [exercise-01-orchestrator-worker-topology-design](exercise-01-orchestrator-worker-topology-design/README.md)
  — a flat orchestrator-worker topology for a five-provider competitive brief, with a
  token envelope that defends the `~15x` multiplier via the distilled-return ratio.
- [exercise-02-handoffs-vs-manager-as-tools](exercise-02-handoffs-vs-manager-as-tools/README.md)
  — a manager-as-tools backbone with one bounded technical handoff, explicit answer
  ownership, typed contracts, and an un-bypassable security guardrail.
- [exercise-03-mcp-a2a-integration-architecture](exercise-03-mcp-a2a-integration-architecture/README.md)
  — an incident-response integration spec: four MCP edges, one cross-team A2A edge,
  per-edge contracts, trust boundaries, and trace-ID propagation across both seams.
- [exercise-04-coordination-and-escalation-design](exercise-04-coordination-and-escalation-design/README.md)
  — a one-page coordination/escalation spec for a travel planner: orchestrator-mediated
  coordination, a deterministic conflict rule, numeric loop bounds, a five-rung ladder,
  and two fail-safe human gates.

## Using these solutions

Each exercise README follows the same shape: **Approach** (the reasoning and design
choices), **Reference solution** (the worked artifact — topology, ownership map,
integration spec, or escalation spec), **Meeting the acceptance criteria** (mapped to
the learning exercise), **Common pitfalls**, and **Verification** (how to check a
submission, including the `NOTES.md` reflection prompts). Read the learning module's
exercise brief first, attempt the artifact, then compare against the reference — the
value is in the judgment (when fan-out earns its `~15x`, who owns the answer, which seam
is MCP vs. A2A, what a tripped bound does), not in matching the exact numbers or wording.
