# mod-309-governance-compliance-domain/exercise-02-plugin-lifecycle-governance — Solution

This is a reference governance design — schemas, manifests, registry rows, gate
policies, and a runbook — not running code. It specifies the
evaluate → approve → version → monitor → revoke lifecycle for the **TutorMesh**
plugin platform and works the three queued plugins (`math-solver`,
`sis-writeback`, `translate-connector`) through it.

> **Not legal advice.** Plugin governance touching regulated student data also
> implicates FERPA/COPPA obligations (see exercise-03). Treat the data-handling
> review here as the gate that triggers that work, not a substitute for it.

## Approach

A plugin is new code running with the agent's authority. Govern it as a supplier
relationship *and* a life-cycle artifact at once. The design follows the five
lifecycle stages, each a gate or a control rather than a courtesy:

1. **Manifest as the contract.** Nothing is reviewed against intentions — every
   submission declares its tools, data classes, egress, requested scope, and a
   signed version/hash in a machine-readable manifest. Least privilege is the
   default; a plugin that asks for more scope than its function needs is scoped
   down or rejected at this gate.
2. **Evaluate to a rating + recommended scope**, not pass/fail. Provenance sets
   the depth of review; capability sets the risk. A write to a system of record
   and an external egress of student-derived text are structurally higher-risk
   than a stateless read, regardless of who submitted them.
3. **Tiered approval**, recorded, with conditions enforced by the registry rather
   than by trust. Match approval rigor to the rating so low-risk plugins are not
   gated like high-risk ones (over-governance drives contributors to route
   around the process).
4. **Version by signed hash.** Approval is of a specific signed version; any
   change re-enters Evaluate. The standing exception — a remote backend that can
   change behavior with no version bump — is named and monitored harder.
5. **Monitor at runtime and revoke fast.** Scope is enforced at call time and
   logged; drift against the evaluation baseline triggers re-evaluation; a
   fleet-wide, fast, logged kill switch is the containment control, and it is
   drilled.

## Reference solution

### 1. Plugin manifest schema

Submitted per plugin, per version. This is a governance schema, not code.

```text
plugin_id:        <stable id>
version:          <semver>
content_hash:     <sha-256 of the signed artifact>
signature:        <detached signature over content_hash, by author key>
provenance:       first_party | vetted_partner | open
declared_tools:   [ { name, mode: read|write, target_system }, ... ]
data_classes:     [ student_education_record | child_pii | none | ... ]
external_egress:  [ { endpoint, data_leaving, region }, ... ]
requested_scope:  <least-privilege scope statement>
remote_backend:   none | { endpoint, can_change_without_version_bump: bool }
deletion_support: yes | no    # can it honor a per-subject deletion request?
```

#### Filled manifests for the three queued plugins

```text
# math-solver  (OPEN submission)
plugin_id:        math-solver
version:          1.2.0
content_hash:     sha256:<...>
signature:        <author-key sig>
provenance:       open
declared_tools:   [ { name: solve_expression, mode: read, target_system: external_compute_api } ]
data_classes:     [ none ]            # takes a math expression, not student PII
external_egress:  [ { endpoint: compute.example.com, data_leaving: math_expression, region: US } ]
requested_scope:  call external_compute_api with a math expression; no record access
remote_backend:   { endpoint: compute.example.com, can_change_without_version_bump: true }
deletion_support: n/a                 # holds no subject data
# LEAST-PRIVILEGE FLAG: submission v1.2 requested read access to the student
#   context "for personalization." Denied — the function needs only the
#   expression string. Scope reduced to expression-only.
```

```text
# sis-writeback  (FIRST-PARTY)
plugin_id:        sis-writeback
version:          3.1.0
content_hash:     sha256:<...>
signature:        <first-party-key sig>
provenance:       first_party
declared_tools:   [ { name: write_progress_note, mode: write, target_system: student_information_system } ]
data_classes:     [ student_education_record ]
external_egress:  [ ]                 # no external egress; writes internal SoR
requested_scope:  write a progress note to the single student in the active session;
                  scoped to the requesting district tenant
remote_backend:   none
deletion_support: yes                 # notes are deletable per subject
# Higher risk: WRITE to a system of record holding education records.
```

```text
# translate-connector  (VETTED PARTNER)
plugin_id:        translate-connector
version:          1.4.0
content_hash:     sha256:<...>
signature:        <partner-key sig>
provenance:       vetted_partner
declared_tools:   [ { name: translate_text, mode: read, target_system: external_translation_api } ]
data_classes:     [ student_education_record ]   # translates agent output derived from records
external_egress:  [ { endpoint: mt.partner.example, data_leaving: agent_output_text, region: US } ]
requested_scope:  send agent output text for translation; return translated text
remote_backend:   { endpoint: mt.partner.example, can_change_without_version_bump: true }
deletion_support: contractual         # partner must not retain; enforced via DPA
# Higher risk: EXTERNAL EGRESS of potentially student-derived text to a third party.
```

