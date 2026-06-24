# mod-309-governance-compliance-domain/exercise-05-governance-and-accountability-spec — Solution

This is the capstone reference artifact: the fleet-level governance,
accountability, and risk-control specification for the four-agent **TutorMesh**
fleet. It is the union of six living artifacts — agent registry, risk-tier model,
RACI, risk register, control catalog, and incident/change process — and it
integrates exercises 01–04. It is specification, not code.

> **Not legal advice.** Framework references (ISO/IEC 42001:2023, NIST AI 100-1)
> and statutory references (FERPA, COPPA) are stated for mapping; verify against
> the primary sources, and have counsel review obligations touching minors and
> regulated data.

## Approach

A single agent fits in your head; a fleet does not. Fleet governance must become
specifications and registries that hold without anyone remembering the details —
an operator runs against them, an auditor reviews them, and a new team inherits
them. Each artifact must be living, owned, and traceable back to a control. The
six artifacts answer six questions:

1. **Registry** — what exists and who owns it (and the rule: not in the registry =
   not authorized to run).
2. **Risk-tier model** — how much rigor, scaled to the impact of a wrong or
   malicious action, so Tier 4 is gated hard and Tier 1 moves fast.
3. **RACI** — who decides, with exactly one Accountable per activity, named as a
   role with a person, not a committee.
4. **Risk register** — what could go wrong, with residual risk explicit and owned,
   including at least one minor-affecting risk.
5. **Control catalog** — how each obligation is realized and *evidenced*, with the
   framework reference it satisfies.
6. **Incident/change process** — how change and failure are handled, with a
   *drilled* fleet-wide kill switch and a continual-improvement loop.

The capstone judgment the exercise tests is **aggregate risk**: each agent may be
individually acceptable while the fleet's combined data access or action volume is
not, and someone must own that view.

## Reference solution

### TutorMesh Fleet Governance Specification

```text
Owner: AIMS owner        Review cadence: quarterly (off-cycle triggers below)
Version: 1.0             Scope: the four production agents + shared KB, plugin
                         registry, base safety prompt, managed inference endpoint
```

### Artifact 1 — Agent registry

Rule: **an agent not in the registry is not authorized to run** — the runtime
checks the registry at start, exactly as it does for plugins (exercise-02).

| Agent | Owner (role) | Purpose | Tier | Data classes | Tools / plugins | HITL | Model + prompt ver | Status | Impact assess ref |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| `tutor` | Tutoring product owner | Student-facing tutoring for minors | 4 | Education record; child PII (<13) | `answer_question`, `generate_practice_problem`, `give_feedback`, `retrieve_curriculum` | Crisis escalation; HITL on record-affecting output | model `m-2.1` / prompt `tutor-v7` | active | IA-tutor-03 |
| `progress-summarizer` | Progress product owner | Writes dashboard notes; drafts parent summaries | 3 | Education record | `draft_progress_note`, `sis-writeback`, `retrieve_curriculum` | Blocking HITL before parent send / record write | model `m-2.1` / prompt `psum-v4` | active | IA-psum-02 |
| `lesson-drafter` | Curriculum owner | Teacher-facing lesson-plan drafting | 2 | Curriculum (non-PII) | `retrieve_curriculum`, `draft_lesson` | Teacher reviews drafts (post-hoc) | model `m-2.1` / prompt `ldraft-v3` | active | IA-ldraft-01 |
| `usage-reporter` | Analytics owner | Read-only internal analytics | 1 | De-identified usage telemetry | `query_usage` (read-only) | None (no consequential action) | model `m-1.4` / prompt `usage-v2` | active | IA-usage-01 |

### Artifact 2 — Risk-tier model

Tiers defined by **impact of a wrong or malicious action**, with a control set per
tier.

```text
TIER 4 — Critical: acts on minors/regulated subjects or irreversible/high-impact
                   actions. -> mandatory HITL on consequential actions; full audit;
                   highest review cadence; legal/privacy sign-off.
TIER 3 — High:     writes to systems of record, or touches regulated data without
                   minor/irreversible impact. -> scoped write tools; audit;
                   named-owner sign-off; eval gate.
TIER 2 — Moderate: reads regulated/internal data; no external action.
                   -> read scoping; audit; standard eval gate; owner approval.
TIER 1 — Low:      read-only on non-sensitive data; ephemeral output.
                   -> lightweight review; monitoring; auto-approve under policy.
```

Placements and justification:

- `tutor` → **Tier 4**: it acts directly on minors with education-record access;
  a wrong or malicious action reaches a child. Hardest gate.
- `progress-summarizer` → **Tier 3**: it writes to a system of record (via
  `sis-writeback`) and produces parent-facing output, but through a HITL gate; not
  Tier 4 only because no autonomous child-facing action — a human approves every
  consequential send.
