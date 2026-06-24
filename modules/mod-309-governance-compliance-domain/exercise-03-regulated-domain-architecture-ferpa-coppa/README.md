# mod-309-governance-compliance-domain/exercise-03-regulated-domain-architecture-ferpa-coppa — Solution

This is a reference architecture artifact: the FERPA/COPPA translation, a
data-flow diagram, an audit-architecture design, an accountability ADR, and a
compliance matrix for the **TutorMesh student-tutoring agent**. It is
architecture translation, not legal analysis — the citations are to primary
sources so the clause number is the evidence.

> **Not legal advice.** This treats statutory text at the architectural level;
> binding interpretation is counsel's job. State law (e.g., California's SOPIPA,
> Cal. B&P Code §§ 22584–22585) and district data-processing agreements add
> obligations beyond this exercise. Verify every citation against the current
> regulatory text on eCFR before relying on it.

## Approach

A regulation is a constraint on behavior; an architecture is components,
boundaries, and flows. The job is to translate each requirement into a boundary
on the diagram and a log you can produce on demand, using Chapter 2's five
questions — data, flow, action, control, evidence. A requirement that does not
change the flow (question 2) and the enforcing component (question 4) has not
been translated. The deliverables follow from that:

1. **Run the five-question method** on each FERPA and COPPA requirement, ending
   each in a concrete control and an inspectable evidence source.
2. **Draw the data-flow diagram** with every regulated boundary marked — and,
   critically, show the *absent* edge to marketing/profiling/training. An absent
   edge you have verified is itself a control.
3. **Design the audit architecture** to be tamper-evident and queryable by
   subject, and reconcile the tension between COPPA minimization/retention
   *ceilings* and audit-retention *floors* (log references and decisions, not raw
   child PII).
4. **Name an accountable owner and record the ADR** for autonomous read /
   no-external-disclosure; route consequential actions through a HITL gate
   captured in the same audit record.
5. **Produce the compliance matrix** — one row per requirement, each citing the
   primary source and naming a control, evidence, and owner.

The recurring insight the exercise is built around: FERPA's purpose-limitation
boundary and COPPA's no-commercial-use boundary collapse into the *same*
architectural boundary.

## Reference solution

### 1. The translation method (per requirement)

```text
Requirement: FERPA school-official exception (34 CFR 99.31(a)(1))
  1. WHAT DATA?    Student education records (grades, IEP data, identifiers).
  2. WHAT FLOW?    District SIS -> scoping/redaction -> agent context -> tutoring
                   output. No path to marketing, training, or another student.
  3. WHAT ACTION?  read_student_record, scoped to the one student in session;
                   NO disclose_externally capability.
  4. WHAT CONTROL? Field-level scoping + purpose tag "instructional"; tool
                   whitelist excludes external disclosure; vendor under district
                   direct control via the DPA; region-pinned inference.
  5. WHAT EVIDENCE?Audit log of every record access by student/time/purpose;
                   the DPA naming the vendor as a school official.

Requirement: FERPA purpose limitation / no secondary use (34 CFR Part 99; 99.33)
  1. DATA  Education-record PII.
  2. FLOW  Stays inside the instructional purpose; never reaches training,
           marketing, analytics-for-resale, or a second student's context.
  3. ACTION Any read; the danger is a downstream re-use edge.
  4. CONTROL Tagged data lineage keyed to purpose "instructional"; an ABSENT edge
           (verified) from the regulated path to any training/marketing subsystem.
  5. EVIDENCE Lineage records; architecture review attesting the absent edge;
           egress-deny config.

Requirement: FERPA parental access / correction (34 CFR 99.10-99.12)
  1. DATA  The student's records, including any derived/stored copies.
  2. FLOW  Per-subject query across SIS-derived stores, agent memory, vector index.
  3. ACTION find_records(subject), return, correct/delete.
  4. CONTROL Per-subject indexing; a deletion/correction path that reaches agent
           memory and the vector index (both are regulated data stores).
  5. EVIDENCE Subject-rights request log; deletion/correction completion record.

Requirement: COPPA verifiable consent (16 CFR Part 312; FTC "COPPA and Schools")
  1. DATA  Personal information of a child under 13.
  2. FLOW  No collection proceeds until a recorded consent is present; school
           may provide consent ONLY for a solely-educational purpose.
  3. ACTION Any collection/processing of child PII.
  4. CONTROL Fail-closed consent/purpose check in the request path; absent or
           withdrawn consent -> deny.
  5. EVIDENCE Consent record (school-provided), keyed to student + purpose.

Requirement: COPPA no commercial profiling / behavioral advertising (16 CFR 312)
  1. DATA  Child PII / behavior.
  2. FLOW  Regulated path has NO connection to any ad/marketing/profiling system.
  3. ACTION Profiling/targeting capability — denied by design.
  4. CONTROL Hard boundary: absent edge to ad/profiling subsystems (verified),
           same boundary as FERPA purpose limitation.
  5. EVIDENCE Architecture attestation of the absent edge; deny-by-default egress.

Requirement: COPPA minimization & retention limits (16 CFR 312)
  1. DATA  Child PII.
  2. FLOW  Collect only what the educational function needs; delete on schedule
           and on request.
  3. ACTION read/store.
  4. CONTROL Field-level minimization at the retrieval edge; retention lifecycle
           policy; per-subject deletion.
  5. EVIDENCE Retention policy + deletion-job records; minimization config.
```

