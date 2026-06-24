# mod-309-governance-compliance-domain/exercise-02-multi-regime-regulated-domain-architecture — Solution

## Approach

The deliverable is a **convergence/divergence comparison table** that proves one
architecture serves four regulated regimes — **healthcare (HIPAA)**, **finance
(GLBA/SOX/PCI)**, **public sector**, and **edtech (FERPA/COPPA)** — without forking
the build. The governing discipline is **peer balance**: the four regimes are analyzed
at equal rigor, none is the default, and none is treated as a mere exception to
another. The same case-assistance agent is dropped into a hospital, a bank, an agency,
and a school district; the job is to find the *deltas*, not to rebuild it four times.

The choices that drive the analysis below:

- **Five primitives, four parameter profiles.** Privacy, residency, auditability,
  accountability, and retention are the same five questions in every regime. The regime
  differs only in the *answers* — which is exactly what makes a convergent core
  possible. Decomposing all four into the same primitive grid is what keeps them peers.
- **Convergence means same mechanism, different parameter.** A control converges when
  one mechanism serves all four and only a parameter changes (classification labels,
  retention schedule, gate authority). That is the build-once set.
- **Divergence splits two ways.** *Parameter-level* divergence is just a different value
  on the convergent core. *Structural* divergence is a control one regime needs that the
  others do not — a pluggable component (verifiable parental consent, card-data scope
  reduction, public-records disclosure). Calling a structural delta "just a parameter"
  is the failure mode; so is forking the whole architecture for it.
- **Stacking and conflict are surfaced, not silently resolved.** When two regimes land
  on one platform, per-class routing handles the stack. When two regimes genuinely
  conflict (a retention floor versus a deletion right on the same datum), the
  architecture *escalates* — it does not quietly pick a winner.

## Reference solution

### 1. Each regime decomposed into the five primitives

All four rows are kept at the same level of rigor. No regime is the baseline.

