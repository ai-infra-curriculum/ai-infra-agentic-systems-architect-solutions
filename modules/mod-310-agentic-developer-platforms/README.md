# mod-310-agentic-developer-platforms — Solutions

Reference solutions for the Agentic Systems Architect module **mod-310 —
Agentic Developer Platforms & SDLC Integration** (~L48). Each exercise solution
is a worked plugin/platform architecture on Claude Code's extension standards
(Agents, Skills, Hooks, MCP), with real config and code where the exercise calls
for it. Config formats are verified against the current Claude Code docs at
[code.claude.com/docs](https://code.claude.com/docs).

## Index

- [exercise-01 — Plugin Architecture on Agents, Skills, Hooks, and MCP](exercise-01-plugin-architecture-agents-skills-hooks-mcp/README.md)
  — Partition six platform requirements onto the right extension point, then
  build the artifacts: `plugin.json`, a read-only `security-reviewer` subagent,
  a `pr-conventions` skill, a protected-path `PreToolUse` hook (with the real
  guard script), and an Atlassian `.mcp.json` entry.
- [exercise-02 — AI-for-SDLC Tool-Use Patterns and Trust Boundaries](exercise-02-ai-for-sdlc-tool-use-patterns/README.md)
  — A per-stage trust-boundary spec: distinct access profiles, least-privilege
  credentials held in the integration layer, deterministic gates on irreversible
  actions, and egress governance — defeating a concrete prompt-injection probe.
- [exercise-03 — Context-Hydration Strategies for SDLC Tasks](exercise-03-context-hydration-strategies/README.md)
  — A layered, just-in-time hydration plan (L0–L3) for "debug a failing CI test,"
  budgeted against a load-everything baseline (~8k vs ~500k+ tokens), with
  exclusions and safe-degradation rules.
- [exercise-04 — DevOps and PM Toolchain Integration](exercise-04-devops-toolchain-integration/README.md)
  — Per-integration interface choices (REST / GraphQL / events) for the
  Jira → implement → PR → CI loop, with an event-driven backbone, a GraphQL query
  that collapses an N+1 cascade, and webhook robustness.
- [exercise-05 — Developer Tooling Adoption and DX](exercise-05-developer-tooling-adoption-and-dx/README.md)
  — An adoption plan for a skeptical 50-engineer org: one-command activation,
  agent-consumed skills vs. human task docs, bidirectional feedback that turns
  bad runs into eval cases, and eval-gated releases with measurable signals.

Each solution README follows the same shape: **Approach**, **Reference
solution** (with config/code), **Meeting the acceptance criteria**, **Common
pitfalls**, and **Verification**.
