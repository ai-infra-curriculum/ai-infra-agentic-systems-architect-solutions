# mod-309-governance-compliance-domain/exercise-05-data-handling-residency-design — Solution

## Approach

The deliverable is the **data-handling mechanism built once and parameterized per
regime**: a classification scheme, a data-flow boundary map with minimization points, a
residency map, a per-class retention schedule, and a **regime-profile object** that
re-tunes all of it for a new regime without changing the architecture. The governing
principle is the split between a **fixed mechanism** (classify, minimize-at-boundary,
region-pin, per-class lifecycle) and **regime parameters** (the identifiable rule, class
labels, allowed regions, retention schedules, gate authority). A new regime is a new
*profile*, not a new build.

The choices that drive the design below:

- **Classification is the precondition for every downstream control.** Minimization,
  residency, retention, and erasure all key off the class tag applied at ingest. An
  untagged datum cannot be minimized, region-pinned, retained correctly, or found for
  erasure. So the tagging *mechanism* is fixed and universal; only the *labels* and the
  *"what counts as identifiable"* rule are regime parameters.
- **The model prompt, tool calls, and logs are the forgotten boundaries.** Teams
  minimize at the database edge and forget that identifiable data flows straight into a
  model prompt, out through a tool call, and into a trace. The boundary map places a
  minimization control at *every* boundary, especially those three.
- **Residency is not just storage.** The inference endpoint, the vector/document store,
  the tool backends, and the trace pipeline are *all* in scope. A model endpoint
  failing over to another region is a residency violation as real as a database in the
  wrong country.
- **Audit records outlive raw data.** Retention pulls two ways — keep-for-audit vs.
  delete-for-privacy. Separating the audit store from the raw store (and referencing
  data by class, per exercise-04) is what lets a deletion right erase the raw datum while
  the audit reference survives.

## Reference solution

### 1. Data-classification scheme

Sensitivity classes tagged at **ingest** (the fixed mechanism):

| Class | Meaning (the fixed slot) | The regime parameterizes... |
| --- | --- | --- |
| identifiable | Data that identifies an individual. | *what counts as identifiable* (the rule) — e.g., PHI vs. NPI+cardholder vs. citizen-PII vs. education-record-of-a-minor. |
| sensitive-category | A heightened-protection subclass. | which categories are sensitive (health, financial/card, controlled, minor). |
| internal | Non-public org data, not about an individual. | thresholds for internal handling. |
| public | Already-public / non-sensitive. | (rarely varies). |

**Why classification is the precondition:** every later control reads the class tag. No
classification → no minimization target, no residency rule, no retention schedule, no way
to find a subject's data for erasure. So the **tagging mechanism is fixed**; the **class
label set and the identifiable-rule are regime parameters** (carried by the regime
profile in task 5). Same tagger, different rule of what to tag.

### 2. Data-flow boundary map with minimization points

```text
   ingest ─(CLASSIFY)─▶ agent core ─(MINIMIZE)─▶ model prompt
                            │
                            ├─(SCOPE + MINIMIZE)─▶ tools / sub-agents ─▶ egress
                            └─(MINIMIZE)─────────▶ logs / traces
```

A minimization control (strip / mask / tokenize / redact) sits at **every** boundary
where data reaches a component that should not see it in the clear:

| Boundary | Why it is a leak point | Minimization control |
| --- | --- | --- |
| ingest → agent core | raw data enters; nothing is tagged yet | classify first (the precondition), then everything downstream can act |
| agent core → **model prompt** | identifiable data flows into a prompt sent to an inference endpoint — *most-forgotten* | mask/redact identifiers; pass references or pseudonyms where the task allows |
| agent core → **tool calls / sub-agents** | a tool is an egress boundary; data leaves the trust boundary | scope the call (least privilege) + minimize the payload; block disallowed classes in the call body |
| tools → egress | external send | minimized payload only; regulated classes blocked from leaving |
| agent core → **logs / traces** | traces silently capture prompts, args, and outputs — *most-forgotten* | redact identifiable fields before they hit the trace pipeline |

The three explicitly-included boundaries — **model prompt, tool calls, logs** — are the
ones real systems forget; the map forces a control on each.

### 3. Residency map

