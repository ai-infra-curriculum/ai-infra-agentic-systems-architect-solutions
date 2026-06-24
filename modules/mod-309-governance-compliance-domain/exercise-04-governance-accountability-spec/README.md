# mod-309-governance-compliance-domain/exercise-04-governance-accountability-spec — Solution

## Approach

The deliverable is three interlocking **fleet-level specifications** — a *lineage* spec,
an *audit-trail* spec, and a *human-accountability* spec — plus a risk-control catalog
and a worked incident walk-through. The bar is that an engineer can implement them and
an auditor can test them: precise field lists and schemas, not prose about
"observability." The specs are **regime-neutral in structure** — the active regime
profile only sets the gate authority and retention parameters (consumed from exercise-02,
not re-derived here). The scenario fixes one regime to make the parameters concrete; the
structure would be identical under any of the four peer regimes.

The choices that drive the specs below:

- **Lineage answers "what produced this," audit answers "what happened," and they are
  linked by `run_id`.** Lineage is the provenance of an artifact (version/model/prompt/
  tools/data/run/regime). The audit trail is the ordered sequence of consequential events
  in that run. A failure is traceable only when both exist and share the run id.
- **Lineage is itself minimized.** It references data *by class and source*, never raw
  content — so the provenance record does not become a second copy of the sensitive data
  it describes.
- **Accountability without intervention power is hollow.** Every accountable role in the
  RACI is wired to a concrete intervention path (kill-switch, per-agent disable, tool
  revocation) it can actually invoke. A role that "owns" a behavior it cannot stop is a
  scapegoat, not an owner.
- **The audit trail must outlive the raw data and resist a privileged operator.** It is
  append-only and tamper-evident, on a per-regime retention window that can exceed the
  raw-data window, and it answers a stated *acid-test query*.

## Reference solution

The active regime for the worked specs: a **finance (GLBA/SOX/PCI)** deployment — chosen
to make the parameters concrete (gate authority = control owner / segregation of duties;
retention = SOX multi-year). The structure is regime-neutral; swapping in healthcare,
public sector, or edtech changes only the authority and retention parameters.

### 1. Lineage spec (provenance of an output/decision)

For **any** consequential output or decision, the fleet records the provenance chain:

```text
LineageRecord {
  run_id:            the durable run identifier (links to mod-308 run history)
  agent_version:     which agent code version produced it (a rainbow color)
  model:             { model_version, provider, region }
  prompt_version:    which prompt template/version
  tool_versions:     [ tool@x.y.z, ... ]   // resolved from the exercise-03 registry
  data:              { class, source }       // BY CLASS/SOURCE — never raw content
  active_regime:     regime_profile_id        // from exercise-02
  produced_at:       timestamp
}
```

- **Minimized:** `data` carries *class and source*, not the content — so lineage is not a
  shadow copy of the sensitive record. The raw datum lives in the data plane under the
  retention/erasure controls of exercise-05; lineage points at it by reference.
- **Links to durable run history:** `run_id` is the join key to the mod-308 durable
  history, so the provenance chain attaches to the full run timeline (every step, retry,
  and gate). `tool_versions` are the exact pins from the exercise-03 registry record.

### 2. Audit-trail spec (sequence of consequential events)

**What is recorded** — only *consequential* events, not every log line:

- Actions taken (writes to records, notifications to external parties).
- Decisions, **including the model's** (what the agent chose and why).
- Human gate decisions (approve/edit/reject/respond, with the decider).
- Policy enforcements (a guardrail block, an out-of-scope tool call denied).
- Version/config changes (a deploy, a tool admission/revocation, a regime-profile change).

**The record schema** (fixed field list):

```text
AuditRecord {
  ts:                 timestamp
  run_id:             links to the lineage record + durable history
  agent:              agent_id + agent_version
  actor:              agent | human:<role>     // who acted
  event_type:        action | model_decision | gate_decision |
                      policy_enforcement | version_change
  subject_data_class: the class of data the event concerned (minimized)
  outcome:           executed | blocked | rejected | edited | escalated
  rationale:         short reason / evidence ref (optional but expected on gates)
}
```

**Immutability / integrity:** append-only and **tamper-evident**, riding the durable
history. Append-only-in-a-table is the floor; the stretch hardens it with hash-chaining
so a privileged operator cannot rewrite the past undetected.

**Retention + queryability:** retained for the **per-regime** window (here: SOX
multi-year — a *parameter* from the regime profile, not hardcoded). Queryable by
**subject, run, agent, time, and action type**.

**Auditor's acid-test query** the spec can answer:

> *"For subject S between dates D1 and D2: every consequential action taken about them,
> by which agent version and model, what each gate decided and who decided it, and which
> were blocked by policy."*

If the schema and indices above are implemented, this query returns a complete,
attributable answer — which is the whole point of the audit trail.

