# mod-309-governance-compliance-domain/exercise-01-iso-42001-controls-mapping — Solution

This is a reference governance artifact, not code. It maps ISO/IEC 42001:2023
clauses and Annex A control areas onto the fictional **TutorMesh** agentic
platform, crosswalks three rows to the NIST AI RMF, and includes an AI system
impact assessment for the parent-facing progress-summary agent. Clause and Annex
references are stated against the published structure of ISO/IEC 42001:2023 and
NIST AI 100-1; the normative ISO text is paywalled, so verify against the
standard itself before relying on this for a real audit.

> **Not legal or audit advice.** This is a teaching reference. A real AIMS scope,
> statement of applicability, and impact assessment must be reviewed by the
> accountable owner and, where regulated data is involved, by counsel.

## Approach

The skill being graded is *translation*: turning a management-system clause into
a thing an auditor can point at in the architecture, with a named owner and an
inspectable evidence source. The method, applied row by row:

1. **Scope first (Clause 4).** You cannot map controls onto a system you have not
   bounded. Name the agents, data classes, and interested parties — minors and
   districts explicitly — because the scope statement decides which Annex A areas
   are even in play.
2. **Risk before controls (Clause 6).** Run a lightweight risk assessment drawn
   from the NIST GenAI Profile (NIST AI 600-1) categories. The risk register is
   what justifies each Annex A inclusion and each exclusion — ISO/IEC 42001 is
   risk-based, so "include everything" is itself a finding.
3. **Statement of applicability.** For each Annex A control area, include with a
   reason or exclude with a reason. A reasoned exclusion is a governance act, not
   a gap.
4. **One row per mapped requirement.** Every row resolves a clause/Annex area to a
   real TutorMesh component, the risk it treats, an owner role, and a concrete
   evidence source. A row with an empty evidence cell is aspirational and does
   not count.
5. **Crosswalk, don't duplicate.** Show that operating with the NIST RMF and
   certifying against ISO/IEC 42001 can share controls — the same control emits
   the same evidence under two framework names.
6. **Impact assessment for the highest-stakes output.** The progress-summary
   agent writes parent-facing text about a minor; it gets its own one-page AI
   system impact assessment tying each harm to a mitigating control already in
   the mapping table.

The deliverable is a controls-mapping table an auditor could read and a team
could build against — not a policy narrative.

## Reference solution

### AIMS scope statement (Clause 4)

> The TutorMesh AI Management System covers the four production agents — `tutor`
> (student-facing tutoring), `lesson-drafter` (teacher-facing lesson-plan
> drafting), `progress-summarizer` (writes notes to the teacher dashboard and
> drafts parent-facing summaries), and `usage-reporter` (read-only de-identified
> internal analytics) — together with the shared curriculum knowledge base, the
> plugin registry and its admitted plugins, the base safety prompt, and the
> managed cloud inference endpoint used by all agents. In-scope data classes:
> student education records (FERPA), personal information of children under 13
> (COPPA), teacher account data, and de-identified usage telemetry. **Interested
> parties:** students including minors under 13, parents/guardians, teachers and
> district staff, contracting school districts (the FERPA data controllers), the
> managed-inference sub-processor, plugin authors, and the U.S. Department of
> Education and FTC as relevant regulators. **Out of scope:** the district's own
> student information system of record (governed by the district), and any
> non-production sandbox tenant that holds only synthetic data.

### Lightweight risk assessment (Clause 6 input)

Risks drawn from the NIST GenAI Profile (NIST AI 600-1) categories. Likelihood
and impact are rated Low/Medium/High; composite is the governing rating.

| Risk ID | Description (GenAI Profile category) | Affected parties | Likelihood | Impact | Composite |
| --- | --- | --- | --- | --- | --- |
| R-01 | Confabulation: tutor asserts an ungrounded fact a child believes | Students (minors) | Medium | High | High |
| R-02 | Data leakage: education-record PII egresses to an external endpoint (e.g., a plugin) | Students, district | Medium | High | High |
| R-03 | Information security / secondary use: child data reused for model training, analytics-for-resale, or another student's context | Students (minors) | Low | High | High |
| R-04 | Harmful content to a minor: self-harm, grooming, bullying signals mishandled | Students (minors) | Low | High | High |
| R-05 | Value-chain / third-party: an unvetted or drifted plugin runs with agent authority | Students, district | Medium | High | High |
| R-06 | Automation-amplified error: a wrong progress summary reaches a parent and affects a child's record | Students, parents | Medium | Medium | Medium |
| R-07 | Accountability gap: an agent takes an action with no traceable owner or approval | District, students | Low | Medium | Medium |
| R-08 | Confabulation in lesson drafts: factually wrong teacher-facing material | Teachers, students | Medium | Low | Low-Medium |

### Statement of applicability (Annex A control areas)

ISO/IEC 42001:2023 Annex A is a reference control set; each area is selected by
the risk assessment above. (Annex A area titles are paraphrased from the
published Annex A structure — verify exact titles against the standard.)