- `lesson-drafter` → **Tier 2**: reads curriculum (non-PII), drafts for a teacher,
  no external action and no regulated-data write.
- `usage-reporter` → **Tier 1**: read-only over de-identified telemetry, ephemeral
  output, no consequential action — moves fast under policy.

### Artifact 3 — RACI (exactly one Accountable per row)

| Activity | Responsible | Accountable | Consulted | Informed |
| --- | --- | --- | --- | --- |
| Approve a new Tier 4 agent | Agent team lead | Head of AI platform | Legal, Privacy, Security | Fleet operators |
| Approve a plugin (exercise-02) | Platform eng | Plugin governance owner | Security | Affected agent owners |
| Run an impact assessment | Agent owner | Agent owner | Privacy | AIMS owner |
| Respond to a safety incident | On-call SRE | Incident commander | Owner, Legal | Leadership |
| Set / change a risk tier | Agent owner | AIMS owner | Risk lead | Platform team |
| Execute a fleet-wide kill switch | On-call SRE | Incident commander | — | All owners |

Rule enforced: exactly one **A** per row, named as a role with a person behind it,
not a committee — the risk traces to one human who signed for it.

### Artifact 4 — Risk register

Sourced from the NIST GenAI Profile categories and exercises 01–04. Residual risk
is explicit and owned; at least one risk affects a minor.

| Risk ID | Description | Affected parties | Likelihood / Impact | Mitigating control(s) | Residual | Owner | Review date |
| --- | --- | --- | --- | --- | --- | --- | --- |
| R-014 | Tutor asserts an ungrounded fact to a student | Students (minors) | Medium / High → High | Grounded retrieval; output claim check; bias-to-defer (C-20) | Low | Tutoring product owner | quarterly |
| R-021 | Education-record PII egresses via a plugin | Students, district | Medium / High → High | Plugin registry + scope enforcement (C-12); egress deny-by-default | Low–Med | Plugin governance owner | quarterly |
| R-023 | Crisis signal from a minor mishandled | Students (minors) | Low / High → High | Crisis detection + named-human escalation (C-18) | Low | Safety lead | quarterly |
| R-030 | Parent-facing summary is wrong and rubber-stamped | Students, parents | Medium / Medium → Med | HITL gate + anti-rubber-stamp surface (C-15) | Low–Med | Progress product owner | quarterly |
| R-035 | A drifted/compromised plugin runs with agent authority | Students, district | Medium / High → High | Signed/pinned versions; drift monitor; kill switch (C-07, C-12) | Low–Med | Plugin governance owner | quarterly |
| R-041 | **Aggregate:** combined fleet access to education records exceeds any single agent's footprint | Students (minors), district | Medium / High → High | Aggregate-access dashboard; AIMS-owner review of combined data reach (C-22) | Medium | AIMS owner | quarterly |
| R-047 | Accountability gap: an action with no traceable owner/approval | District, students | Low / Medium → Med | Registry "not in registry = not authorized"; authority traces to approval (C-05) | Low | AIMS owner | quarterly |

### Artifact 5 — Control catalog

Each control: ID, framework reference, architectural realization, tiers it
applies to, owner, and the **evidence source**.

| Control ID | Framework ref | Realization | Applies to tiers | Owner | Evidence source |
| --- | --- | --- | --- | --- | --- |
| C-01 | 42001 Annex A (life cycle); NIST MANAGE | Gated promotion pipeline + eval suite | All | Release manager | Pipeline config; eval results per release |
| C-05 | 42001 Cl. 5 / Annex A (roles); NIST GOVERN | Agent registry with named owner; not-in-registry = not authorized | All | AIMS owner | Registry; authorization-deny logs |
| C-07 | NIST MANAGE; 42001 Cl. 10 | Fleet-wide kill switch | T3–T4 | Incident commander | Drill records; revocation logs |
| C-12 | 42001 Annex A (third party) | Plugin registry + signing + scope enforcement (exercise-02) | All | Plugin governance owner | Approval + signature-verification + scope-block logs |
| C-15 | 42001 Cl. 6; GDPR Art. 22 / FERPA | HITL gate on consequential actions; anti-rubber-stamp surface | T4 (T3 partial) | Agent owner | Approval logs with human identity + decision |
| C-18 | 42001 Annex A (use of AI); NIST MANAGE | Crisis detection + named-human escalation | T4 | Safety lead | Escalation runbook; crisis-routing logs + latency |
| C-20 | NIST MEASURE; 42001 Cl. 9 | Grounded retrieval + output claim-vs-source check + defer | T3–T4 | Quality/eval lead | Verification logs; gated-assertion + defer counts |
| C-22 | 42001 Cl. 9 (perf. eval); NIST MEASURE | Aggregate-access dashboard reviewed by AIMS owner | All (fleet) | AIMS owner | Dashboard; management-review minutes |
| C-25 | 42001 Annex A (data for AI); FERPA/COPPA | Field-level scoping + purpose tag + tagged lineage | T2–T4 | Data governance lead | Lineage records; scoping config |

