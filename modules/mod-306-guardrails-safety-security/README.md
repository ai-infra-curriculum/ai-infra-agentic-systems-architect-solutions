# mod-306-guardrails-safety-security: Guardrails, Safety & Security — Solutions

Reference solutions for the Agentic Systems Architect guardrails module. Each
exercise turns "the agent can do anything" into "a hijacked agent can reach
nothing dangerous" through placement, least privilege, deterministic policy,
scoped tokens, sandboxing, and adversarial testing.

## Exercises

- [exercise-01 — Guardrail Placement Architecture](exercise-01-guardrail-placement-architecture/README.md):
  request-path diagram with input/output/per-tool-call guardrails on the right
  trust boundaries; deterministic `authorize_tool_call`.
- [exercise-02 — Prompt Injection Threat Model](exercise-02-prompt-injection-threat-model/README.md):
  STRIDE-style model of indirect injection (OWASP LLM01:2025) with each threat
  mapped to a structural control and a bounded residual risk.
- [exercise-03 — Least-Privilege Tool Permissions](exercise-03-least-privilege-tool-permissions/README.md):
  refactor broad tools into narrow ones, a risk-tiered permission matrix, and a
  tiered approval `gate` with approval-fatigue defense.
- [exercise-04 — OAuth, RBAC & Token Management](exercise-04-oauth-rbac-token-management-for-toolchains/README.md):
  per-integration OAuth flows, a leak-resistant token lifecycle, audience binding,
  and effective authority = RBAC ∩ OAuth scopes ∩ per-call policy.
- [exercise-05 — CLI Agent Sandboxing](exercise-05-cli-agent-sandboxing-local-and-remote/README.md):
  remote and local sandbox topologies, named egress allowlists, and a
  defense-in-depth argument where no single boundary is load-bearing.
- [exercise-06 — Excessive Agency Controls](exercise-06-excessive-agency-controls/README.md):
  audit and redesign for OWASP LLM06:2025 across excessive functionality,
  permissions, and autonomy, with an autonomy gate that proposes irreversible
  actions.

Each solution README carries: Approach, Reference solution (worked artifacts with
runnable Python where it fits), Meeting the acceptance criteria, Common pitfalls,
and Verification.