| Annex A control area | Decision | Reason |
| --- | --- | --- |
| Policies for AI | INCLUDED | Treats R-07; an AI policy is the leadership artifact (Clause 5) every control hangs from. |
| Internal organization (roles, responsibilities) | INCLUDED | Treats R-07; named accountable owners are required for a minor-serving fleet. |
| Resources for AI systems (data, tooling, compute, human) | INCLUDED | Treats R-02/R-03; data and compute resources carry regulated data and must be inventoried. |
| AI system impact assessment | INCLUDED | Treats R-01/R-04/R-06; mandatory for a fleet acting on minors. |
| AI system life cycle | INCLUDED | Treats R-01/R-08; gated promotion + eval is the core change control. |
| Data for AI systems | INCLUDED | Treats R-02/R-03; provenance, minimization, and lineage tagging for education records. |
| Information for interested parties | INCLUDED | Treats R-04/R-06; transparency to children, parents, and districts. |
| Use of AI systems | INCLUDED | Treats R-01/R-04; responsible-use and HITL constraints on student-facing output. |
| Third-party and customer relationships | INCLUDED | Treats R-05; plugin governance and the managed-inference sub-processor live here. |

No Annex A area is excluded outright for the production fleet, because every area
is exercised by at least one High/Medium risk. **Per-agent exclusions are
recorded instead at the tier level** (see the over-governance note below): the
Tier 1 `usage-reporter` excludes the HITL-on-consequential-actions control (A —
use of AI systems) on the reasoned ground that it takes no consequential action —
it is read-only over de-identified data. That exclusion is the auditable
"reasoned exclusion."

### Controls-mapping table (core deliverable)

Owner roles are the accountable role (the "A" in a RACI), not a team. Every row
names a real TutorMesh component and a concrete evidence source.

| Framework requirement (clause / Annex A area) | Architectural realization | Risk(s) treated | Owner (role) | Evidence source |
| --- | --- | --- | --- | --- |
| Cl. 5 — Leadership / AI policy | Published TutorMesh AI policy; AIMS owner role staffed and resourced | R-07 | AIMS owner | Signed policy doc with version + review date; org chart entry |
| Cl. 6 — AI risk treatment | Tool-permission tiers; consequential tools (`sis-writeback`, parent-send) require a HITL approval gate | R-02, R-06 | Head of AI platform | Risk register; gate config; HITL approval logs |
| Cl. 6 — AI system impact assessment | Per-agent impact assessment, mandatory before a Tier 3/4 agent ships | R-01, R-04, R-06 | Agent owner | Completed, versioned impact assessments |
| Cl. 9 — Performance evaluation / monitoring | Eval harness + observability traces with per-action attributes; per-agent quality dashboards | R-01, R-08 | Quality/eval lead | Dashboards; trace store; eval-regression history |
| Cl. 9.3 — Management review | Quarterly AIMS management review of this mapping, the risk register, and incident findings | R-07 | AIMS owner | Dated management-review minutes + action log |
| Cl. 10 — Improvement / corrective action | Incident findings → corrective actions → control updates (PDCA loop) | R-04, R-05 | AIMS owner | Corrective-action records linked to incidents |
| Annex A — AI system life cycle | Gated promotion pipeline (eval → staging → prod) with release sign-off and pinned model/prompt versions | R-01, R-08 | Release manager | Pipeline definition; eval-suite results per release; sign-off records |
| Annex A — Data for AI systems | Field-level scoping + purpose-tag "instructional" + tagged lineage at the retrieval boundary | R-02, R-03 | Data governance lead | Lineage records; scoping-layer config; redaction counts |
| Annex A — Third-party / supplier | Plugin registry with evaluate→approve gate and signed manifests; managed inference under a DPA as sub-processor | R-05 | Plugin governance owner | Plugin approval + signature-verification logs; signed DPA |
| Annex A — AI system impact assessment | Impact-assessment register keyed to each agent; gating Tier 3/4 launch on a completed assessment | R-04, R-06 | Agent owner | Impact-assessment register with refs per agent |
| Annex A — Information for interested parties | Age-appropriate AI-interaction disclosure to students; transparency notice to parents/districts | R-04, R-06 | Product owner (tutor) | Disclosure copy + render config; district transparency notice |
| Annex A — Use of AI systems | Crisis-signal detection routing to a named human; HITL on grade/record/IEP-affecting output | R-04 | Safety lead | Escalation runbook; crisis-routing logs; HITL approval logs |

### NIST AI RMF crosswalk (three rows)

Demonstrates that operating with the RMF and certifying against ISO/IEC 42001 can
share controls. NIST AI 100-1 functions: GOVERN / MAP / MEASURE / MANAGE.

| Mapping row | NIST AI RMF function / subcategory (illustrative) | Shared control / evidence |
| --- | --- | --- |
| Cl. 9 — Performance evaluation / monitoring | **MEASURE** (analyze, benchmark, and monitor mapped risks) | One eval harness + trace store serves both the 42001 monitoring clause and the RMF MEASURE function. |
| Annex A — Third-party / supplier | **GOVERN** (accountability and oversight) + **MAP** (third-party context) | The plugin registry's approval records are evidence for both the 42001 third-party control and RMF GOVERN/MAP. |
| Cl. 6 — AI risk treatment (HITL gate) | **MANAGE** (respond to and act on measured risks) | The HITL approval log evidences both 42001 risk treatment and RMF MANAGE. |

