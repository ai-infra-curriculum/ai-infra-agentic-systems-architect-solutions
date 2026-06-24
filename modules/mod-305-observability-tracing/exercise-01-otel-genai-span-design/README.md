# mod-305-observability-tracing/exercise-01-otel-genai-span-design — Solution

## Approach

The exercise asks for two things in order: a **design artifact** (`SPAN_SCHEMA.md`)
that fixes the contract, and then **instrumentation** that provably satisfies it.
Treating the artifact as the source of truth is the whole point — an architect
writes the span schema before any code, because the schema is what makes a
multi-agent trace navigable, and the code is just the thing that has to obey it.

The design decisions that drive this solution:

- **Instrument against the OpenTelemetry GenAI semantic conventions, not a vendor
  SDK.** That is what makes the backend swappable (the payoff exercise-03 then
  cashes in). We pin `opentelemetry-semantic-conventions` and treat the
  experimental attribute names as a contract re-verified on upgrade.
- **Let context propagation build the tree.** Every span is opened with
  `start_as_current_span`, so `chat` and `execute_tool` spans nest under the
  worker's `invoke_agent` span automatically. No span object is ever passed by
  hand. When the emitted tree diverges from the designed tree, the cause is
  always a span opened without being made current.
- **Prompts/completions are span events, not attributes**, gated by one
  content-capture flag that is a governance decision (PII), defaulting to off.
- **Failures are recorded in-context** with `Status(StatusCode.ERROR)` plus
  `record_exception`, so a red span sits inside the run that contained it.
- **Retries and loops carry explicit attributes** (`retry.attempt`,
  `agent.loop.iteration`, `agent.loop.terminated`) so a three-retry chain is
  distinguishable from three legitimate tool calls, and a capped loop is
  distinguishable from a clean finish.

## Reference solution

### The span-schema design artifact (`SPAN_SCHEMA.md`)

This is the deliverable that comes first. It names every span type, its
convention attributes, its parent, and the attribute-vs-event-vs-dropped
decision. Targeted convention version: **`opentelemetry-semantic-conventions`
1.29.0** (GenAI conventions experimental; `invoke_agent` and `execute_tool`
operation names present, content-capture gated by
`OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT`).

```text
SPAN SCHEMA — research-agent (orchestrator + 2 workers + tools)
Convention: OTel GenAI semconv 1.29.0 (experimental — re-verify on upgrade)

────────────────────────────────────────────────────────────────────────
span: invoke_agent {agent_name}            operation = invoke_agent
  parent: (orchestrator) = root of trace
          (worker)       = orchestrator's invoke_agent span
  required attributes:
    gen_ai.operation.name      = "invoke_agent"
    gen_ai.agent.name          = "orchestrator" | "research" | "numbers"
  run attributes (set at close):
    gen_ai.usage.input_tokens  = sum of child chat input tokens
    gen_ai.usage.output_tokens = sum of child chat output tokens
  loop attributes (worker only):
    agent.loop.iteration       = 0..N   (current iteration)
    agent.loop.terminated      = "final" | "max_iterations"
────────────────────────────────────────────────────────────────────────
span: chat {model}                         operation = chat
  parent: the enclosing invoke_agent span (current span)
  required attributes:
    gen_ai.operation.name      = "chat"
    gen_ai.system              = "openai" | "anthropic" | ...
    gen_ai.request.model       = "gpt-4o-2024-08-06"   (EXACT model)
    gen_ai.usage.input_tokens  = int
    gen_ai.usage.output_tokens = int
    gen_ai.response.finish_reasons = ["stop"] | ["length"] | ["tool_calls"]
  retry attribute (on retried calls only):
    retry.attempt              = 1..K
  EVENTS (captured only when content-capture flag is ON):
    gen_ai.user.message        prompt content
    gen_ai.choice              completion content
────────────────────────────────────────────────────────────────────────
span: execute_tool {tool_name}             operation = execute_tool
  parent: the enclosing invoke_agent span (current span)
  required attributes:
    gen_ai.operation.name      = "execute_tool"
    gen_ai.tool.name           = "web_search" | "calculator"
  EVENTS (captured only when content-capture flag is ON):
    gen_ai.tool.message        sanitized args / result
────────────────────────────────────────────────────────────────────────

DATA CLASSIFICATION
  attribute  : structured, low-cardinality, always safe to keep
               (operation name, model, token counts, tool name, finish reason)
  event      : large or PII-bearing (prompts, completions, tool args/results)
               → gated behind ONE content-capture flag; default OFF
  dropped    : raw user identifiers (only hashed user.id is ever stored);
               full retrieved documents (store a hash/uri, not the bytes)

CONTENT-CAPTURE RATIONALE
  Prompts and completions routinely carry user PII. Capture is a governance
  flag, not a debug toggle. Default OFF in prod; ON in dev / for a sampled,
  access-controlled subset. Flipping it is a policy decision (mod-309), so it
  lives in config, never hard-coded true.
```

