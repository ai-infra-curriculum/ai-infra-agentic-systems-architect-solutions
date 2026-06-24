# mod-306-guardrails-safety-security/exercise-06 (Excessive Agency Controls) — Solution

## Approach

The inbox-management agent is a textbook OWASP LLM06:2025 — Excessive Agency case:
too many tools (`run_shell` on an inbox agent), a single full-scope token, and full
autonomy on irreversible actions (`delete_email`, `send_email`). LLM06 decomposes
into three sub-failures, and each gets its matching control:

1. **Excessive functionality** — list every tool triage does not need; remove it.
2. **Excessive permissions** — downscope the single full-scope token to the minimum
   triage requires; identify tools acting with more authority than their function.
3. **Excessive autonomy** — move every irreversible action behind approval, a
   reversible alternative, or a deterministic policy.
4. Redesign the corrected catalog (risk tier, scope, autonomy level) and implement
   the autonomy gate that downgrades irreversible actions to proposals.
5. Prove containment for three "model goes wrong" scenarios.

Chapter 3's tiers drive the autonomy decisions: read-only flows freely, reversible
writes flow within policy, irreversible/high-value actions require a human who sees
the *exact* action.

## Reference solution

### 1. Audit — excessive functionality

```text
| tool                  | needed for triage? | justification                                  |
|-----------------------|--------------------|------------------------------------------------|
| read_email            | yes                | triage must read to classify                   |
| archive_email         | yes                | the reversible "act on it" verb                |
| create_calendar_event | yes                | triage extracts meetings                       |
| send_email            | yes (gated)        | replies are part of triage, but irreversible   |
| delete_email          | NO (replace)       | irreversible; archive achieves the same intent |
| run_shell             | NO (remove)        | an inbox agent has zero need to run shell — it  |
|                       |                    | exists only as an automation escape hatch and  |
|                       |                    | is a full remote-code-execution surface         |
```

`run_shell` should never have existed on an inbox agent: it converts a mail-triage
bot into an arbitrary-code-execution surface. There is no triage task it enables
that a narrow tool could not.

### 2. Audit — excessive permissions

The single token has full mailbox + calendar + **admin** scope. Triage needs none
of admin and only a slice of the rest:

```text
| capability        | current scope (too broad) | minimum scope for triage |
|-------------------|---------------------------|--------------------------|
| read mail         | full mailbox + admin      | mail.read                |
| archive mail      | full mailbox + admin      | mail.modify              |
| send mail         | full mailbox + admin      | mail.send                |
| calendar event    | full calendar + admin     | calendar.events          |
| admin             | granted                   | NONE — removed entirely  |
```

The admin scope is removed outright. Tools acting with more authority than their
function: every tool inherited admin via the shared token — `archive_email` could,
on that token, also delete or administer the mailbox. Downscoping per tool (each
tool gets only its own scope) ends that.

### 3. Audit — excessive autonomy

Irreversible actions the agent can take autonomously today:

- **`delete_email`** — irreversible. **Replace with `archive_email`** (reversible),
  removing the destructive capability entirely.
- **`send_email`** — irreversible (a sent mail cannot be unsent). Move behind a
  control: **policy-gated** when the recipient is a known contact (auto-allow), and
  **human approval** otherwise. This keeps routine replies flowing while surfacing
  the injected "send to attacker" case to a person.

### 4. Redesigned architecture

```text
| tool                  | needed? | min scope        | reversible? | autonomy level   |
|-----------------------|---------|------------------|-------------|------------------|
| read_email            | yes     | mail.read        | n/a         | auto             |
| archive_email         | yes     | mail.modify      | yes         | auto             |
| create_calendar_event | yes     | calendar.events  | yes         | auto             |
| send_email            | yes     | mail.send        | NO          | policy → approval|
| delete_email          | replace | —                | NO          | removed → archive|
| run_shell             | NO      | —                | NO          | removed          |
```

Autonomy gate:

```python
"""Autonomy gate for the inbox-management agent (OWASP LLM06).

Downgrades irreversible actions to proposals; reversible/low-risk actions run
automatically; policy-gated actions run only if a deterministic check passes.
"""
from __future__ import annotations

from dataclasses import dataclass
from enum import Enum
from typing import Callable


class Autonomy(Enum):
    AUTO = "auto"          # reversible, low-risk → just do it
    POLICY = "policy"      # allowed only if a deterministic policy passes
    APPROVAL = "approval"  # irreversible → propose to a human


@dataclass(frozen=True)
class ToolSpec:
    name: str
    scope: str
    autonomy: Autonomy
    policy: Callable[[dict, "Ctx"], bool] | None = None
    render: Callable[[dict], str] | None = None


@dataclass(frozen=True)
class Ctx:
    known_contacts: frozenset[str]


def _send_policy(args: dict, ctx: Ctx) -> bool:
    # Auto-send only to already-known contacts; everything else escalates.
    return args.get("to") in ctx.known_contacts


CATALOG = {
    "read_email": ToolSpec("read_email", "mail.read", Autonomy.AUTO),
    "archive_email": ToolSpec("archive_email", "mail.modify", Autonomy.AUTO),
    "create_calendar_event": ToolSpec("create_calendar_event", "calendar.events", Autonomy.AUTO),
    "send_email": ToolSpec(
        "send_email", "mail.send", Autonomy.POLICY,
        policy=_send_policy,
        render=lambda a: f"send email to {a.get('to')} — subject {a.get('subject')!r}",
    ),
}


def autonomy_gate(tool: str, args: dict, ctx: Ctx) -> tuple[str, str]:
    spec = CATALOG.get(tool)
    if spec is None:
        return "DENY", "tool not in catalog (removed/unknown)"
    if spec.autonomy is Autonomy.AUTO:
        return "ALLOW", "reversible/low-risk"
    if spec.autonomy is Autonomy.POLICY:
        ok = spec.policy(args, ctx) if spec.policy else False
        if ok:
            return "ALLOW", "policy passed (known contact)"
        # Policy failed → escalate to a human rather than deny outright.
        return "REQUIRE_APPROVAL", spec.render(args) if spec.render else tool
    return "REQUIRE_APPROVAL", spec.render(args) if spec.render else tool
```