For each class, the allowed region(s) for storage *and* processing — with the commonly
missed components in scope:

| In-scope component | Why it is in scope (the common miss) | Residency rule |
| --- | --- | --- |
| model inference endpoint | the prompt (with whatever it carries) is processed at the endpoint's region | pin to allowed region; block/queue on cross-region failover |
| vector / document store | retrieved context lives here | pin to allowed region |
| tool backends | a tool call processes data wherever the backend runs | tool backend region must be in the allowed set or the call is blocked |
| trace pipeline | traces carry prompts/args/outputs to wherever traces are stored | trace storage pinned to allowed region |

**Crossing controls:** where data *must* cross a boundary, the crossing is **explicit,
logged, and minimized** — never implicit. A class with no permitted region outside its
home forces a **per-region deployment of the whole stack** (inference + store + tools +
traces all in-region) rather than a cross-region call. The **allowed-region set is a
regime parameter** (e.g., in-jurisdiction-only for a public-sector profile; a contractual
region for others).

### 4. Per-class retention schedule

| Data class | Min lifetime | Max lifetime | Destruction method | Note |
| --- | --- | --- | --- | --- |
| input (raw ingested) | operational need | privacy limit | secure delete | shortest practical; erasure target |
| retrieved context | none beyond run | privacy limit | secure delete | often ephemeral / cache-TTL'd |
| prompts / completions | audit need | privacy limit | secure delete | **shorter than audit records — do not conflate the two stores** |
| logs / traces | debug need | privacy limit | secure delete | redacted (task 2); short TTL |
| run state | until run + grace | privacy limit | secure delete | durable-history-bound (mod-308) |
| audit records | **regime floor (may be long)** | regime ceiling | controlled, post-retention | **may outlive the raw data they describe** (exercise-04) |

- **Automated lifecycle enforcement:** TTLs and deletion jobs that *actually fire* — a
  retention policy nobody enforces is a liability, not a control. Each class has an owner
  and a scheduled destruction job verified to run.