### 2. The data-flow diagram

```text
  ┌──────────────┐  raw records   ┌────────────────────┐ purpose-tagged,
  │ District SIS │ ─────────────▶ │ scoping + redaction │ minimized fields
  │ (SoR)        │                │ (field-level)       │
  └──────────────┘                └─────────┬───────────┘
                                            │
                         ┌──────────────────▼───────────────────┐
                         │ consent / purpose check (FAIL CLOSED) │  no/withdrawn
                         │ school-provided consent, "instruct."  │ ─────────────▶ DENY
                         └──────────────────┬───────────────────┘
                                            │ ok
                                            ▼
                                 ┌────────────────────┐
                                 │ agent context      │ ◀── only this student,
                                 │ (prompt + tools)    │     only needed fields
                                 └─────────┬──────────┘
                                            │ scoped tool calls
                                            ▼
                                 ┌────────────────────┐     ┌──────────────────┐
                                 │ tutoring output    │ ──▶ │ tamper-evident   │
                                 │ (to this student)  │     │ audit log (WORM) │
                                 └────────────────────┘     └──────────────────┘

  ABSENT EDGES (verified controls — no path exists):
     agent context  ──X──▶  marketing / behavioral advertising
     agent context  ──X──▶  commercial profiling
     education data ──X──▶  model training / fine-tuning
     student A context ──X──▶ student B context

  Regulated boundaries (rules attach here):
     [B1] SIS -> scoping        : minimization (FERPA/COPPA)
     [B2] consent/purpose check : lawful basis + purpose limit (COPPA + FERPA)
     [B3] tool whitelist        : no external disclosure (FERPA 99.31/99.33)
     [B4] egress deny-by-default: cross-border / sub-processor (DPA)
```

The absent edges are the heart of the design: a path that *could* let child data
reach marketing, profiling, or training is a violation waiting in the diagram, so
their verified absence is a control with its own evidence (an architecture
attestation plus deny-by-default egress config).

### 3. The audit architecture

**What each entry captures:** trace ID; timestamp; actor (agent + version);
subject *reference* (a tokenized student ID, not raw PII); action type
(read/write/tool call); the tool and arguments *summary*; purpose tag; consent
ref; result status; and the pinned model + prompt + plugin versions in effect.

**Tamper-evidence:** append-only / WORM storage with hash-chained entries; write
access to the log is separated from the agent's identity, so a compromised agent
cannot rewrite history.

**Retention:** to the regulatory horizon for disclosure recordkeeping; the log
lifecycle policy is itself a compliance artifact.

**Queryable:** indexed by subject reference, actor, action type, and time, so
"show every record access for student X in window Y" returns quickly.

**Minimization vs. retention reconciliation.** COPPA sets a retention *ceiling* on
child PII; auditability sets a retention *floor* on the record of actions. These
collide only if the log stores raw child PII. Resolution: **log references and
decisions, not raw child PII** — the audit entry holds a tokenized subject
reference and a description of the action/decision, while the underlying personal
data is deleted on its own schedule. The token resolves to the subject only
through a separately governed mapping that is itself subject to deletion, so the
audit trail survives PII deletion without re-exposing the child.

### 4. Accountability

