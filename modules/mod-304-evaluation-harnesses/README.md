# mod-304-evaluation-harnesses: Evaluation Harnesses for Agentic Systems — Solutions

Reference solutions for the Evaluation Harnesses module (mod-304). Each solution is a
runnable harness — trajectory and final-state scorers, a layered tool-call correctness
cascade, a calibrated LLM-judge rubric, and an eval-gated release pipeline — built so
the four exercises **compose** into one deployment gate.

## Index

- [exercise-01-trajectory-eval-design](exercise-01-trajectory-eval-design/README.md)
  — a versioned dataset and independent dual-lens scorers (final-state strict,
  trajectory predicates loose), with a quadrant report that flags the
  outcome-correct-but-trajectory-unsafe cases.
- [exercise-02-tool-call-correctness-harness](exercise-02-tool-call-correctness-harness/README.md)
  — a two-layer (selection + arguments) harness whose checks run as a
  short-circuiting cascade, reporting per-layer pass rates and the judge calls the
  cascade saved.
- [exercise-03-llm-as-judge-rubric](exercise-03-llm-as-judge-rubric/README.md)
  — a pinned four-axis rubric calibrated against a human gold set, with per-axis
  agreement, a human–human ceiling, an inspect-fix-remeasure cycle, and a gateability
  verdict per axis.
- [exercise-04-eval-gated-release-pipeline](exercise-04-eval-gated-release-pipeline/README.md)
  — an offline (relative + hard) gate plus shadow, windowed canary, automated
  rollback, and the label flywheel, proven to block a regressing build and promote a
  clean one.

## Using these solutions

Each exercise README follows the same shape: **Approach** (the design reasoning),
**Reference solution** (runnable harness code plus the worked pipeline artifact),
**Meeting the acceptance criteria** (mapped to the learning exercise), **Common
pitfalls**, and **Verification** (how to check a submission, including the `NOTES.md`
reflection prompts). The Python is self-contained — exercises 01, 03, and 04 use only
the standard library; exercise 02 needs `jsonschema`. Read the learning module's
exercise brief first, build your own harness, then compare against the reference; the
value is in the judgment — which lens, which layer, which axis is gateable — not in
matching the exact numbers.