### 2. The evaluation gate

What Evaluate checks, and what it outputs per plugin.

| Check | What it inspects |
| --- | --- |
| Declared-scope vs. function | Is requested scope minimal for the declared tools? (least privilege) |
| Security review | Static analysis, dependency/supply-chain scan, secret scan, egress review; sandbox dynamic analysis for higher tiers |
| Behavioral eval (sandbox) | Runs an eval suite: does it do what it claims, refuse what it should, stay in scope? Captures a baseline to diff against later |
| Data-handling review | If it touches regulated data: minimization, region pinning, DPA, deletion support (inherits exercise-03 obligations) |

Output is a **risk rating** and a **recommended scope** (not pass/fail):

| Plugin | Trust tier | Capability | Risk rating | Recommended scope |
| --- | --- | --- | --- | --- |
| `math-solver` | open | stateless read, external compute, no PII | Medium (open provenance + remote backend, but no regulated data) | Expression-only; no student-context read; deny the requested context access |
| `sis-writeback` | first-party | write to a system of record holding education records | High (irreversible write to SoR) | Write a single progress note for the in-session student, in the requesting tenant; HITL approval upstream |
| `translate-connector` | vetted partner | external egress of student-derived text | High (regulated-data egress to a third party) | Translate agent output only; partner under DPA, no-retention; US region pinned; monitor backend drift |

Justifying the tier differences: provenance changes the *depth of review*
(`math-solver` gets the most adversarial security/sandbox scrutiny because it is
anonymous), but capability changes the *risk rating*. `sis-writeback` is
first-party yet rated High because a write to a system of record is irreversible;
`translate-connector` is a vetted partner yet rated High because student-derived
text leaves the boundary. A trusted author does not lower the risk of a dangerous
capability.

### 3. The approval gate and tiering

| Approval tier | Policy | Plugins |
| --- | --- | --- |
| Auto-approve under policy | First-party, read-only, no regulated data, no external egress | (none of the three; reserved for low-risk reads) |
| Named-owner sign-off | Any write, or any regulated-data read with no external egress | `sis-writeback` |
| Legal/privacy review + named-owner sign-off | External egress of regulated/derived data, or open-provenance external egress | `translate-connector`; `math-solver` (open + external egress) |

An **approval record** contains: plugin_id, approved version + hash, granted
scope (the recommended scope, possibly tightened), conditions, named approver,
review/expiry date, and the linked evaluation rating. Conditions are enforced by
the registry, not by trust — for example:

```text
approval_ref:  AP-2026-0142
plugin_id:     translate-connector @ 1.4.0 (hash sha256:<...>)
granted_scope: translate agent_output_text only; US region; partner under DPA no-retention
conditions:    expires 2027-06-01; re-eval on any version bump; backend-drift monitor required
approver:      Plugin governance owner (J. Rivera)
```

### 4. Versioning and the registry

**Registry schema (source of truth at runtime):**

```text
plugin_id | approved_version + hash | scope_grant | owner | risk_rating |
approval_ref | status (active | deprecated | revoked) | remote_backend_flag
```

Filled rows:

| plugin_id | approved version + hash | scope_grant | owner | risk | approval_ref | status | remote backend |
| --- | --- | --- | --- | --- | --- | --- | --- |
| math-solver | 1.2.0 / sha256:… | expression-only, external compute | Plugin gov. owner | Medium | AP-2026-0138 | active | yes (monitored) |
| sis-writeback | 3.1.0 / sha256:… | write 1 note, in-session student, per tenant | SIS integration owner | High | AP-2026-0140 | active | no |
| translate-connector | 1.4.0 / sha256:… | translate agent output, US, no-retain | Plugin gov. owner | High | AP-2026-0142 | active | yes (monitored) |

**Signing and pinning.** The runtime loads only a plugin version whose content
hash matches the registry and whose signature verifies against the author key.
An unsigned or hash-mismatched artifact **fails closed** — it does not load.

**Version bump walk — `translate-connector` ships v2.0:**

```text
1. v2.0 submitted with new content_hash + signature + updated manifest.
2. Registry still pins 1.4.0; v2.0 is NOT loadable yet (hash mismatch -> fail closed).
3. v2.0 re-enters EVALUATE (full gate; diff behavior against the 1.4 baseline).
4. EVALUATE -> rating + recommended scope; if egress/data_classes changed, legal/privacy re-review.
5. APPROVE -> new approval record AP-...; conditions re-stated.
6. Registry updated: approved_version -> 2.0.0 + new hash. 1.4.0 -> deprecated (window) or revoked.
7. Runtime now loads only 2.0.0; 1.4.0 stops loading on its deprecation date.
```

**Standing risk — remote backend that changes without a version bump.** Both
`math-solver` and `translate-connector` call a remote endpoint whose behavior can
change with no version bump and therefore no re-evaluation trigger. This is
flagged `remote_backend_flag = yes (monitored)` and handled by Monitor: a
synthetic-probe canary runs the evaluation baseline against the live backend on a
schedule, and any output-quality or egress drift from baseline raises an alert
and triggers re-evaluation. Where possible, prefer plugins whose behavior is
pinnable; where not, monitor harder and shorten the review cadence.