> NIST publishes an ISO/IEC 42001 crosswalk in the AI RMF Knowledge Base; the
> mappings above are illustrative and should be confirmed against that crosswalk
> for a certification effort.

### AI system impact assessment — progress-summary agent (one page)

**System under assessment.** `progress-summarizer`: reads a student's recent
tutoring activity and education-record fields, drafts a parent-facing progress
summary, and writes a note to the teacher dashboard. Output is parent-facing and
concerns a minor — the highest-consequence text the fleet produces.

**Affected individuals.** Students (minors, some under 13), their
parents/guardians who receive the summary, and teachers who forward it.

**Potential harms.**

- *Inaccurate characterization of a child* (confabulation, R-01/R-06): a wrong or
  unfair summary reaches a parent and shapes how a child is treated.
- *Disclosure beyond purpose* (R-02/R-03): education-record detail leaks to an
  unauthorized party or an external endpoint.
- *Over-reliance / automation bias* (R-06): a teacher rubber-stamps a draft they
  did not really read, and an automated characterization becomes a record.
- *Loss of agency / transparency* (R-04): the parent does not know an AI drafted
  the summary, or cannot contest it.

**Mitigating controls (each ties to a row above).**

- Confabulation/over-reliance → **HITL gate before parent send** (Cl. 6; Annex A —
  use of AI systems): a named teacher approves/edits/rejects; identity + decision
  logged. The review surface shows the claim, its source, and a confidence
  signal so the loop is real, not theater.
- Disclosure beyond purpose → **field-level scoping + purpose tag + tagged
  lineage** (Annex A — data for AI systems); no egress path to marketing/training.
- Transparency → **AI-interaction disclosure to parents** (Annex A — information
  for interested parties).
- Accountability → **named agent owner** in the registry (Cl. 5; Annex A —
  internal organization); every send traces to the approving human.

**Residual risk.** Low–Medium, owned by the `progress-summarizer` agent owner,
reviewed quarterly at the management review (Cl. 9.3).

## Meeting the acceptance criteria

- **Scoped AIMS statement + reasoned statement of applicability.** The scope
  statement names agents, data classes, minors, and districts; the SoA includes
  each Annex A area with a reason and records a reasoned per-tier exclusion (the
  Tier 1 `usage-reporter` HITL exclusion) — not "include everything."
- **Controls-mapping table with at least 10 rows, each naming a concrete component
  and a concrete evidence source.** The table has 12 rows; every row names a real
  TutorMesh component and an inspectable evidence artifact.
- **At least three rows carry a correct NIST AI RMF crosswalk.** Three rows are
  crosswalked to MEASURE, GOVERN/MAP, and MANAGE with the shared evidence named.
- **Impact assessment names affected minors/parents and ties each harm to a
  mitigating control in the table.** The progress-summary assessment names minors
  and parents and links every harm to a table row.
- **No vague-commitment rows.** Every row resolves to a component and an
  evidence source; there are no "committed to…" entries.

## Common pitfalls

- **Mapping a clause to a policy instead of a component.** "We have a data-privacy
  policy" is not a control. The row must name the *thing that enforces it* (the
  scoping layer, the gate) and the *evidence it emits*. If columns 2 and 5 do not
  change, the clause was not translated.
- **Citing a clause number wrong.** Stating "Clause 8 is risk assessment" when it
  is operation undermines the whole artifact. Verify clause titles and Annex A
  area names against ISO/IEC 42001:2023 itself; the chapter and resources give
  the structure but the standard is authoritative.
- **Including every Annex A control "to be safe."** ISO/IEC 42001 is risk-based;
  an unjustified inclusion is as much a finding as a gap. Tie each inclusion to a
  risk ID and record reasoned exclusions.
- **An empty evidence column.** A control with no inspectable artifact is
  aspirational. Every row needs a log, record, config, or signed document an
  auditor could actually request.
- **Naming a committee as the owner.** Accountability that names a team is not
  accountability. The owner is a role with one person behind it.

## Verification

This solution is a governance artifact; "verification" means structural and
factual self-checks, not a test run.

- **Row completeness.** Confirm the controls-mapping table has at least 10 rows and
  that every row has all five columns filled, including evidence.
- **Scope coverage.** Confirm the table covers Clause 6, Clause 9, Clause 10, and
  Annex A areas for life cycle, data for AI systems, third-party relationships,
  and AI system impact assessment (the minimum the exercise names).
- **Crosswalk count.** Confirm at least three rows carry a NIST AI RMF function.
- **Reasoned SoA.** Confirm the statement of applicability has at least one
  reasoned exclusion, not blanket inclusion.
- **Reference accuracy.** Re-read each clause/Annex A reference against ISO/IEC
  42001:2023 and NIST AI 100-1; correct any mismatch before use.
- **Impact-assessment linkage.** Confirm every harm in the progress-summary
  assessment maps to a control row by name.
