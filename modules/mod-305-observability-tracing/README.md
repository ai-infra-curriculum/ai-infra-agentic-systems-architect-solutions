# mod-305-observability-tracing — Solutions

Reference solutions for the Observability & Tracing module (Agentic Systems
Architect, ~L48). Each exercise solution carries an approach, a worked reference
solution with runnable code, an acceptance-criteria mapping, common pitfalls, and
verification steps.

## Exercises

- [exercise-01-otel-genai-span-design](exercise-01-otel-genai-span-design/README.md)
  — design the `SPAN_SCHEMA.md` contract and instrument an orchestrator-worker run
  to the OTel GenAI conventions (`invoke_agent` / `chat` / `execute_tool`), with
  context-propagated nesting, in-context failure spans, and retry/loop attributes.
- [exercise-02-quality-and-drift-signals](exercise-02-quality-and-drift-signals/README.md)
  — define the non-APM signal taxonomy (`SIGNALS.md`), score faithfulness
  asynchronously joined by `trace_id`, and detect a deploy-attributable quality
  regression with a justified drift alert rule.
- [exercise-03-observability-platform-evaluation](exercise-03-observability-platform-evaluation/README.md)
  — weight requirements to a specific system, run a real POC across two of
  {LangSmith, Langfuse, Arize Phoenix}, build a POC-derived weighted scoring
  matrix, and write the build-vs-buy ADR.