- **Right-to-erasure flow:** given a subject id, enumerate and remove their data across
  agent memory, the vector/document store, logs/traces, and run state — **enabled by the
  task-1 classification** (you can only find a subject's data if it was tagged). The audit
  reference survives because it is minimized (references by class, not content).
- **Schedules are regime parameters:** the min/max values per class come from the regime
  profile (SOX multi-year floor vs. consent-bound vs. records-schedule), not the
  architecture.

### 5. Regime-profile object (the parameterization)

```text
RegimeProfile {
  regime_id:           "healthcare" | "finance" | "public" | "edtech" | ...
  identifiable_rule:   predicate -> what counts as identifiable/sensitive
  class_labels:        [ ...the sensitivity class set... ]
  allowed_regions:     { class -> [ allowed region set ] }
  retention_schedules: { class -> { min, max, destroy_method } }
  gate_authority:      { action_class -> { role, threshold } }   // ref exercise-02/04
}
```

**Same architecture, two profiles** (proving nothing structural changes — only
parameters):

| Parameter | Profile A — finance (GLBA/SOX/PCI) | Profile B — public sector |
| --- | --- | --- |
| identifiable_rule | NPI + cardholder data | citizen PII (+ controlled categories) |
| class_labels | [identifiable, sensitive=card/NPI, internal, public] | [identifiable, sensitive=controlled, internal, public] |
| allowed_regions | contractual region set | **in-jurisdiction only** |
| retention_schedules | SOX multi-year floor on audit records | records-schedule (some permanent) |
| gate_authority | control owner; segregation of duties | agency-designated official |

Under both profiles the *mechanism* is identical: classify at ingest → minimize at every
boundary → pin classes to regions → enforce per-class lifecycle. Only the boxed
parameters differ. Switching from A to B changes the profile object and **nothing
structural** — no new component, no new boundary, no code path added or removed.

### Stretch: erasure operation, residency-violation detector, convergence reconciliation

- **Erasure operation in detail.** Given `subject_id`: (1) query the classification index
  built at ingest for every datum tagged to that subject across **agent memory (mod-303)**,
  **vector store**, **logs/traces**, and **run state**; (2) issue scoped deletes to each
  store; (3) tombstone the subject in the index so late-arriving data is caught; (4) leave
  the **audit trail** intact because it references the subject *by class, minimized* — the
  trail proves what happened without holding the erased content. This is *only* possible
  because task-1 classification tagged the data at ingest — untagged data cannot be
  enumerated for erasure.
- **Residency-violation detector.** A runtime control that inspects, on each
  cross-component hop, whether a data class is reaching a region outside its
  `allowed_regions` (e.g., a model endpoint **failed over** to a secondary region). On a
  violation it **blocks the call** (or queues it for an in-region endpoint), emits an
  audit `policy_enforcement` event, and alerts the data-governance owner — turning a silent
  residency breach into a caught, logged, owned event.
- **Reconciliation with exercise-02's convergence map.** The **convergent** controls here
  — classification, minimization-at-boundary, region pinning, per-class lifecycle — are one
  mechanism parameterized by the profile (matching exercise-02's converge-param verdicts).
  The **structural** pieces from exercise-02 (card-data scope reduction, parental consent)
  are *not* parameters on this mechanism; they are pluggable components that activate only
  for the regimes that need them. So this design is the convergent core of exercise-02 made
  concrete, with the structural deltas left as activation flags on the profile.

## Meeting the acceptance criteria

- **Classification defined; identifiable-rule and labels explicitly regime parameters over
  a fixed tagger** — the class table separates the fixed slot from the regime-parameterized
  rule/labels.
- **Data-flow map places minimization at every boundary, incl. prompt, tools, logs** — the
  diagram plus boundary table put a control on ingest, model prompt, tool calls, egress, and
  logs/traces, with the three forgotten ones called out.
- **Residency map treats inference, storage, tool backends, traces as in-scope; crossing
  controls; allowed-region a parameter** — the residency table lists all four components,
  defines explicit/logged/minimized crossings and the per-region-deployment trigger, and
  parameterizes the region set.
- **Retention covers every class with min/max/destruction, separates audit from raw,
  includes erasure, schedules are parameters** — the six-class table, the audit-outlives-raw
  note, automated enforcement, the erasure flow, and per-regime schedules.
- **Regime-profile object concrete; same architecture under two profiles, only parameters
  change** — the profile object plus the finance/public-sector side-by-side showing
  identical mechanism and differing parameters only.

## Common pitfalls

- **Minimizing only at the database edge.** The model prompt, tool calls, and trace
  pipeline are the boundaries that leak — identifiable data flows into all three and is the
  most-forgotten exposure. Put a control on each.
- **Treating residency as a storage-only concern.** The inference endpoint and trace
  pipeline process data in *some* region; a failover to another region is a real violation.
  All four components are in scope.
- **One store for audit and raw data.** Conflate them and you cannot honor a deletion right
  without breaking the audit retention floor. Separate stores, reference data by class.
- **Hardcoding thresholds.** Baking one regime's identifiable rule, region set, or retention
  numbers into the architecture means a new regime is a new build. Push all of it into the
  regime-profile object; the mechanism stays fixed.
- **Retention policy with no enforcement.** A TTL nobody fires is worse than none — it is a
  documented promise the system breaks. The deletion jobs must actually run and be verified.

## Verification

A completed submission is correct when:

- The classification scheme defines the sensitivity classes, explains classification as the
  precondition for every downstream control, and marks the identifiable-rule and label set
  as regime parameters over a fixed tagging mechanism.
- The data-flow map places a minimization control at every boundary and explicitly includes
  the model prompt, tool calls, and logs/traces.
- The residency map puts the inference endpoint, vector/document store, tool backends, and
  trace pipeline all in scope, defines explicit/logged/minimized crossing controls and the
  per-region-deployment trigger, and makes the allowed-region set a parameter.
- The retention schedule covers input, retrieved context, prompts/completions, logs, run
  state, and audit records with min/max/destruction, separates audit records from raw data,
  includes a right-to-erasure flow, and makes the schedules parameters.
- The regime-profile object is concrete, and the same architecture is shown under two of the
  four peer regimes with only parameters changing and nothing structural.
- `NOTES.md` answers the three prompts: the most-forgotten boundary and what leaks through
  it, a data class where keep-for-audit vs. delete-for-privacy conflict and how separate
  stores resolve it, and exactly what changed (and what did not) when switching the regime
  profile.
