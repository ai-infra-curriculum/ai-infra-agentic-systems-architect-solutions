# mod-304-evaluation-harnesses/exercise-02-tool-call-correctness-harness — Solution

## Approach

The exercise asks for a harness that reports *which layer* failed for every tool call
and runs the checks as a **cost-controlling cascade**. The whole value is in the
decomposition from Chapter 2: a call can fail at **selection** (wrong tool) or at one
of three **argument** tiers (schema → values → faithfulness), and each failure points
at a different fix. A blended "tool accuracy: 82%" hides the one number that tells you
what to edit.

Design choices that drive the reference implementation:

- **Every verdict carries a `layer` tag.** `selection`, `arguments_schema`,
  `arguments_values`, `arguments_faithfulness`. The aggregator groups by tag, so the
  output is a per-layer pass-rate breakdown, not a single rate.
- **The cascade short-circuits cheapest-first.** Schema validity (near-free
  `jsonschema`) gates value constraints, which gate the judge. A judge call **never**
  runs on a call that already failed a deterministic check, and the harness *counts*
  the judge calls saved so you can see the cost win.
- **Three deterministic tools with a constrained field each.** `lookup_order`
  (bounded `order_id`), `list_orders` (an enum `status` plus a `limit <= 100` bound and
  a `start <= end` cross-field rule), and `send_email` (a `recipients` array whose IDs
  must be *consistent with the task input*). The constrained fields are what let the
  value tier catch schema-valid-but-wrong calls.
- **The judge is stubbed, not mocked away.** A `DeterministicJudge` returns a graded
  faithfulness verdict from a recorded answer key, so the harness is fully runnable
  with no API key, while the *interface* (`score_arguments(task, tool, args) ->
  Verdict`) is exactly what a real LLM judge from exercise-03 plugs into.

The load-bearing demonstration is a single trace where one call picks a **forbidden**
tool (selection failure), one call is **schema-invalid** (missing required field), one
is **schema-valid but value-wrong** (a `limit` of 5000), and one needs the **judge**
(recipients extracted from "the two Phoenix managers"). Each produces a *different*
layer verdict, and the schema/value failures are disposed of before any judge call.

## Reference solution

### Tool schemas (`tools.py`)

```python
"""Tool parameter schemas (JSON Schema, Draft 2020-12) and task-class policy."""

TOOL_SCHEMAS: dict[str, dict] = {
    "lookup_order": {
        "type": "object",
        "properties": {"order_id": {"type": "integer", "minimum": 1}},
        "required": ["order_id"],
        "additionalProperties": False,
    },
    "list_orders": {
        "type": "object",
        "properties": {
            "status": {"type": "string", "enum": ["open", "shipped", "refunded"]},
            "limit": {"type": "integer", "minimum": 1},
            "start": {"type": "integer"},
            "end": {"type": "integer"},
        },
        "required": ["status", "limit"],
        "additionalProperties": False,
    },
    "send_email": {
        "type": "object",
        "properties": {
            "recipients": {
                "type": "array",
                "items": {"type": "string"},
                "minItems": 1,
            },
            "body": {"type": "string"},
        },
        "required": ["recipients", "body"],
        "additionalProperties": False,
    },
    "delete_record": {
        "type": "object",
        "properties": {"record_id": {"type": "integer"}},
        "required": ["record_id"],
        "additionalProperties": False,
    },
}

# Per-task-class selection policy.
TASK_POLICY = {
    "read_only_status": {
        "required": {"lookup_order"},
        "forbidden": {"send_email", "delete_record"},
        "max_calls_per_tool": 2,
    },
    "notify_managers": {
        "required": {"list_orders", "send_email"},
        "forbidden": {"delete_record"},
        "max_calls_per_tool": 2,
    },
}

# Hard value bound the schema cannot express on its own.
LIMIT_MAX = 100
```

### The harness (`tool_harness.py`)