### Process-level setup

`telemetry.py` — set up the `TracerProvider`, `Resource`, exporter, and the
content-capture flag exactly once per process.

```python
"""OpenTelemetry GenAI bootstrap. Imported once at process start."""
from __future__ import annotations

import os

from opentelemetry import trace
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import (
    BatchSpanProcessor,
    ConsoleSpanExporter,
)

# Pin semconv attribute keys as constants so an upgrade is a one-file diff.
GEN_AI = "gen_ai"
OP_NAME = f"{GEN_AI}.operation.name"
AGENT_NAME = f"{GEN_AI}.agent.name"
SYSTEM = f"{GEN_AI}.system"
REQUEST_MODEL = f"{GEN_AI}.request.model"
INPUT_TOKENS = f"{GEN_AI}.usage.input_tokens"
OUTPUT_TOKENS = f"{GEN_AI}.usage.output_tokens"
FINISH_REASONS = f"{GEN_AI}.response.finish_reasons"
TOOL_NAME = f"{GEN_AI}.tool.name"

# Governance flag: capturing prompt/completion content is OFF unless asked.
CAPTURE_CONTENT = (
    os.getenv("OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT", "false")
    .lower()
    == "true"
)


def _build_exporter():
    """OTLP when an endpoint is configured; console otherwise (zero-dep start)."""
    endpoint = os.getenv("OTEL_EXPORTER_OTLP_ENDPOINT")
    if endpoint:
        from opentelemetry.exporter.otlp.proto.http.trace_exporter import (
            OTLPSpanExporter,
        )

        return OTLPSpanExporter(endpoint=f"{endpoint.rstrip('/')}/v1/traces")
    return ConsoleSpanExporter()


def init_tracing(version: str = "2026.06.0", environment: str = "dev"):
    """Idempotent-enough bootstrap: call once at startup, return the tracer."""
    resource = Resource.create(
        {
            "service.name": "research-agent",
            "service.version": version,
            "deployment.environment": environment,
        }
    )
    provider = TracerProvider(resource=resource)
    provider.add_span_processor(BatchSpanProcessor(_build_exporter()))
    trace.set_tracer_provider(provider)
    return trace.get_tracer("agent.runtime")
```

### Instrumenting the run to match the schema

`agent.py` — a runnable orchestrator-worker system using a deterministic fake
model and fake tools, so the trace is reproducible without a provider key. Swap
`FakeModel`/`FakeTool` for real clients and the instrumentation is unchanged.

