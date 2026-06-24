# mod-308-deployment-durable-execution/exercise-02-hitl-approval-architecture — Solution

## Approach

The deliverable is a *contract*, not a UI: a HITL gate specified well enough that an
engineer can build it on Temporal signals, LangGraph `interrupt`, or any durable
substrate without inventing the lifecycle, the authority rules, the timeout behavior,
or the idempotent-resume discipline. Two gates with deliberately different shapes —
the compliance sign-off (all four decisions; bounces back into the agent loop) and the
wire transfer (no `respond`; two-approver server-validated authority) — so the design
has to prove it can express both, not just one.

The choices that drive the spec below:

- **A gate is a durable interrupt, not a blocking call.** Every gate persists
  `{gate, proposed action, allowed decisions, deadline}` and *yields the worker*. The
  process is then free to die; the human's decision arrives later as a signal that
  resumes the run on any worker. This is the entire reason the wait survives a deploy —
  the state is in the durable plane, not in the pod (Chapter 3).
- **Each of approve/edit/reject/respond is distinguished by what it does to run
  state**, not by its label: approve resumes past the gate, edit resumes with
  human-authored args, reject returns control to the agent to replan, respond injects
  an observation and continues. The valid *subset* per gate is the core artifact.
- **Authority is validated server-side on the signal.** The `decider_identity` carried
  on the decision is checked against the gate's `authority_policy` — never trusted from
  the UI — and the two-approver rule requires two *distinct* validated identities.
- **TTL is one policy with three payoffs.** The approval deadline bounds the gate's
  wait, bounds how long a sleeping run can pin an old rainbow color alive (exercise-03),
  and bounds the fleet tail (exercise-04). Each gate states its TTL and default action.

## Reference solution

### 1. Decision set per gate

