# mod-302-multi-agent-orchestration/exercise-02 — Solution

## Approach

The exercise turns on one decision — handoffs vs. manager-as-tools — and it is
rigged so that decision is forced by a single requirement: the account/security
domain is risk-sensitive, and Chapter 2 says guardrails are **diffuse under
handoffs** (every specialist must re-enforce them) and **a single choke point under
manager-as-tools** (enforce once, at the manager). A system that must guarantee an
un-bypassable authorization gate cannot afford diffuse enforcement. That pushes the
answer toward manager-as-tools — but the technical domain wants a long multi-turn
diagnosis, which is exactly where handoffs shine. So the defensible answer is a
**blend with a manager backbone**, not a pure model.

The design choices that follow from that:

- **Manager owns the user-facing answer at all times.** One voice, one
  accountability point, one place the safety/authorization guardrail lives. This is
  the Chapter 2 manager-as-tools win, and it is non-negotiable because of the
  security domain.
- **Billing and security are manager-as-tools calls.** They are request/response
  shaped — "process this refund," "reset this password after auth" — and both touch
  money or access, so they must return *through* the manager where the guardrail
  sits. Specialists never speak to the user for these.
- **Technical troubleshooting is a *bounded handoff*.** Multi-turn diagnosis is the
  one place a specialist genuinely benefits from owning the conversation (Chapter 2:
  "a specialist that must walk a user through a multi-step process is better off
  holding the conversation"). So the manager hands off to the technical specialist,
  which owns the answer *for that slice*, and hands back when diagnosis completes.
  The handback edge is drawn explicitly and names the manager as the returning owner
  — an implicit handback is how conversations get stranded (Chapter 2, orphaned
  ownership).

The guardrail is the load-bearing part. Because security flows through the manager
as a tool call, the authorization check lives at the manager and **cannot be reached
except through it** — there is no edge that lets a user-facing specialist mutate
account state directly. The technical specialist, even though it owns the
conversation during diagnosis, has **no security tool** in its scope, so a user who
asks it for a password reset mid-diagnosis is handed back to the manager rather than
served. That is the proof the gate is un-bypassable: the bypass would require an edge
the topology does not contain.

## Reference solution

### Decision matrix

```text
| Decision                | Choice                          | Why                                            |
| ----------------------- | ------------------------------- | ---------------------------------------------- |
| Delegation model        | manager-as-tools backbone +     | security forces a single guardrail choke       |
|                         | one bounded handoff (technical) | point; technical wants multi-turn ownership    |
| Answer owner (billing)  | manager (always)                | money-touching; must return through guardrail  |
| Answer owner (technical)| technical specialist DURING     | multi-turn diagnosis benefits from specialist  |
|                         | diagnosis; manager before/after | ownership; manager resumes on handback         |
| Answer owner (security) | manager (always)                | risk-sensitive; auth gate lives at manager     |
| Guardrail choke point   | manager, on the security tool   | only path to account mutation; un-bypassable   |
| Transfer/return contract| typed (see below)               | prevents context loss + orphaned ownership     |
```

### Control flow

Billing and security are tool calls (manager always owns the answer). Technical is a
handoff with an explicit handback that re-names the manager as owner.

```text
CONTROL FLOW — manager backbone, one bounded handoff

  user ──▶ ┌─────────────────────────────┐
           │          manager            │  owns answer by default
           │  (single guardrail choke)   │
           └──┬───────────┬───────────┬──┘
     call{request,        │           │  handoff{reason,intent,state,owner=technical}
       return_shape}      │           ▼
        ▼                 │     ┌──────────────────┐
  ┌──────────┐            │     │ technical        │  owns answer DURING diagnosis,
  │ billing  │            │     │ specialist       │  talks to user directly
  │ (tool)   │            │     └────────┬─────────┘
  └────┬─────┘            │              │ handback{result,owner=manager}
       │ return{result}   │              ▼   (manager resumes ownership + voice)
       └──────────────────┤        back to manager
                          │
                  call{request,return_shape}     ┌───────────────────────────┐
                          ▼                       │ AUTHORIZATION GUARDRAIL   │
                  ┌──────────────┐  must pass ───▶│ runs at manager BEFORE    │
                  │ security     │                │ the security tool is even │
                  │ (tool)       │◀── only after  │ invoked. No other edge    │
                  └──────┬───────┘    auth passes  │ reaches account mutation. │
                         │ return{result}          └───────────────────────────┘
                         ▼
                  manager composes ──▶ user   (one consistent voice end to end)
```

Ownership is single-valued at every point: the manager owns the answer except during
an active technical diagnosis, where the technical specialist owns it — and the
handback edge re-assigns ownership to the manager by name, so there is never a
moment with zero owners or two.

### Contracts

```text
HANDOFF payload (manager ──▶ technical):
  {
    reason:      "multi_turn_hardware_diagnosis",
    user_intent: "<the diagnosed symptom, verbatim>",
    state:       { ticket_id, account_tier, prior_steps_tried[] },
    new_owner:   "technical_specialist"        // names the new owner explicitly
  }

HANDBACK payload (technical ──▶ manager):
  {
    result:      "<diagnosis outcome + recommended action>",
    state:       { resolved: bool, escalation_needed: bool, steps_taken[] },
    new_owner:   "manager"                      // returns ownership by name
  }

MANAGER-AS-TOOLS call (manager ──▶ billing | security):
  request:       { action, params, auth_context }   // auth_context meaningful for security
  return_shape:  { status, result, audit_id }       // fixed shape the manager composes from

SECURITY tool precondition (enforced at the manager, not in the tool):
  the manager runs authorization(auth_context) and refuses to emit the security
  call at all unless it passes. The tool is never reachable pre-auth.
```

### Guardrail placement — the hard part

The authorization/safety guardrail lives **at the manager, on the path to the
security tool**, and it is un-bypassable because:

1. **Security is a tool, not a handoff.** No specialist ever holds the conversation
   while touching account state. The only edge into account mutation is the
   manager's `call` to the security tool, and that call is gated by `authorization()`
   which the manager runs *before* emitting it.
2. **The technical specialist — the one agent that does hold the conversation — has
   no security tool in scope.** A user mid-diagnosis who says "while you're here,
   reset my password" cannot be served by the technical specialist; the specialist
   hands back to the manager, which routes the request through the gated security
   tool. The bypass a handoff-only design would allow (a specialist quietly
   performing a sensitive action) is structurally impossible because the edge does
   not exist.
3. **One choke point, one audit.** Every account mutation carries the `audit_id` from
   the single gated path, so the security log has exactly one chokepoint to monitor.

Had this been a pure-handoff design, the guardrail would be diffuse: the security
specialist would own the conversation and the manager's authorization rule would be
silently bypassed the moment control transferred (Chapter 2, diffuse guardrails).
The blend keeps the technical handoff's benefit while denying any handoff into the
risk-sensitive domain.

## Meeting the acceptance criteria

- **Model chosen and justified against voice, autonomy, and guardrail placement** —
  manager-as-tools backbone (one voice, one guardrail choke) with a single bounded
  handoff for technical (the one domain whose multi-turn autonomy earns it).
- **Exactly one named owner everywhere** — manager owns by default; technical owns
  *only* during active diagnosis; the handback edge re-names the manager. No point
  has zero or two owners.
- **Diagram shows handback / return edges with owner named** — the technical
  handback carries `new_owner=manager`; billing and security show `return{result}`
  edges back to the manager.
- **Typed transfer/return contracts that prevent context loss** — handoff carries
  `{reason, user_intent, state, new_owner}`; handback carries `{result, state,
  new_owner}`; tool calls carry `{request, return_shape}`.
- **Security choke point identified and shown un-bypassable** — the auth gate runs
  at the manager before the security tool is reachable, and the only
  conversation-owning specialist (technical) has no security tool, so no transfer
  can route around the gate.

## Common pitfalls

- **Choosing pure handoffs because they feel "more agentic."** Handoffs scatter the
  guardrail across every specialist; the security domain alone makes that
  indefensible. If you pick handoffs here, you must replicate the authorization rule
  into every specialist that can touch account state — and prove the replication
  can't drift. The manager backbone avoids that entirely.
- **Leaving ownership orphaned after the technical handoff.** If the technical
  specialist finishes and no edge re-assigns the answer, the user is stranded
  (Chapter 2). Every handback must name the returning owner; draw the edge, don't
  imply it.
- **Letting the technical specialist hold security tools "for convenience."** The
  instant the conversation-owning specialist can mutate account state, the
  un-bypassable gate becomes bypassable. Scope security to the gated manager path
  only; the technical specialist hands sensitive requests back.
- **Untyped transfer payloads.** A handoff that drops `reason` or `state` forces the
  receiver to re-derive intent, burning turns and risking a wrong diagnosis. The
  payload is a typed contract, not a vibe.
- **Ignoring the manager's context cost.** Manager-as-tools re-pays every
  specialist's return in the manager's context (Chapter 2 cost row). On a
  three-domain conversation this is real; the mitigation is the same as
  exercise-01 — specialists return *distilled* results in the `return_shape`, not
  raw working transcripts.

## Verification

A completed submission is correct when:

- The delegation choice is justified by all three levers (voice, autonomy,
  guardrail), not just one, and the security domain is explicitly the reason a pure
  handoff is rejected.
- Tracing any conversation, every point has exactly one named answer owner; the
  technical handoff has a matching, explicitly-drawn handback that re-names the
  owner.
- Each transfer/return edge in the diagram has a typed payload listed, and the
  payloads carry enough (`reason`, `user_intent`, `state`) to avoid re-derivation.
- The security guardrail's choke point is a single named path, and the submission
  shows *why* no other edge reaches account mutation — ideally by pointing out that
  the only conversation-owning specialist lacks the security tool.
- `NOTES.md` answers the three prompts: the all-three-domain trace with ownership
  movements and whether the voice stays consistent (it does — the manager is the
  voice except during diagnosis); the context-cost delta vs. the pure alternative
  (manager-as-tools costs more context, pure handoffs cost the guardrail); and the
  ping-pong bound (cap manager↔technical handoffs at K, then escalate — ties to
  exercise-04).
