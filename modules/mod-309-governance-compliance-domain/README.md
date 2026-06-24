# mod-309-governance-compliance-domain — Solutions

Reference solutions for the Governance, Compliance & Domain Constraints module
(~L48). Each exercise solution is a worked **governance artifact** — specs,
mappings, matrices, diagrams, and policies, not code — built around the fictional
**TutorMesh** K-12 agentic platform.

> **Not legal or audit advice.** These are teaching references. Framework
> citations (ISO/IEC 42001:2023, NIST AI 100-1) and statutory citations (FERPA,
> COPPA) are stated for architectural mapping; verify against the primary sources
> and engage counsel for binding interpretation.

## Exercise index

- [exercise-01-iso-42001-controls-mapping](exercise-01-iso-42001-controls-mapping/README.md)
  — AIMS scope, risk-based statement of applicability, a 12-row ISO/IEC 42001
  controls-mapping table, a three-row NIST AI RMF crosswalk, and an impact
  assessment for the parent-facing progress-summary agent.
- [exercise-02-plugin-lifecycle-governance](exercise-02-plugin-lifecycle-governance/README.md)
  — the evaluate → approve → version → monitor → revoke lifecycle: manifest
  schema, three filled manifests, tiered approval, a signed registry, and a
  fleet-wide kill-switch runbook.
- [exercise-03-regulated-domain-architecture-ferpa-coppa](exercise-03-regulated-domain-architecture-ferpa-coppa/README.md)
  — FERPA/COPPA translated into a data-flow architecture: the five-question
  method, a data-flow diagram with verified absent edges, a tamper-evident audit
  design, an accountability ADR, and a cited compliance matrix.
- [exercise-04-k12-edtech-agentic-constraints](exercise-04-k12-edtech-agentic-constraints/README.md)
  — minor-safety architecture: a bounded action space, a crisis-escalation stack,
  a tiered human-in-the-loop model, a hallucination-containment pipeline, and a
  control-spec table.
- [exercise-05-governance-and-accountability-spec](exercise-05-governance-and-accountability-spec/README.md)
  — the capstone fleet governance spec: the six artifacts (registry, risk-tier
  model, RACI, risk register, control catalog, incident/change process) for the
  four-agent TutorMesh fleet, including aggregate-risk ownership.

## How to read these

Each solution follows the same shape: **Approach** (the method), **Reference
solution** (the worked artifacts), **Meeting the acceptance criteria**, **Common
pitfalls**, and **Verification**. The artifacts compose — the framework mapping
(01) and FERPA/COPPA translation (03) feed the control catalog and risk register
of the capstone (05), and the plugin lifecycle (02) and minor-safety controls
(04) appear there as fleet controls.