| Decision | Compliance sign-off | Wire transfer | Effect on run state |
| --- | --- | --- | --- |
| approve | Valid | Valid | Resume past the gate; execute the proposed action as specified. |
| edit | Valid (edit the risk tier) | Valid (edit amount/recipient down) | Resume with human-authored args treated as authoritative; execute the *edited* action. |
| reject | Valid (back to request more docs) | Valid (cancel the transfer) | Do not execute; return control to the agent with a rejection signal so it replans. |
| respond | **Valid** (answer the agent's question) | **Excluded** | Inject the human's free text as an observation; the agent continues reasoning without executing. |

**Why `respond` is excluded from the wire-transfer gate.** Respond exists for gates
where the agent asked a *question* and the human's prose advances reasoning. A wire
transfer is a terminal, irreversible money-movement decision: the only meaningful
human verbs are "send it," "send a corrected version," or "do not send." A free-text
"respond" at a money gate is an attractive nuisance — it invites an approver to type
guidance instead of making the binding accept/modify/decline call, blurring an
audit-critical decision into an ambiguous note. Restricting the set to
approve/edit/reject forces a clean, recordable decision on every transfer.

### 2. Durable-interrupt lifecycle

```text
   ┌────────────┐   1. reach gate, persist                              ┌──────────┐
   │  agent run │──────  {gate_id, run_id, proposed:A, allowed:[...],   │ durable  │
   │ (workflow) │        deadline, status:WAITING}  ──────────────────▶ │  store   │
   └─────┬──────┘   2. YIELD worker (no thread held; process may die)    └────┬─────┘
         │                                                                    │
         ▼                                                                    │
   ===== process can be KILLED / DEPLOYED / DRAINED here — state is safe ==== │
         │            (the run holds zero compute; it is a row in the store)  │
         │                                                                    │
   ┌─────┴──────┐   5. SIGNAL(decision) wakes the run on ANY worker     ┌─────┴──────┐
   │  resumed   │◀──── validate identity + decision vs authority_policy │  approval  │
   │  run       │      apply branch (approve/edit/reject/respond)       │   API      │
   └────────────┘      ◀────── 4. human decides (minutes…days) ──────── └─────┬──────┘
         │                                                                    │
         │   3a. deadline timer fires before any decision                     │
         ▼                                                                    │
   ┌──────────────┐   default action: reject | escalate | notify              │
   │ TIMED_OUT    │   (per gate; see contract)                                │
   └──────────────┘
```

The marked band is the whole point: between persisting the gate and receiving the
signal, the process is free to be killed, rolled by a deploy, or drained by a node —
because the run's state lives in the durable store, not the worker. The deadline is a
durable timer running in parallel; whichever fires first (a valid signal or the timer)
resolves the gate exactly once.

### 3. Approval contract

| Field | Compliance sign-off | Wire transfer |
| --- | --- | --- |
| allowed_decisions | approve, edit, reject, respond | approve, edit, reject |
| authority_policy (who, thresholds, multi-approver) | role = `compliance_officer`; single approver; edit limited to the risk-tier field | role = `finance`; amount <= $50k -> 1 approver; amount > $50k -> 2 **distinct** approvers; edit may only *lower* amount or correct recipient |
| timeout (deadline + default action) | 72h; default = **escalate** to compliance manager (the run is reversible at this stage — still gathering docs) | 24h; default = **reject** (irreversible action; safe default is do-not-send) |
| correlation_key | `(run_id, gate_id="compliance_signoff")` | `(run_id, gate_id="wire_transfer")` |
| audit_record (fields captured) | proposed action + screening evidence, agent rationale, allowed set, decision, decider identity + role, edited risk tier if any, decided_at, deadline | proposed transfer (amount, recipient, account), agent rationale, decision(s), **each** decider identity + role, edited amount/recipient if any, decided_at per approver, deadline |
| idempotency (duplicate decision handling) | first valid decision for `(run_id, gate_id)` wins; duplicates ignored | first valid decision *per distinct approver*; a re-delivered signal from the same approver is a no-op; the gate resolves only when the required count of distinct valid approvers is reached |

The wire-transfer authority rule, made concrete:

```text
   amount <= $50k  -> 1 approver,  role = finance
   amount >  $50k  -> 2 DISTINCT approvers, role = finance
   each signal: validate decider_identity has role=finance SERVER-SIDE
                (do not trust the UI's claim)
   resolve APPROVE only when count(distinct valid finance approvers) >= required
   default on timeout (24h) -> REJECT   (irreversible action; safe default)
```

### 4. Idempotent resume

A decision delivered twice (the approval API retried on a network blip) must resume the
run exactly once. The gate applies the at-least-once discipline from Chapter 1, keyed
by `(run_id, gate_id)` and — for multi-approver gates — by `(run_id, gate_id,
decider_identity)`:

```text
   on SIGNAL(decision) at gate G for run R:
     if decision invalid for G (not in allowed_decisions):        reject signal, audit
     if decider_identity fails authority_policy:                  reject signal, audit
     # single-approver gate (compliance):
     if store.resolved(R, G):                                     IGNORE (already resumed)
     else: store.mark_resolved(R, G, decision); resume run once
     # multi-approver gate (wire > $50k):
     if store.has_vote(R, G, decider_identity):                   IGNORE (duplicate vote)
     else: store.add_vote(R, G, decider_identity, decision)
           if count(distinct valid APPROVE votes) >= required:    resume run once
```

Because the first valid decision (or vote) for a key is recorded before the run
resumes, a retried delivery finds the key already present and is ignored — the run
resumes exactly once no matter how many times the signal arrives.

### 5. Connect to deploy and tail policy

An approval that can wait forever pins the old rainbow color alive forever (exercise-03)
and stretches the fleet tail (exercise-04). The TTLs above close that:

- **Compliance sign-off — 72h, escalate.** A run waiting here can keep its code version
  alive up to 72h. At the deadline it does not hang: it escalates to a manager, which
  either resolves it or routes it to a default. Version sprawl is therefore bounded to
  ~72h of deploys, after which the gate self-resolves and the run drains.
- **Wire transfer — 24h, reject.** A money gate gets a tighter TTL and a *fail-safe*
  default (do-not-send). Twenty-four hours bounds both the version it pins and its
  contribution to the tail; the reject default guarantees no transfer fires merely
  because nobody clicked.

The connection to make explicit: **the same TTL that times out an approval is the
upper bound on how long a sleeping run can keep an old code version (a rainbow color)
from retiring.** Without it, one un-actioned approval could pin v1 alive indefinitely
and force the platform to carry that version forever.

### Stretch: escalation ladder, proposal payload, idempotency reconciliation

- **Escalation ladder (durable timers, not a poll).** Express it as three durable
  timers armed when the gate is entered: at 4h fire `notify(approver_2)`; at 24h fire
  `escalate(manager)`; at the gate TTL fire the default action. Each timer is a durable
  event; none holds a worker. The first valid decision cancels the remaining timers.
  This is a ladder of signals/timers in the durable plane, never a `while` loop.
- **Proposal payload.** The gate surfaces `{proposed_action, arguments, agent_rationale,
  screening_evidence (compliance) | transfer_details + source-of-funds (wire),
  allowed_decisions, deadline}` so the human decides on evidence, not a bare "approve?".
  A gate with no rationale or evidence is a rubber stamp by design (Chapter 3).
- **Reconciling with exercise-01's idempotency register.** The gated actions are nearly
  the same set as the irreversible side effects: the wire-transfer gate sits exactly at
  an irreversible money write, and the compliance gate sits just before the go-live
  side effects (ERP/billing already created, welcome email pending). The slight
  divergence: the compliance gate guards a *transition* (go-live) rather than a single
  side effect, and the document-upload wait in exercise-01 is a *wait* but not a *gate*
  (no human decision, just an event). Gates and idempotent side effects answer the same
  question — "which actions are consequential enough to need special handling?" — so
  the overlap is expected and the divergences are explainable.

## Meeting the acceptance criteria

- **Explicit, justified decision set per gate, distinguished by run-state effect** —
  the per-gate table lists the valid subset and the effect of each; `respond` is
  excluded from the wire gate with a written rationale.
- **Durable-interrupt lifecycle shows survival of a process death, resumed by signal**
  — the diagram marks the kill/deploy/drain band and explains the run survives because
  state is in the durable plane; resume happens on any worker via the signal.
- **Both contracts complete; wire transfer encodes $50k two-approver and validates
  identity server-side** — the contract table fills all six fields for both gates, and
  the authority block encodes `>$50k -> 2 distinct finance approvers` with server-side
  identity validation.
- **Duplicate decision resumes the run exactly once** — the idempotent-resume pseudocode
  records the first valid decision/vote before resuming, so retries are ignored.
- **Each gate has a TTL with a default action and its effect on version sprawl/tail is
  explained** — 72h/escalate and 24h/reject, each tied to bounding the rainbow color it
  pins and the fleet tail.

## Common pitfalls

- **Modeling the gate as a blocking `input()`.** It holds a worker open and dies with
  the pod on the next deploy; the human's later click lands nowhere. The gate must
  persist its state and yield.
- **Trusting the UI's claim of who decided.** If `decider_identity` is taken from the
  client instead of validated server-side against the authority policy, anyone can
  forge an approver. Validate on the signal.
- **Counting two votes from one identity as two approvers.** The two-approver rule
  requires *distinct* validated identities; the idempotency key `(run_id, gate_id,
  decider_identity)` is what stops one person self-approving twice.
- **No timeout, or a non-safe default.** A gate with no TTL pins a code version forever
  and stalls the run; an irreversible gate that defaults to *approve* on timeout is a
  loaded gun. Irreversible gates default to reject.
- **A context-free proposal.** "Approve?" with no action, arguments, or evidence trains
  humans to rubber-stamp, destroying the gate's value. Always surface the rationale and
  evidence.

## Verification

A completed submission is correct when:

- Each gate lists exactly which of approve/edit/reject/respond are valid, and the wire
  gate excludes `respond` with a stated reason.
- The lifecycle diagram has an explicit "process can die here" band between persisting
  the gate and receiving the decision, with the survival reason (state in the durable
  plane) written out.
- The wire-transfer contract row encodes `<=$50k -> 1 approver`, `>$50k -> 2 distinct
  finance approvers`, server-side identity validation, and a 24h timeout defaulting to
  reject.
- The idempotent-resume logic records the first valid decision (or per-approver vote)
  *before* resuming, so a duplicate signal is a provable no-op.
- Both gates state a TTL and a default action, and the submission ties the TTL to
  bounding rainbow version sprawl (exercise-03) and the fleet tail (exercise-04).
- `NOTES.md` answers the three prompts: why blocking `input()` breaks under a mid-wait
  deploy (the pod dies, the run's in-memory state is lost), the new risk `edit`
  introduces (a human changing agent args — made accountable by the audit record
  capturing the edited args + decider), and what stops a single approver self-approving
  twice on the wire gate (distinct-identity vote keying).