### Artifact 6 — Incident & change process

**Change-control gates** (no silent promotion to production):

```text
new agent   -> impact assessment + tier assignment + owner sign-off (tier-appropriate)
               + registry entry
new version -> eval pass vs. baseline + owner sign-off + registry version pin update
new plugin  -> exercise-02 evaluate->approve gate + registry entry
tier change -> AIMS-owner sign-off (Accountable per RACI) + control-set update
```

**Incident runbook** (maps to NIST MANAGE — respond and recover):

```text
detect  -> triage -> contain (pause agent / revoke plugin / fleet kill switch)
        -> eradicate -> recover -> review (post-incident)
```

**Kill switch:** fleet-wide, fast, logged, and **DRILLED** — an undrilled kill
switch is a hope, not a control. Drill cadence is quarterly with a measured
mean-time-to-revoke; the `sis-writeback` runbook step (exercise-02) is one
instance.

**Continual improvement (PDCA, 42001 Cl. 10):** incident findings and audit
findings feed corrective actions back into controls and these specs; corrective
actions are tracked to closure and reviewed at the management review.

**Off-cycle review triggers:** a Tier 4 safety incident, a new regulation or
district-contract change, a failed kill-switch drill, or a new aggregate risk
crossing threshold on the C-22 dashboard.

### Aggregate-risk note (the capstone judgment)

No single registry row captures R-041: `tutor`, `progress-summarizer`, and
`lesson-drafter` are each individually scoped, but their **combined** reach into
education records is a larger blast radius than any one. It is surfaced by the
C-22 aggregate-access dashboard (total regulated-data access and action volume by
tier), owned by the AIMS owner, and reviewed at the quarterly management review —
the one place the fleet is governed as a whole rather than agent by agent. The
highest-leverage fleet-wide control is **C-07, the kill switch**, because it has
the largest shared blast radius: one action contains the entire fleet.

## Meeting the acceptance criteria

- **All six artifacts exist and are populated for the four-agent fleet, each living
  and owned.** Registry, tier model, RACI, risk register, control catalog, and
  incident/change process are all present with named owners and a review cadence.
- **Registry has a named accountable owner per agent and the not-in-registry rule.**
  Each agent row names a role owner; C-05 states the runtime authorization rule.
- **Tier model is proportionate and the four agents are placed with justification.**
  Tier 4 (`tutor`) is gated hard, Tier 1 (`usage-reporter`) moves fast, with
  per-agent reasoning.
- **RACI has exactly one Accountable per activity, named as a role.** Six
  activities, each with a single A.
- **Risk register makes residual risk explicit and owned, includes a minor-affecting
  risk; control catalog has an evidence column and correct framework refs.** Seven
  risks (R-014/R-023 affect minors; R-041 is the aggregate risk); nine controls
  each with framework ref + evidence.
- **Incident/change process has defined gates and a drilled fleet-wide kill switch,
  with a continual-improvement loop.** Change gates, the detect→review runbook, a
  drilled kill switch with MTTR, and the PDCA loop are specified.

## Common pitfalls

- **A registry that is a wiki, not a runtime check.** If the runtime does not
  enforce "not in registry = not authorized," the registry is documentation, not
  a control. C-05 makes it load-bearing.
- **Uniform heavy governance.** Gating Tier 1 like Tier 4 is over-governance that
  trains teams to misclassify down to escape it. Proportionality (and auditing a
  sample of self-assigned tiers) keeps the model honest.
- **A committee in the Accountable column.** "The governance board is accountable"
  diffuses accountability into no accountability. Exactly one role with a person.
- **A control catalog with no evidence column.** A control you cannot evidence is
  aspirational; the evidence column is what an auditor reads first and what
  produces a per-tenant statement of applicability.
- **Missing the aggregate view.** Governing agent-by-agent hides fleet-level risk
  (combined data access, total action volume). Without an owned aggregate
  dashboard, R-041-class risks have no home.

## Verification

- **Six-artifact completeness.** Confirm all six artifacts are present and
  populated for all four agents, each with an owner and a review cadence.
- **Registry rule.** Confirm the not-in-registry = not-authorized rule is stated
  and backed by a control (C-05), not just narrated.
- **Tier proportionality.** Confirm Tier 4 carries mandatory HITL + legal sign-off
  and Tier 1 auto-approves, and that each agent's placement is justified.
- **One Accountable per RACI row.** Confirm every activity has exactly one A,
  named as a role.
- **Evidence + framework refs.** Confirm every control-catalog row has an evidence
  source and a framework reference, and re-check the 42001/NIST references.
- **Drilled kill switch + aggregate risk.** Confirm the kill switch is drilled with
  an MTTR and that at least one risk (R-041) and one control (C-22) own the
  aggregate fleet view.