```python
"""Two-layer tool-call correctness harness with a short-circuiting cascade."""
from __future__ import annotations

from dataclasses import dataclass, field

from jsonschema import Draft202012Validator

from tools import LIMIT_MAX, TASK_POLICY, TOOL_SCHEMAS


@dataclass
class Verdict:
    layer: str
    passed: bool
    reasons: list[str] = field(default_factory=list)


@dataclass
class ToolCall:
    tool: str
    args: dict
    needs_faithfulness_judge: bool = False


@dataclass
class Task:
    task_id: str
    task_class: str
    text: str
    # Ground-truth IDs the task refers to, for the task-consistency check.
    expected_recipient_ids: set[str] = field(default_factory=set)


# --------------------------------------------------------------------------- #
# 1. Selection layer (deterministic)
# --------------------------------------------------------------------------- #
def check_selection(called_tools: list[str], task_class: str) -> Verdict:
    policy = TASK_POLICY[task_class]
    reasons: list[str] = []
    called = set(called_tools)

    if hit := called & policy["forbidden"]:
        reasons.append(f"called forbidden tool(s): {sorted(hit)}")
    if missing := policy["required"] - called:
        reasons.append(f"missing required tool(s): {sorted(missing)}")
    cap = policy["max_calls_per_tool"]
    for tool in called:
        n = called_tools.count(tool)
        if n > cap:
            reasons.append(f"{tool} called {n}x (max {cap})")

    return Verdict("selection", not reasons, reasons)


# --------------------------------------------------------------------------- #
# 2a. Argument schema layer (deterministic, always run)
# --------------------------------------------------------------------------- #
def check_arguments_schema(args: dict, schema: dict) -> Verdict:
    errors = sorted(
        Draft202012Validator(schema).iter_errors(args),
        key=lambda e: list(e.path),
    )
    reasons = [
        f"{'/'.join(map(str, e.path)) or '<root>'}: {e.message}" for e in errors
    ]
    return Verdict("arguments_schema", not reasons, reasons)


# --------------------------------------------------------------------------- #
# 2b. Argument value layer (deterministic predicates)
# --------------------------------------------------------------------------- #
def check_arguments_values(call: ToolCall, task: Task) -> Verdict:
    reasons: list[str] = []
    args = call.args

    # Bound: limit must not exceed the operational max.
    if "limit" in args and args["limit"] > LIMIT_MAX:
        reasons.append(f"limit {args['limit']} exceeds max {LIMIT_MAX}")

    # Cross-field invariant: start <= end when both present.
    if "start" in args and "end" in args and args["start"] > args["end"]:
        reasons.append(f"start {args['start']} > end {args['end']}")

    # Task-consistency: recipient IDs must match the IDs the task referenced.
    if call.tool == "send_email" and task.expected_recipient_ids:
        got = set(args.get("recipients", []))
        if got != task.expected_recipient_ids:
            reasons.append(
                f"recipients {sorted(got)} != task IDs "
                f"{sorted(task.expected_recipient_ids)}"
            )

    return Verdict("arguments_values", not reasons, reasons)


# --------------------------------------------------------------------------- #
# 2c. Argument faithfulness (LLM judge — interface only here)
# --------------------------------------------------------------------------- #
class DeterministicJudge:
    """Stand-in for an exercise-03 LLM judge. Returns a graded faithfulness
    verdict from a recorded answer key so the harness runs without an API key.
    Swap for a real judge by matching the score_arguments signature."""

    def __init__(self, answer_key: dict[str, bool]) -> None:
        self._key = answer_key
        self.calls = 0

    def score_arguments(self, task: Task, tool: str, args: dict) -> Verdict:
        self.calls += 1
        key = f"{task.task_id}:{tool}"
        ok = self._key.get(key, False)
        reason = [] if ok else ["arguments not a faithful reading of the task"]
        return Verdict("arguments_faithfulness", ok, reason)


# --------------------------------------------------------------------------- #
# Cascade: cheapest-first, short-circuit before the judge
# --------------------------------------------------------------------------- #
def evaluate_tool_call(
    call: ToolCall, task: Task, judge: DeterministicJudge | None = None
) -> dict[str, Verdict]:
    verdicts: dict[str, Verdict] = {}

    schema_v = check_arguments_schema(call.args, TOOL_SCHEMAS[call.tool])
    verdicts["arguments_schema"] = schema_v
    if not schema_v.passed:
        return verdicts  # malformed — no point checking values or judging

    value_v = check_arguments_values(call, task)
    verdicts["arguments_values"] = value_v
    if not value_v.passed:
        return verdicts  # value-wrong — judge would add cost, not signal

    if judge is not None and call.needs_faithfulness_judge:
        verdicts["arguments_faithfulness"] = judge.score_arguments(
            task=task, tool=call.tool, args=call.args
        )
    return verdicts


# --------------------------------------------------------------------------- #
# Suite aggregation: per-layer pass rates + judge calls saved
# --------------------------------------------------------------------------- #
def run_suite(
    cases: list[tuple[Task, list[ToolCall]]], judge: DeterministicJudge
) -> dict:
    layers = [
        "selection",
        "arguments_schema",
        "arguments_values",
        "arguments_faithfulness",
    ]
    totals = {layer: [0, 0] for layer in layers}  # [passed, total]
    judge_eligible = 0

    for task, calls in cases:
        sel = check_selection([c.tool for c in calls], task.task_class)
        totals["selection"][1] += 1
        totals["selection"][0] += int(sel.passed)

        for call in calls:
            if call.needs_faithfulness_judge:
                judge_eligible += 1
            verdicts = evaluate_tool_call(call, task, judge=judge)
            for layer, v in verdicts.items():
                totals[layer][1] += 1
                totals[layer][0] += int(v.passed)

    rates = {layer: (p / t if t else None) for layer, (p, t) in totals.items()}
    return {
        "per_layer": totals,
        "pass_rates": rates,
        "judge_calls_made": judge.calls,
        "judge_calls_eligible": judge_eligible,
        "judge_calls_saved": judge_eligible - judge.calls,
    }
```