**Accountable owner.** The `tutor` agent owner (a named role — e.g., "Tutoring
product owner") is the single accountable owner; it is recorded in the agent
registry (exercise-05), not buried in a wiki.

**Architecture decision record:**

```text
ADR-309-03: Autonomous read of education records; no external disclosure
Status:   Accepted
Context:  The tutoring agent must read the in-session student's education record
          to personalize tutoring for a child under 13, under the FERPA
          school-official exception and school-provided COPPA consent.
Decision: The tutor agent MAY read the in-session student's scoped, purpose-tagged
          education record autonomously (no per-read human gate). It MAY NOT
          disclose education-record data externally, write to a permanent record,
          or reach another student's context — these capabilities are absent by
          design and any attempt is a containment failure to alert on.
Rationale:Per-read HITL is disproportionate and unworkable at tutoring scale; the
          minimization + purpose-tag + absent-disclosure-edge controls bound the
          read. Disclosure and record-mutation are the consequential actions, and
          those are denied or gated, not the read.
Risk accepted: Residual risk of an over-broad read, mitigated by minimization and
          monitored via the audit log. Accepted by: Tutoring product owner.
```

**HITL gates.** Consequential actions — writing a progress note that becomes part
of a record, or any parent-facing disclosure (handled by `progress-summarizer`,
exercise-04) — route through a human approval gate, and the approving human's
identity and decision land in the *same* audit record as the action, making
accountability and auditability one artifact.

### 5. The compliance matrix

| Requirement (cite source) | Data class | Boundary / control | Evidence source | Owner |
| --- | --- | --- | --- | --- |
| FERPA school-official exception (34 CFR § 99.31(a)(1)) | Education record | Scoping + purpose tag "instructional"; vendor under district direct control; no `disclose_externally` tool | Access log by student/time/purpose; signed DPA | Tutoring product owner |
| FERPA purpose limitation / no secondary use (34 CFR Part 99; § 99.33) | Education record | Tagged lineage; verified absent edge to training/marketing/other-student | Lineage records; absent-edge architecture attestation; egress-deny config | Data governance lead |
| FERPA parental access / correction (34 CFR §§ 99.10–99.12) | Education record (incl. derived copies) | Per-subject indexing; deletion/correction path reaching agent memory + vector index | Subject-rights request log; deletion/correction completion record | Data governance lead |
| FERPA disclosure recordkeeping (34 CFR § 99.32) | Education record | Tamper-evident audit log of accesses/disclosures | WORM/hash-chained audit log | Tutoring product owner |
| COPPA verifiable consent (16 CFR Part 312; FTC "COPPA and Schools") | Child PII (< 13) | Fail-closed consent/purpose check in request path; school-provided consent for solely-educational purpose | Consent record keyed to student + purpose | Privacy owner |
| COPPA no commercial profiling / behavioral advertising (16 CFR Part 312) | Child PII / behavior | Hard absent edge to ad/profiling subsystems (same boundary as FERPA purpose limit) | Absent-edge attestation; deny-by-default egress | Privacy owner |
| COPPA minimization & retention (16 CFR Part 312) | Child PII | Field-level minimization at retrieval edge; retention lifecycle; per-subject deletion | Minimization config; retention policy + deletion-job records | Data governance lead |
| Cross-border / sub-processor (DPA; FERPA direct control) | Education record / child PII | Region-pinned inference + storage; managed model API treated as sub-processor under the DPA | Region config; signed DPA listing sub-processors | AIMS owner |

## Meeting the acceptance criteria

- **Every requirement ends in a control and an evidence source.** Each
  five-question translation and each matrix row terminates in a named control and
  an inspectable artifact, not a policy statement.
- **Diagram marks regulated boundaries and the verified absent edge.** Boundaries
  B1–B4 are marked and the four absent edges (marketing, profiling, training,
  cross-student) are shown as verified controls.
- **Audit is tamper-evident, queryable by subject, and the
  minimization-vs-retention tension is reconciled.** WORM + hash-chain, subject
  indexing, and the "log references/decisions, not raw child PII" reconciliation
  are all specified.
- **Named owner + ADR; consequential actions route through a HITL gate captured in
  the audit record.** The tutor owner is named, ADR-309-03 records the autonomous-
  read / no-disclosure decision, and write/disclosure actions are gated with the
  human decision logged alongside the action.
- **Matrix cites the primary source per row and names control, evidence, owner.**
  All eight rows carry a correct statutory/regulatory citation plus the three
  required columns.

## Common pitfalls

- **Translating to a policy, not a boundary.** "We will not re-use student data"
  with no tagged lineage and no verified absent edge is a promise, not a control.
  The flow and the enforcing component must change.
- **Forgetting agent memory and the vector index are regulated stores.** A
  deletion path that reaches the SIS but not the agent's memory or vector index
  leaves child PII behind and fails the parental-deletion right.
- **Logging raw child PII into the audit trail.** This pits the audit floor
  against the COPPA ceiling and makes deletion impossible. Log a tokenized
  reference and the decision, not the raw record.
- **Conflating FERPA and COPPA.** They are not interchangeable — FERPA governs
  education-record *disclosure*, COPPA governs *collection from under-13s*. A K-12
  agent must satisfy both at once; design to the union.
- **Miscaught citations.** Citing "§ 99.31" loosely or attributing the COPPA Rule
  to the wrong CFR title undercuts the evidence. The clause number is the
  evidence — get it exact and verify on eCFR.

## Verification

- **Translation completeness.** Confirm each requirement has all five questions
  answered and ends in a control + evidence (not a policy).
- **Absent-edge check.** Confirm the diagram explicitly shows the marketing,
  profiling, training, and cross-student edges as absent/verified.
- **Audit properties.** Confirm tamper-evidence (WORM/hash-chain), subject-indexed
  queryability, and the references-not-raw-PII reconciliation are all present.
- **Accountability artifacts.** Confirm a single named owner and an ADR exist, and
  that consequential actions route through a HITL gate logged in the audit record.
- **Citation accuracy.** Re-check every FERPA (20 U.S.C. § 1232g; 34 CFR Part 99)
  and COPPA (15 U.S.C. §§ 6501–6506; 16 CFR Part 312) reference against eCFR.