| Regime | Privacy (what's sensitive) | Residency | Auditability | Accountability (who decides) | Retention | Sector-specific extra |
| --- | --- | --- | --- | --- | --- | --- |
| Healthcare (HIPAA) | PHI: identifiable health data — diagnoses, treatment, identifiers tied to care. | Often contractual/region-pinned per BAA; data kept where the covered entity permits. | Access/disclosure accounting; who-saw/touched-what trail for PHI. | Covered entity / clinician authority for care-affecting actions; minimum-necessary access. | Multi-year per record/legal hold; long minimums common. | Minimum-necessary rule; breach-notification duty on PHI exposure. |
| Finance (GLBA/SOX/PCI) | NPI (nonpublic personal/financial info); **cardholder data** (PCI) as a sharply scoped sensitive class. | Per-institution policy; some data region-constrained by contract/regulator. | SOX integrity over financial records/changes; immutable evidence of controls. | Finance/control owner authority; segregation of duties on money-moving actions. | SOX/records retention multi-year; tax/legal floors. | **Card-data scope reduction** (tokenize/never-store PAN); SOX change-control evidence. |
| Public sector | Citizen PII; sometimes classified/controlled categories; varies by agency mandate. | Frequently **must stay in-jurisdiction** (government-region/cloud). | Open-records auditability *and* internal accountability; FOIA-style disclosure posture. | Agency-designated official authority; documented decision authority for actions on citizens. | Records-schedule governed; some records permanent, some scheduled destruction. | **Public-records disclosure** obligation (responsive records must be producible). |
| Edtech (FERPA/COPPA) | Education records (FERPA); personal data of a **minor** (COPPA). | Per-district/vendor agreement; data kept per the institution's terms. | Access logging on education records; parental access/inspection rights. | School/district authority; parent stands in for the minor on consequential actions. | District retention schedule; consent-bound — data tied to the consent that permitted it. | **Verifiable parental consent** for the minor; data-use limits for under-13. |

The grid reads the same way across all four: each regime answers the *same* five
questions plus one sector-specific extra. That symmetry is what proves they are peers.

### 2. The convergent core (build once, parameterize)

| Convergent control | Shared mechanism (build once) | Per-regime parameter that varies |
| --- | --- | --- |
| Data classification | Tag every datum with a sensitivity class at ingest. | Label set + the "what counts as identifiable/sensitive" rule (PHI / NPI+card / citizen-PII / education-record+minor). |
| Minimization | Strip/mask/tokenize at each boundary a datum should not cross in the clear. | Which classes minimize where (e.g., card data tokenized pre-store in finance; health identifiers masked into prompts in healthcare). |
| Immutable audit log | One append-only, tamper-evident log of consequential events. | Retention window + which events are "consequential" + disclosure posture (public-records vs. internal). |
| Accountability gate | HITL gate on consequential actions, with a named authority. | *Who* the authority is (clinician / control-owner / agency official / school + parent) and the consequence threshold. |
| Residency / region pinning | Pin each class to an allowed region set; log any crossing. | The allowed-region set (in-jurisdiction for public sector; contractual region for the others). |
| Retention | Per-class TTL with min/max and a destruction method. | The schedule values (multi-year floors differ; consent-bound vs. records-schedule). |
| Access control | Least-privilege / minimum-necessary access to each class. | The access rule's strictness and the role taxonomy per regime. |

Seven controls converge: one mechanism each, configured by a regime profile.

### 3. The divergent deltas

**Parameter-level divergence** (a value on the convergent core, not a new component):

- Allowed-region set (in-jurisdiction for public sector; contractual region elsewhere).
- Retention schedule values (healthcare/finance multi-year floors; edtech consent-bound;
  public-sector records-schedule with some permanent records).
- Identifiable/sensitive rule (PHI vs. NPI+cardholder vs. citizen-PII vs.
  education-record+minor).
- Gate authority identity and threshold (clinician / control-owner / agency official /
  school+parent).

**Structural divergence** (a pluggable component one regime needs and others do not):

| Structural delta | Regimes that activate it | Why it cannot be a parameter |
| --- | --- | --- |
| Verifiable parental consent | Edtech (COPPA, for a minor) | It introduces a *new actor* (the parent) and a *new pre-condition* (verified consent before processing) — a workflow and a verification mechanism, not a value on an existing control. |
| Card-data scope reduction (tokenize/never-store PAN) | Finance (PCI) | It changes the *data path*: PAN is tokenized before it can reach the store/prompt/logs. That is a new boundary component, not a stricter setting on an existing one. |
| Public-records disclosure | Public sector (FOIA-style) | It adds an *outbound* obligation — responsive records must be discoverable and producible to an external requester — a query/disclosure path the other regimes do not have. |
| Disclosure / breach-notification workflow | Healthcare (HIPAA breach rule); finance (GLBA-adjacent) | A triggered external-notification process; a workflow component, though it can be partly shared across the regimes that activate it. |

Each structural delta is a **pluggable component**: present for the regimes that need
it, absent (and inert) for the rest. The architecture does not fork — it admits or
omits a component per profile.

### 4. Convergence/divergence comparison table (headline deliverable)

| Control | Healthcare | Finance | Public sector | Edtech | Verdict |
| --- | --- | --- | --- | --- | --- |
| Data classification | PHI label set | NPI + cardholder label set | citizen-PII (+ controlled) label set | education-record + minor label set | **converge param** — one tagger, label set is the parameter |
| Minimization | mask health identifiers at boundaries | tokenize card data; mask NPI | mask citizen PII | mask minor PII | **converge param** — one minimize-at-boundary, which-class-where varies |
| Immutable audit log | long retention; access accounting | SOX integrity | public-records + internal | consent-bound retention | **converge** — one tamper-evident log, retention as parameter |
| Accountability gate | clinician authority | control-owner; segregation of duties | agency official | school + parent | **converge param** — one gate, authority/threshold is the parameter |
| Residency | contractual region | contractual region | **in-jurisdiction** | per-agreement region | **converge param** — one region-pin, allowed-set is the parameter |
| Retention | multi-year floor | SOX/tax floor | records-schedule | consent-bound | **converge param** — one per-class TTL, schedule is the parameter |
| Verifiable parental consent | n/a | n/a | n/a | **required** | **diverge-structural** — edtech-only component |
| Card-data scope reduction | n/a | **required** | n/a | n/a | **diverge-structural** — finance-only component |
| Public-records disclosure | n/a | n/a | **required** | n/a | **diverge-structural** — public-sector-only component |

Six controls converge; three are structural deltas, each attributed to the single
regime that activates it. That ratio is the headline: most of the platform is built
once.

### 5. Stacking and conflict

**Stacking (handled by per-class routing).** The agent serves a finance use case where
a record contains both *health* data (a medical claim) and *payment* data (the card
used). Both regimes apply on one platform. Per-class routing handles it: classification
tags the health datum as PHI and the card datum as cardholder data; the PHI follows the
healthcare minimization/retention parameters, the card data follows PCI's tokenize-and-
scope-reduce *structural* path. Two profiles act on two classes inside one run — no
fork, because the convergent core routes by class and the structural component (card
tokenization) activates only on the class that needs it.

**Conflict (surfaced and escalated, not silently chosen).** A subject exercises a
**deletion right** on a datum that a regime **retention floor** still requires to be
kept (e.g., a record under a multi-year retention minimum that the subject asks to
erase). These pull in opposite directions on the *same datum*. The architecture must
not silently pick "delete" (violating the retention floor) or "keep" (violating the
deletion right). Instead the conflict-resolution path:

```text
   on (deletion_request, datum D) where retention_floor(D) not yet elapsed:
     do NOT auto-resolve
     mark D conflicted  ──▶  emit escalation event to the accountable role
                             (per the active regime profile's gate authority)
     options surfaced: (a) honor deletion of the raw datum but retain a
       minimized audit reference; (b) defer deletion until the floor elapses,
       with a logged justification; (c) legal-hold override
     the chosen resolution is recorded in the audit trail with the decider
```

The point: the conflict becomes a *visible, owned decision* with an audit record — not
a default the architecture chose on its own. Separating the audit store from the raw
store (exercise-05) is what makes option (a) — erase the raw datum, keep a minimized
audit reference — possible without breaking either obligation.

### Stretch: fifth regime, regime-profile object, most-restrictive-wins failure

- **Adding GDPR as a fifth peer.** Most of the convergent core absorbs it by parameter:
  classification (add "special-category" labels), residency (EU allowed-region set),
  retention (storage-limitation schedule), accountability (controller/processor roles as
  gate authority). The new *structural* piece is the **data-subject-rights workflow**
  (access/portability/erasure on request) — but erasure largely reuses the right-to-
  erasure flow already built for the conflict case, so even the fifth regime is mostly
  parameters plus one mostly-shared component.
- **The regime-profile object** (the concrete configuration that tunes the core):

  ```text
  RegimeProfile {
    regime_id:            "healthcare" | "finance" | "public" | "edtech" | ...
    identifiable_rule:    predicate defining what counts as identifiable/sensitive
    class_labels:         [ ...sensitivity classes for this regime... ]
    allowed_regions:      { class -> [allowed region set] }
    retention_schedules:  { class -> { min, max, destroy_method } }
    gate_authority:       { action_class -> { role, threshold, approver_count } }
    structural_components: [ "parental_consent" | "card_scope_reduction" |
                             "public_records_disclosure" | ... ]  // activate these
  }
  ```

  Switching regimes swaps this object; nothing structural in the core changes.

- **Most-restrictive-wins producing the wrong answer.** A naive "apply the strictest
  parameter across stacked regimes" policy breaks on the deletion-vs-retention conflict:
  the strictest privacy stance says *delete now*, the strictest retention stance says
  *keep for years* — "most restrictive" is undefined because the two restrictions point
  opposite ways. Auto-defaulting to either violates the other. This is exactly the case
  that must **escalate** to the accountable role rather than resolve to an automatic
  stricter default; "most restrictive" is not a total order when two obligations
  conflict on the same datum.

## Meeting the acceptance criteria

- **All four regimes decomposed at equal rigor, none the default** — the five-primitive
  grid has one row per regime at the same depth, each with its sector-specific extra; no
  regime is framed as the baseline or as another's exception.
- **Convergent core identified with mechanism + varying parameter** — seven controls,
  each with its shared mechanism and the per-regime parameter that varies.
- **Divergence split into parameter vs. structural, structural deltas attributed** —
  parameter-level list plus a structural-delta table naming parental consent (edtech),
  card-data scope (finance), and public-records disclosure (public sector), each tied to
  its activating regime with a reason it cannot be a parameter.
- **Comparison table spans all four regimes and the listed controls with a verdict** —
  the nine-row synthesis table covers classification, minimization, audit log, gate,
  residency, retention, and the three structural deltas, with a verdict per row.
- **Stacking and conflict both handled** — stacking via per-class routing on a
  health+payment record; conflict via surface-and-escalate on deletion-vs-retention,
  not a silent choice.

## Common pitfalls

- **Treating one regime as the main one.** If the analysis reads as "healthcare, and the
  others are variations," it has failed the peer-balance discipline. All four answer the
  same five questions at the same depth.
- **Calling a structural delta a parameter.** Verifiable parental consent is not a
  stricter retention value — it adds a new actor and a pre-condition workflow. Forcing it
  into the convergent core hides a real component and under-builds the edtech profile.
- **Forking the architecture for a structural delta.** The opposite error: building a
  separate finance system because of PCI. Card-data scope reduction is one *pluggable
  component*, not a second platform.
- **Silently resolving a conflict.** Auto-deleting (breaking the retention floor) or
  auto-keeping (breaking the deletion right) hides a decision that must be owned. Surface
  and escalate it with an audit record.
- **Conflating the audit store with the raw store.** If they are one store, you cannot
  erase a subject's raw datum while keeping the audit trail intact — and the
  deletion-vs-retention conflict becomes unsolvable.

## Verification

A completed submission is correct when:

- The five-primitive table has all four regimes filled at equal rigor, each with a
  sector-specific extra, and no regime is privileged as the default.
- The convergent-core list states a shared mechanism and a varying parameter for each
  control.
- Divergence is split into parameter-level vs. structural, with at least one structural
  delta named per regime that has one, attributed to the activating regime(s).
- The convergence/divergence comparison table spans all four regimes across at least
  classification, minimization, audit log, gate, residency, retention, plus the
  structural deltas, with a converge/converge-param/diverge-structural verdict per row.
- A stacking case is resolved by per-class routing and a conflict case is resolved by
  surface-and-escalate (not a silent default).
- `NOTES.md` answers the three prompts: the rough converge-vs-diverge fraction (and what
  it implies about adding a fifth regime), which regime tempted being treated as "the
  main one" and how balance was kept, and why one structural delta genuinely cannot be
  reduced to a parameter.