### Driver with the diagnostic trace (`run_harness.py`)

```python
from tool_harness import DeterministicJudge, Task, ToolCall, run_suite

# Faithfulness answer key for calls that reach the judge.
ANSWER_KEY = {"t-notify:send_email": True}

CASES: list[tuple[Task, list[ToolCall]]] = [
    # read-only status question that wrongly calls a write tool (SELECTION fail)
    (
        Task("t-status", "read_only_status", "Status of order 4490?"),
        [
            ToolCall("lookup_order", {"order_id": 4490}),
            ToolCall("send_email", {"recipients": ["x"], "body": "fyi"}),  # forbidden
        ],
    ),
    # schema-invalid call: missing required 'order_id' (ARGUMENTS_SCHEMA fail)
    (
        Task("t-bad-schema", "read_only_status", "Status of order 7?"),
        [ToolCall("lookup_order", {})],
    ),
    # schema-valid but value-wrong: limit 5000 (> 100) (ARGUMENTS_VALUES fail)
    (
        Task("t-bigfetch", "read_only_status", "Status of order 7?"),
        [
            ToolCall("lookup_order", {"order_id": 7}),
            ToolCall("list_orders", {"status": "open", "limit": 5000}),
        ],
    ),
    # faithfulness call: recipients extracted from NL, schema+value clean (JUDGE)
    (
        Task(
            "t-notify",
            "notify_managers",
            "Email the two Phoenix managers about refunds.",
            expected_recipient_ids={"mgr-az-01", "mgr-az-02"},
        ),
        [
            ToolCall("list_orders", {"status": "refunded", "limit": 50}),
            ToolCall(
                "send_email",
                {"recipients": ["mgr-az-01", "mgr-az-02"], "body": "Refund summary."},
                needs_faithfulness_judge=True,
            ),
        ],
    ),
]

if __name__ == "__main__":
    judge = DeterministicJudge(ANSWER_KEY)
    report = run_suite(CASES, judge)
    print("per-layer pass rates:")
    for layer, rate in report["pass_rates"].items():
        shown = "n/a" if rate is None else f"{rate:.2f}"
        passed, total = report["per_layer"][layer]
        print(f"  {layer:<24} {shown:>5}  ({passed}/{total})")
    print(
        f"judge calls: made={report['judge_calls_made']} "
        f"eligible={report['judge_calls_eligible']} "
        f"saved={report['judge_calls_saved']}"
    )
```

### Expected output

```text
per-layer pass rates:
  selection                 0.75  (3/4)
  arguments_schema          0.86  (6/7)
  arguments_values          0.83  (5/6)
  arguments_faithfulness    1.00  (1/1)
judge calls: made=1 eligible=1 saved=0
```