### 3. Human-accountability spec

**Decision gates** (HITL gate → accountable role, parameterized by the active regime):

| Action class | Gate? | Accountable role (finance profile) | Under another regime |
| --- | --- | --- | --- |
| Read record (low consequence) | no gate; logged | n/a | n/a |
| Write a case note | gate at threshold | operations owner | clinician / agency official / school |
| External-party notification with effect | gate | control owner | covered-entity authority / designated official |
| Money-moving / irreversible action | gate, 2 approvers | finance control owner + second approver (segregation of duties) | parameterized authority + count |

**Fleet RACI** (for the fleet's *behavior*, not a project):

| Behavior | Responsible | Accountable | Consulted | Informed |
| --- | --- | --- | --- | --- |
| Day-to-day fleet operation | platform/SRE on-call | fleet governance lead | regime compliance owner | product owner |
| Consequential gate decisions | the gate's named approver | fleet governance lead | legal/compliance | affected business unit |
| Tool admission/revocation | extension-governance owner | fleet governance lead | security | platform team |
| Incident response | incident commander | fleet governance lead | legal, security, comms | execs, affected parties |

**Intervention paths** (who may invoke each):

```text
   kill-switch (halt the whole fleet)      -> incident commander OR governance lead
   per-agent disable (stop one agent/ver)  -> platform on-call OR governance lead
   tool revocation (pull a tool)           -> extension-governance owner
                                              (registry status flip, exercise-03)
```

**Contestability** (appeal path for an affected individual):

```text
   affected individual disputes an action
     -> intake at the contestability handler (named role: governance lead's
        delegate / ombudsperson)
     -> handler pulls the run via run_id: lineage + audit trail show what was
        done, by which version, what each gate decided and who decided it
     -> handler can trigger remediation (reverse if reversible) + records the
        disposition in the audit trail
   named handler: the contestability officer (a specific role, not "support")
```

### 4. Risk-control catalog

Top fleet risks consumed from the RMF Map of exercise-01:

| Risk | Control | Component | Evidencing spec | Owner |
| --- | --- | --- | --- | --- |
| R1 harmful/incorrect output | output guardrail + HITL gate on high-consequence claims | mod-306 guardrail; mod-308 gate | audit trail (gate_decision records) | fleet governance lead |
| R2 action beyond intent | HITL gate on consequential actions | mod-308 decision gates | human-accountability spec (gates) | gate's accountable role |
| R3 tool misuse / over-scope | runtime scope enforcement + revocation | exercise-03 registry + runtime | lineage (tool_versions) + audit (policy_enforcement) | extension-governance owner |
| R4 data leakage | minimization + region pinning | exercise-05 boundary controls | lineage (data by class) + audit | data governance owner |
| R5 prompt injection | input guardrails + untrusted-content quarantine | mod-306 | audit (policy_enforcement) | fleet governance lead |
| R6 drift | drift detection vs. Evaluate baseline | mod-304/305 + exercise-03 | audit (version_change) + registry baseline | extension-governance owner |
| R7 untraceable incident | lineage + audit + accountability specs | this exercise | the three specs themselves | fleet governance lead |

**Open findings** (rows whose component or owner is not yet wired):

- **OF-1:** tamper-evidence (hash-chaining) for the audit trail is specified but not yet
  implemented — until then R7's integrity guarantee is partial (see stretch).
- **OF-2:** the periodic governance-review cadence has no named owner schedule yet (see
  stretch).

### 5. Incident walk-through (failure made traceable)

**Incident:** a fleet agent took a wrong, harmful action affecting an individual — it
sent an external-party notification with a damaging error about subject S.

**With the specs:**

```text
   1. Contestability intake: S disputes the notification -> handler opens case,
      gets run_id from the action reference.
   2. LINEAGE (by run_id): agent_version = v2.3.1, model = {m, provider, region},
      prompt_version = p7, tool_versions = [notify@2.1.0], data = {class:
      customer-NPI, source: records}, active_regime = finance.
      -> WHICH version/model/tools/data produced it: answered.
   3. AUDIT TRAIL (by run_id, ordered): model_decision (chose to notify, rationale),
      gate_decision (control owner approved at the external-notification gate,
      decider recorded), action (executed), then the dispute event.
      -> WHAT was decided and BY WHOM: answered.
   4. ACCOUNTABILITY: the gate's accountable role (control owner) owns the
      decision; the fleet governance lead is accountable for the fleet behavior.
      -> WHO is accountable: answered.
   5. REMEDIATION: handler triggers tool revocation if the tool misbehaved
      (registry flip) and/or per-agent disable of v2.3.1; if the action is
      reversible, reverses it; disposition recorded in the audit trail.
      -> contestability + revocation paths: exercised.
```

**Without the specs:** the investigation has a notification and an angry individual and
*no run_id join*. You cannot say which of several live agent versions produced it, which
model/region, which tool, or whether a human approved it — because gate decisions were
never recorded with a decider, lineage never captured the version, and "we have logs"
means scattered application logs with no subject/run index. The incident is
*unanswerable*: no attribution, no accountable owner, no clean remediation. The contrast
*is* the value of the three specs.

### Stretch: tamper-evidence, review cadence, deletion-right reconciliation

- **Tamper-evidence mechanism.** Hash-chain each audit record (`hash_n = H(record_n ||
  hash_{n-1})`) and periodically anchor the head hash to an external/WORM store. A
  privileged operator who edits an old record breaks the chain from that point forward —
  detectable on verification. Append-only-in-a-DB is insufficient because a DBA with write
  access can rewrite a row *and* its "append-only" metadata; the chain makes any edit
  evident without trusting the operator.
- **Periodic governance review.** The fleet governance lead reviews the risk-control
  catalog **quarterly**; an **out-of-cycle** review is triggered by any incident, a new
  regime onboarding, or a structural change to the fleet (new high-consequence tool class).
  Each review re-checks every catalog row's component+owner and reopens findings.
- **Lineage retention vs. deletion-right.** The audit trail may need to outlive the raw
  data it describes (SOX) while a deletion right requires erasing the subject's raw datum.
  Resolution: because lineage and audit reference data **by class/source, not content**,
  erasing the raw datum (in the data plane, exercise-05) leaves the audit trail intact —
  the trail still proves *what happened to a customer-NPI record* without holding the NPI
  itself. The audit store and the raw store are separate, on separate retention windows;
  erasure hits the raw store, the audit reference survives.

## Meeting the acceptance criteria

- **Lineage records the full chain, is minimized, links to durable history + registry** —
  the LineageRecord lists version/model/prompt/tools/data/run/regime, references data by
  class/source, joins mod-308 via run_id, and pulls tool versions from the exercise-03
  registry.
- **Audit-trail spec fixes what/schema/immutability/retention+queryability and states
  the acid-test query** — consequential-event list, fixed AuditRecord schema, append-only/
  tamper-evident, per-regime retention, five query axes, and the stated auditor query.
- **Human-accountability names roles per gate, a fleet RACI, intervention paths with
  who-may-invoke, and a named contestability handler** — all four present, gate authority
  parameterized by regime.
- **Risk-control catalog ties risk → control → component → spec → owner, with open
  findings** — seven rows plus two listed open findings.
- **Incident walk-through traceable end-to-end with explicit without-specs contrast** —
  the five-step trace resolves version/model/tools/data/decider/owner/remediation, and the
  without-specs paragraph shows it is unanswerable.

## Common pitfalls

- **Omitting agent version from lineage.** With several live versions, an incident you
  cannot attribute to a version is an incident you cannot fix or bound — the without-specs
  contrast turns on exactly this.
- **"We have audit logging."** Ordinary application logging is not an audit trail: it
  rarely has a fixed consequential-event schema, is not append-only/tamper-evident, and is
  not indexed by subject/run. The spec pins all three.
- **Accountability with no intervention power.** Naming an accountable role that cannot
  invoke a kill-switch/disable/revoke is a scapegoat assignment. Wire each accountable
  role to a path it can actually trigger.
- **One store for audit and raw data.** Conflating them makes the deletion-right vs.
  retention reconciliation impossible — separate the stores and reference data by class.
- **Hardcoding retention/authority.** These are regime parameters consumed from
  exercise-02. Baking SOX multi-year or a finance approver into the spec breaks it for the
  other peer regimes; the structure stays neutral and the parameters come from the profile.

## Verification

A completed submission is correct when:

- The lineage spec lists agent version, model (+ provider/region), prompt version, tool
  versions, data class/source, run id, and active regime; references data by class/source;
  and joins both the durable run history and the tool registry by run_id.
- The audit-trail spec enumerates the consequential event types, gives a fixed record
  schema, states append-only + tamper-evidence, sets a per-regime retention window, lists
  the five query axes, and writes out the auditor acid-test query it answers.
- The human-accountability spec names the accountable role at each gate (parameterized by
  regime), gives a fleet RACI, lists kill-switch/disable/revoke intervention paths with who
  may invoke each, and names the contestability handler.
- The risk-control catalog ties each top risk to a control, component, evidencing spec, and
  owner, with any missing-component/owner rows listed as open findings.
- The incident walk-through resolves which version/model/tools/data/run, what was decided
  and by whom, who is accountable, and the contestability/revocation paths — and the
  without-specs contrast is written out.
- `NOTES.md` answers the three prompts: a concrete case where omitting agent version makes
  an incident unanswerable, three things the spec pins down that ordinary logging does not,
  and one accountable role traced to an intervention path it can actually invoke.