### 5. Monitoring and revocation

**Runtime controls:**

- *Scope enforcement.* The sandbox/permission layer enforces the granted scope at
  call time. An out-of-scope attempt is **blocked and logged as a signal** — an
  in-scope plugin should never attempt to exceed scope, so an attempt means drift
  or compromise.
- *Drift monitoring.* Per-plugin error rate, latency, and output-quality signals
  are tracked against the evaluation baseline; deviation triggers re-evaluation
  (this is what catches the `math-solver` v1.4 regression in the reflection).
- *Usage/data-access auditing.* Every invocation lands in the audit log: which
  plugin, version, scope, data class touched, by which agent, in which tenant.

**Revocation path.**

- *Graceful deprecation* gives downstream agents a migration window before a
  version stops loading.
- *Emergency kill switch* flips the registry status to `revoked`, immediately and
  fleet-wide, accepting breakage to stop harm. It is logged, attributed, and
  opens an incident record.

**Incident runbook step — revoke `sis-writeback` fleet-wide:**

```text
RUNBOOK STEP R-KILL-SIS  (trigger: confirmed bad write / compromise of sis-writeback)
1. Incident commander declares; opens incident record INC-<id>.
2. Set registry: sis-writeback status = revoked  (single source-of-truth flip).
3. Runtime control plane pushes registry delta; all agents stop loading/calling
   sis-writeback within the kill-switch SLA (target MTTR < 60s, drilled).
4. Verify: zero successful sis-writeback invocations after T0 in the audit log.
5. Log the revocation: who, when, why; link to INC-<id>.
6. Notify affected agent owners (progress-summarizer depends on it) -> degrade
   gracefully (queue notes for manual write) rather than fail open.
7. Post-incident: root cause -> corrective action -> re-evaluate before un-revoking.
```

## Meeting the acceptance criteria

- **Manifest schema filled for all three plugins, with at least one least-privilege
  reduction.** The schema is defined and filled three times; `math-solver`'s
  requested student-context read is denied and scoped to expression-only — the
  recorded reduction.
- **Justified risk ratings reflecting tier and capability.** `sis-writeback`
  (write to SoR) and `translate-connector` (regulated-data egress) are both rated
  High above `math-solver`'s read, with the tier-vs-capability reasoning made
  explicit.
- **Tiered approval, recorded, enforced by the registry.** Three approval tiers
  map the three plugins; the approval record carries conditions, and conditions
  live in the registry scope_grant/status the runtime consults.
- **Versioning by signed hash; re-evaluation on change; remote-backend drift
  monitored.** Load is hash + signature gated and fails closed; the v2.0 walk
  re-enters Evaluate; the remote-backend flag drives synthetic-probe drift
  monitoring.
- **Fleet-wide, fast, logged revocation with a written runbook step.** The
  `sis-writeback` runbook flips one registry status, propagates within a drilled
  MTTR, verifies via the audit log, and is attributed.

## Common pitfalls

- **Trust by allowlist alone.** "It's a vetted partner" is a name, not a control.
  `translate-connector` is still rated High and still scope-enforced and
  drift-monitored; provenance sets review depth, not the risk rating.
- **Approve once, run forever.** Without re-evaluation on version change and drift
  monitoring against a baseline, a once-safe plugin rots into a liability. The
  v2.0 walk and the `math-solver` regression case both depend on this.
- **Scope by convention.** "Plugins are expected not to write outside their area"
  with no runtime enforcement is a comment, not a boundary. Scope must be
  enforced at call time and out-of-scope attempts logged as signals.
- **No fast revocation path.** If pulling a bad plugin needs a redeploy, you do
  not have containment. Revocation must be a single registry flip that propagates
  fleet-wide within a drilled SLA.
- **Over-governing low-risk plugins.** Gating a first-party read-only plugin with
  the same legal review as a regulated-egress one trains contributors to route
  around the process — assign the auto-approve tier deliberately so the gate
  stays credible.

## Verification

- **Manifest completeness.** Confirm all three manifests fill every schema field
  and that at least one records a least-privilege reduction.
- **Rating consistency.** Confirm the write (`sis-writeback`) and the egress
  (`translate-connector`) outrank the read (`math-solver`), and the reasoning is
  capability-based, not provenance-based.
- **Registry-as-truth.** Confirm the fail-closed rule (unsigned/hash-mismatch does
  not load) and that approval conditions appear in registry fields, not prose.
- **Re-evaluation trigger.** Trace the v2.0 walk and confirm any change re-enters
  Evaluate and that the remote-backend flag has a monitoring control behind it.
- **Kill-switch drill.** Confirm the `sis-writeback` runbook is fleet-wide, single
  source-of-truth, logged, attributed, and has a measurable MTTR target.