```python
"""Runnable orchestrator + two workers + tools, instrumented to SPAN_SCHEMA.md."""
from __future__ import annotations

import asyncio
import random
from dataclasses import dataclass

from opentelemetry import trace as _trace
from opentelemetry.trace import Status, StatusCode

import telemetry as T

tracer = T.init_tracing(version="2026.06.0", environment="dev")
MAX_ITERATIONS = 6


@dataclass
class ModelResponse:
    text: str
    input_tokens: int
    output_tokens: int
    finish_reason: str


class FakeModel:
    """Deterministic stand-in for a provider client."""

    system = "openai"
    model = "gpt-4o-2024-08-06"

    def __init__(self, fail_first_n: int = 0) -> None:
        self._calls = 0
        self._fail_first_n = fail_first_n

    async def chat(self, prompt: str) -> ModelResponse:
        self._calls += 1
        if self._calls <= self._fail_first_n:
            raise RuntimeError("429 rate_limited")
        await asyncio.sleep(0)
        return ModelResponse(
            text=f"answer({prompt[:24]})",
            input_tokens=40 + len(prompt) % 30,
            output_tokens=18,
            finish_reason="stop",
        )


async def call_model(model: FakeModel, prompt: str, attempt: int | None = None):
    """Emit one `chat` span. `attempt` is set only on retried calls. A failure is
    recorded on THIS span (while it is still open), then re-raised — so the red
    span is the chat attempt, not the enclosing worker."""
    with tracer.start_as_current_span(f"chat {model.model}") as span:
        span.set_attribute(T.OP_NAME, "chat")
        span.set_attribute(T.SYSTEM, model.system)
        span.set_attribute(T.REQUEST_MODEL, model.model)
        if attempt is not None:
            span.set_attribute("retry.attempt", attempt)
        if T.CAPTURE_CONTENT:
            span.add_event("gen_ai.user.message", {"content": prompt})
        try:
            resp = await model.chat(prompt)
        except Exception as exc:  # record on the live chat span, then re-raise
            span.set_status(Status(StatusCode.ERROR, str(exc)))
            span.record_exception(exc)
            raise
        span.set_attribute(T.INPUT_TOKENS, resp.input_tokens)
        span.set_attribute(T.OUTPUT_TOKENS, resp.output_tokens)
        span.set_attribute(T.FINISH_REASONS, [resp.finish_reason])
        if T.CAPTURE_CONTENT:
            span.add_event("gen_ai.choice", {"content": resp.text})
        return resp


async def call_model_with_retry(model: FakeModel, prompt: str, max_attempts: int = 3):
    """Retry chain: each attempt is its own `chat` span carrying retry.attempt.
    The failing attempts are already recorded red by `call_model`; here we just
    retry, so a succeeding final attempt leaves the worker span clean."""
    last_exc: Exception | None = None
    for attempt in range(1, max_attempts + 1):
        try:
            return await call_model(model, prompt, attempt=attempt)
        except Exception as exc:  # transient: the chat span is already red; retry
            last_exc = exc
    raise last_exc  # exhausted retries


async def execute_tool(name: str, args: dict) -> str:
    """Emit one `execute_tool` span."""
    with tracer.start_as_current_span(f"execute_tool {name}") as span:
        span.set_attribute(T.OP_NAME, "execute_tool")
        span.set_attribute(T.TOOL_NAME, name)
        if T.CAPTURE_CONTENT:
            span.add_event("gen_ai.tool.message", {"arguments": str(args)})
        await asyncio.sleep(0)
        if name == "flaky_search":
            raise RuntimeError("upstream search 503")
        return f"{name}:result"


async def run_worker(role: str, instruction: str, *, model: FakeModel, plan):
    """Open an invoke_agent span; nested chat/tool spans attach automatically."""
    with tracer.start_as_current_span(f"invoke_agent {role}") as span:
        span.set_attribute(T.OP_NAME, "invoke_agent")
        span.set_attribute(T.AGENT_NAME, role)
        in_tok = out_tok = 0
        terminated = "max_iterations"
        try:
            for i in range(MAX_ITERATIONS):
                span.set_attribute("agent.loop.iteration", i)
                step = plan[i] if i < len(plan) else ("final", None)
                kind, payload = step
                if kind == "tool":
                    await execute_tool(payload, {"q": instruction})
                elif kind == "retry_chat":
                    resp = await call_model_with_retry(model, instruction)
                    in_tok += resp.input_tokens
                    out_tok += resp.output_tokens
                elif kind == "chat":
                    resp = await call_model(model, instruction)
                    in_tok += resp.input_tokens
                    out_tok += resp.output_tokens
                if kind == "final" or (i < len(plan) and plan[i][0] == "final"):
                    terminated = "final"
                    break
            span.set_attribute(T.INPUT_TOKENS, in_tok)
            span.set_attribute(T.OUTPUT_TOKENS, out_tok)
            span.set_attribute("agent.loop.terminated", terminated)
            if terminated == "max_iterations":
                span.set_status(Status(StatusCode.ERROR, "loop hit iteration cap"))
        except Exception as exc:  # worker failed: red span, in-context
            span.set_attribute("agent.loop.terminated", "error")
            span.set_status(Status(StatusCode.ERROR, str(exc)))
            span.record_exception(exc)
            raise


async def run_orchestrator() -> None:
    """Root invoke_agent span; both workers nest under it in one trace."""
    with tracer.start_as_current_span("invoke_agent orchestrator") as root:
        root.set_attribute(T.OP_NAME, "invoke_agent")
        root.set_attribute(T.AGENT_NAME, "orchestrator")

        # Worker A — healthy: a retry chain (3 chat spans, one OK) + a final chat.
        await run_worker(
            "research",
            "summarize the q2 report",
            model=FakeModel(fail_first_n=2),  # 2 failures then success
            plan=[("retry_chat", None), ("tool", "web_search"), ("final", None)],
        )

        # Worker B — fails: a tool 503 surfaces as a red span inside the run.
        try:
            await run_worker(
                "numbers",
                "compute the variance",
                model=FakeModel(),
                plan=[("tool", "calculator"), ("tool", "flaky_search")],
            )
        except Exception:
            pass  # orchestrator survives the worker failure; trace shows it red


if __name__ == "__main__":
    random.seed(7)
    asyncio.run(run_orchestrator())
    # Flush before exit so the console/OTLP exporter emits the batch.
    _trace.get_tracer_provider().shutdown()
```

Running `python agent.py` (with no OTLP endpoint set) prints the full span tree
to the console exporter. The research worker shows a `chat` retry chain whose
first two spans are red (`retry.attempt = 1, 2`, status ERROR) and the third
green; the numbers worker shows a red `execute_tool flaky_search` span nested
under its `invoke_agent`, with the worker span itself red and recording the
exception — all under one `invoke_agent orchestrator` root, one `trace_id`.