Read it as a diagnosis: selection dropped (the read-only task that called the
forbidden `send_email`), one schema failure (`lookup_order` with no `order_id`), and
the value tier caught the `limit: 5000` call (5/6). The seven schema checks are the
seven calls that reached the schema tier; the six value checks are those seven minus
the one schema failure that short-circuited. To see the cascade *save* a judge call,
mark the value-wrong `list_orders` call `needs_faithfulness_judge=True`: `eligible`
rises to 2 but `made` stays at 1, because the value failure short-circuits before the
judge. That gap is the cost win the cascade buys.

## Meeting the acceptance criteria

- **Every verdict tagged by layer; wrong-tool and malformed-args differ** —
  `check_selection` emits `layer="selection"`, the argument checks emit
  `arguments_schema` / `arguments_values` / `arguments_faithfulness`. The forbidden
  `send_email` call shows up under selection; the missing-`order_id` call shows up
  under `arguments_schema`.
- **Schema runs on every call; judge only on flagged calls** —
  `check_arguments_schema` is the first step of `evaluate_tool_call` for every call;
  the judge runs only inside the `needs_faithfulness_judge` branch.
- **Cascade short-circuits, with a count of judge calls saved** — schema failure
  returns before values, value failure returns before the judge, and `run_suite`
  reports `judge_calls_saved = eligible - made`.
- **Per-layer pass-rate breakdown that names the fix** — the report is four rates,
  not one. A selection drop points at tool descriptions; a schema spike points at the
  parameter schema; a value drop points at parameter docs/bounds; a faithfulness drop
  points at task framing or a stronger model (Chapter 2's diagnosis table).

## Common pitfalls

- **Validating the schema last.** Running the judge or value checks before schema
  validation wastes calls on malformed args and produces a faithfulness verdict on a
  call that was never well-formed. Schema is the cheapest, most decisive gate — it
  goes first, always.
- **Folding selection into the argument verdict.** If "called a forbidden tool" and
  "passed a bad limit" land in the same bucket, the breakdown can't tell you whether
  to fix tool *descriptions* or parameter *bounds*. Keep selection a separate layer
  evaluated over the whole trajectory, not per call.
- **Judging what a predicate can check.** The `limit <= 100` bound, the `start <= end`
  invariant, and the recipient-ID consistency check are all deterministic. Sending
  them to a judge burns money and adds variance to a check that has a correct answer.
  Reach for the judge only when the ground truth lives in free text.
- **Counting judge calls saved wrong.** "Saved" is `eligible - made`, where eligible
  means "flagged for faithfulness." A call that was never flagged was never a
  candidate, so it doesn't count as a save — don't inflate the number with calls the
  judge would never have seen.
- **Letting `additionalProperties` default to true.** Without
  `additionalProperties: false`, a hallucinated extra field passes schema validation
  silently. Closed schemas are what make the schema tier catch the model inventing
  parameters.

## Verification

A completed submission is correct when:

- `pip install jsonschema` then `python run_harness.py` prints the four per-layer
  rates and the judge-call line with no other dependencies.
- The selection rate is below 1.0 because of the read-only task that called the
  forbidden `send_email`; flipping that call to an allowed tool restores it to 1.0.
- The missing-`order_id` call shows up only under `arguments_schema` (not selection),
  and the `limit: 5000` call shows up only under `arguments_values`.
- Marking the `limit: 5000` call `needs_faithfulness_judge=True` raises
  `judge_calls_eligible` to 2 while `judge_calls_made` stays 1 — proving the
  short-circuit (and `judge_calls_saved` becomes 1).
- Renaming a required schema field (e.g., `order_id` → `oid`) makes the
  `arguments_schema` rate drop while selection and values are unchanged — the
  localization stretch goal.
- `NOTES.md` answers the three reflection prompts: a schema-valid but value-wrong call
  (`limit: 5000`, caught by the bound predicate — schema can't catch it because 5000
  is a valid integer); a check you first assumed needed a judge but expressed
  deterministically (recipient-ID consistency against the task's expected IDs); and an
  estimate of judge calls saved at suite scale (with 3–8 calls/case over 500 cases,
  deterministic gates dispose of the large majority, leaving only the fuzzy-extraction
  minority for the judge — roughly an order-of-magnitude cost reduction).