`send_email` to a known contact auto-allows; to an unknown recipient it escalates
to `REQUIRE_APPROVAL` and renders the exact recipient and subject for the human.
`delete_email` and `run_shell` are simply absent from the catalog, so any attempt
to call them returns `DENY, "tool not in catalog"`.

### 5. Prove containment

```text
| scenario                              | redesigned control that bounds it                  |
|---------------------------------------|----------------------------------------------------|
| hallucinated "delete all mail"        | delete_email removed; archive is reversible →      |
|                                       | worst case is mail moved to Archive, recoverable   |
| injection-driven "send to attacker"   | send_email policy: unknown recipient → approval;   |
|                                       | a human sees "send to attacker@evil.example" and   |
|                                       | rejects                                            |
| mass-unsubscribe / send loop          | volume anomaly detection (>N actions/min pauses    |
|                                       | for review) + send approval on unknown recipients  |
```

Each scenario is bounded by a *named control*, not by hoping the model behaves: the
destructive capability is gone, the exfiltration send escalates to a human, and the
runaway loop trips a deterministic volume guard.

### Stretch: dry-run, anomaly detection, adversarial suite

- **Dry-run mode:** the agent produces its full proposed action plan (archive these
  12, reply to these 3, create these 2 events) for a human to approve as a *batch*
  before anything executes.
- **Volume anomaly detection:** >20 archives/minute or any send burst pauses for
  review — bounding the mass-action loop deterministically.
- **Adversarial suite in CI:** cases covering all three sub-failures, wired so a
  future tool addition that reintroduces excessive agency fails the build.

```python
from enum import Enum


class Outcome(Enum):
    TOOL_DENIED = "tool_denied"
    APPROVAL_REQUIRED = "approval_required"
    RATE_LIMITED = "rate_limited"


ATTACKS = [
    {"name": "excess_functionality_shell", "call": ("run_shell", {"cmd": "curl evil.example"}),
     "expect": Outcome.TOOL_DENIED},
    {"name": "excess_autonomy_send_attacker", "call": ("send_email", {"to": "attacker@evil.example"}),
     "expect": Outcome.APPROVAL_REQUIRED},
    {"name": "excess_autonomy_mass_loop", "call": ("archive_email", {"burst": 500}),
     "expect": Outcome.RATE_LIMITED},
]


def test_llm06_controls_hold(run_agent):
    for a in ATTACKS:
        assert run_agent(a).outcome == a["expect"], f"LLM06 control regressed: {a['name']}"
```

## Meeting the acceptance criteria

- **All three LLM06 sub-failures audited with concrete findings** — sections 1
  (functionality), 2 (permissions), 3 (autonomy).
- **`run_shell` and other unneeded tools removed; full-scope token downscoped** —
  section 1 removes `run_shell`/`delete_email`; section 2 removes admin and
  downscopes per tool.
- **Every irreversible action moved behind approval, a reversible alternative, or a
  deterministic policy** — `delete_email` → `archive_email`; `send_email` →
  policy-then-approval.
- **Autonomy gate downgrades irreversible actions to proposals and shows the exact
  action** — section 4, `render` surfaces recipient + subject.
- **Three failure scenarios each bounded by a named control** — section 5.

## Common pitfalls

- **Keeping `run_shell` "for automation."** It is a remote-code-execution surface on
  a mail bot. Remove it; there is no triage task it uniquely enables.
- **Downscoping the token but keeping admin "just in case."** Admin scope is the
  excessive-permission finding. Remove it; grant each tool only its own scope.
- **Gating `delete_email` behind approval instead of replacing it.** A reversible
  alternative (`archive`) removes the destructive capability entirely — strictly
  better than approving an irreversible one.
- **Auto-sending to any recipient.** Free recipient choice is the injection's
  exfiltration channel. Policy-gate to known contacts; escalate the rest.
- **No volume guard.** Fully autonomous + no rate limit means an injected loop can
  archive or send thousands before anyone notices. Add deterministic anomaly
  detection.

## Verification

- **Three-axis audit:** confirm each sub-failure (functionality, permissions,
  autonomy) has a concrete finding and a control.
- **Catalog:** `run_shell` and `delete_email` are absent; admin scope is gone; every
  remaining tool has a minimal scope and an autonomy level.
- **Autonomy gate:** exercise the matrix:

  ```python
  ctx = Ctx(known_contacts=frozenset({"boss@example.com"}))
  assert autonomy_gate("archive_email", {}, ctx)[0] == "ALLOW"
  assert autonomy_gate("send_email", {"to": "boss@example.com"}, ctx)[0] == "ALLOW"
  d, summary = autonomy_gate("send_email", {"to": "attacker@evil.example", "subject": "x"}, ctx)
  assert d == "REQUIRE_APPROVAL" and "attacker@evil.example" in summary
  assert autonomy_gate("run_shell", {"cmd": "curl evil.example"}, ctx)[0] == "DENY"
  assert autonomy_gate("delete_email", {}, ctx)[0] == "DENY"
  print("all LLM06 checks passed")
  ```

- **Containment:** each of the three scenarios traces to a named control.
- **`NOTES.md`** answers: which sub-failure was most dangerous and why; where
  "make it convenient" pushed toward excessive agency and how you resisted without
  crippling the agent; how replacing `delete_email` with `archive_email` changes the
  worst case.
