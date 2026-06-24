# mod-308-deployment-durable-execution: Deployment, Durable Execution & Human-in-the-Loop — Solutions

Reference solutions for the Deployment, Durable Execution & Human-in-the-Loop module
(mod-308). These are architecture exercises, so each solution is a worked **design
artifact** — a durable/ephemeral decomposition, a HITL approval contract, a deploy
runbook, a failure-recovery matrix — rather than runnable code. The thread across all
four: a long-running agent's state must outlive the process that produced it, which
turns durable execution, in-flight-safe deploys, persisted HITL gates, and fleet
recovery into four faces of one design problem.

## Index

- [exercise-01-durable-execution-design](exercise-01-durable-execution-design/README.md)
  — the vendor-onboarding agent split into a durable workflow + idempotent activities,
  with a decomposition table, the replay/determinism rule, and an idempotency register
  covering every irreversible side effect.
- [exercise-02-hitl-approval-architecture](exercise-02-hitl-approval-architecture/README.md)
  — two HITL gates (compliance sign-off; $50k two-approver wire transfer) as durable
  interrupts: decision sets, the interrupt lifecycle, full approval contracts, and
  idempotent resume.
- [exercise-03-agent-fleet-deployment-strategy](exercise-03-agent-fleet-deployment-strategy/README.md)
  — diagnosis of why rolling/blue-green sever multi-day runs, a rainbow/never-replace
  strategy with a versioning policy, and a deploy + instant-rollback runbook.
- [exercise-04-failure-recovery-and-resumption](exercise-04-failure-recovery-and-resumption/README.md)
  — ready-step-backlog autoscaling, a worker/zone/dependency recovery matrix, the AZ and
  poison-run incident fixes, and per-run blast-radius caps.

## Using these solutions

Each exercise README follows the same shape: **Approach** (the reasoning and design
choices), **Reference solution** (the worked artifact), **Meeting the acceptance
criteria** (mapped to the learning exercise), **Common pitfalls**, and **Verification**
(how to check a submission, including the `NOTES.md` reflection prompts). Read the
learning module's exercise brief first, attempt the artifact, then compare against the
reference — the value is in the judgment (where state lives, which actions are gated,
how a run survives a process death), not in matching the exact wording.