### Demonstrating the distinctions the acceptance criteria require

- **Retry chain vs. legitimate repeats.** The research worker's
  `("retry_chat", None)` step produces three `chat {model}` spans sharing the
  same logical step, each with `retry.attempt = 1/2/3`; the numbers worker's two
  `("tool", ...)` steps produce two *different* `execute_tool` spans with no
  `retry.attempt`. The presence/absence of `retry.attempt` is what tells them
  apart in the tree.
- **Capped loop vs. clean finish.** Give a worker a plan with no `("final", ...)`
  step and it exits the `for` via exhaustion; the `terminated` attribute is set
  to `"max_iterations"` and the span goes red — a latent runaway, visibly
  distinct from a `"final"` green close.
- **Sessions (stretch).** Add `session.id` / `gen_ai.conversation.id` on the
  root span and re-run with the same id for two turns; both traces group under
  one session in the viewer.

## Meeting the acceptance criteria

- **`SPAN_SCHEMA.md` exists and is complete** — the artifact above names every
  span type (`invoke_agent`, `chat`, `execute_tool`), its convention attributes,
  its parent/hierarchy, and the attribute-vs-event-vs-dropped classification with
  an explicit content-capture/PII rationale, citing semconv 1.29.0.
- **One trace, correct hierarchy** — `run_orchestrator` opens the root
  `invoke_agent orchestrator` span; `run_worker` opens child `invoke_agent`
  spans; all are opened with `start_as_current_span`, so `chat`/`execute_tool`
  nest via context propagation, never by passing span objects.
- **Required attributes present** — `chat` spans carry `gen_ai.request.model`
  and both `gen_ai.usage.*` token attributes; `execute_tool` spans carry
  `gen_ai.tool.name`.
- **Failure is red and in-context** — the `flaky_search` 503 sets
  `Status(StatusCode.ERROR)` and calls `record_exception` on its own span and on
  the enclosing worker span, which renders red inside the surrounding run.
- **Retry distinguishable; capped loop labeled** — `retry.attempt` separates a
  retry chain from repeated legitimate tool calls, and `agent.loop.terminated =
  max_iterations` flags a loop that hit its cap.

## Common pitfalls

- **Opening a span without making it current.** Using `start_span` instead of
  `start_as_current_span` (or starting it outside the `with`) breaks context
  propagation: the children attach to the wrong parent or become orphan roots, so
  the *emitted* tree diverges from the *designed* one. This is the single most
  common first-run bug, and exactly what the reflection question targets.
- **Forgetting to flush before exit.** `BatchSpanProcessor` buffers; a short
  script that exits before the batch flushes drops the whole trace. Call
  `provider.shutdown()` (or `force_flush()`) at the end.
- **Putting prompts/completions in attributes.** Large content as attributes
  bloats every span and silently ships PII to the backend. Content goes in
  **events**, gated behind the content-capture flag, never an attribute.
- **Hard-coding the model name on the `Resource` instead of the `chat` span.**
  The exact model is the axis you compare across deploys; it must be on each
  `chat` span (`gen_ai.request.model`), not stamped once on the resource.
- **Treating semconv keys as stable.** They are experimental and have changed
  across releases. Pin the version and centralize the keys as constants
  (`telemetry.py`) so an upgrade is one diff, not a scavenger hunt.

## Verification

```bash
# 1. Install pinned deps (semconv version is part of the contract).
pip install \
  "opentelemetry-sdk==1.29.0" \
  "opentelemetry-exporter-otlp-proto-http==1.29.0" \
  "opentelemetry-semantic-conventions==0.50b0"

# 2. Run with the console exporter (no backend, no provider key needed).
python agent.py

# 3. Inspect the printed spans: confirm one trace_id across all spans, the
#    invoke_agent orchestrator root, child invoke_agent workers, nested
#    chat/execute_tool spans, retry.attempt on the chat chain, and an
#    ERROR status + exception event on the flaky_search span.
python agent.py | grep -E "name|trace_id|status_code|retry.attempt|terminated"

# 4. Optional: point at a local backend and confirm the tree renders identically
#    WITHOUT changing instrumentation (proves the convention bought portability).
docker run -p 6006:6006 -p 4318:4318 arizephoenix/phoenix:latest
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318 python agent.py
# open http://localhost:6006 → the trace tree matches SPAN_SCHEMA.md

# 5. Verify content capture is governed: default run emits NO message events;
#    only the explicit opt-in does.
python agent.py | grep -c "gen_ai.user.message"          # → 0 (default OFF)
OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true \
  python agent.py | grep -c "gen_ai.user.message"        # → > 0 (opt-in)
```
