# mod-309-governance-compliance-domain/exercise-01-horizontal-framework-controls-mapping — Solution

## Approach

The deliverable is a *clause-to-component map*, not prose about governance. The
discipline that makes it work is staying **horizontal**: the system is classified and
controlled from its own properties — autonomy, action surface, affected parties — and
never from a sector. No HIPAA, GLBA, PCI, FERPA, COPPA, or any statute appears here.
That is not a stylistic preference; it is the whole point. If the horizontal map is
correct, the same agent can later be sold into a hospital, a bank, an agency, or a
school district and the sector layer (exercise-02) only *parameterizes* what is mapped
here — it does not replace it.

The choices that drive the map below:

- **The risk tier is derived, not assumed.** The reference agent is autonomous, calls
  consequential tools (it can modify records and notify external parties), and acts on
  identifiable individuals. Those properties — not a sector — put it in the EU AI Act
  **high-risk** tier, and the tier is what attaches the obligation set *before any
  statute exists*.
- **Every control resolves to a real place or is honestly open.** A map where every
  row says "satisfied" is a lie. Each ISO/IEC 42001 row points at a concrete component,
  boundary, log, gate, or named owner — or is marked `partial`/`open`, and that becomes
  a governance finding.
- **The RMF risk list includes the action surface, not just the output.** A
  tool-calling agent can take a wrong *action* (wire a payment, message the wrong
  recipient, mutate a record) independent of whether its text is "good." Half of real
  agentic risk lives on the action surface; the Map function has to name it.
- **Measure and Manage are different components.** A risk is not handled because one
  pipeline exists. Each top risk gets a named *Measure* component (eval harness,
  red-team, observability) and a separate named *Manage* component (guardrail, HITL
  gate, tool scoping). Conflating them is how risks go un-governed.

## Reference solution

### 1. EU AI Act risk-tier classification

**Tier: HIGH-RISK** — justified from system properties, not sector.

| Property of the reference agent | Why it pushes the tier up |
| --- | --- |
| Runs largely autonomously | Decisions execute without a human in most paths; no per-action human judgment by default. |
| Consequential action surface | Can modify internal records and notify external parties — actions that affect real people and are hard to reverse. |
| Acts on identifiable individuals | Ingests data about people and drafts/takes actions about them; affected parties are individuals, not just internal state. |
| Non-deterministic core | Same input can produce different actions; behavior cannot be fully enumerated in advance. |
| Fleet + frequent deploys + open tool surface | Many versions live at once and third parties extend it — the governed thing is a moving population, not one binary. |

It is **not** unacceptable (no prohibited practice — no social scoring, no
manipulative subliminal technique). It is **above** limited/minimal because the action
surface affects individuals. The transparency duty of the *limited* tier still applies
**in addition**: if a person interacts with the agent, they must be told they are
dealing with an AI system.

Obligations the high-risk tier attaches (named, sector-free):

- **Risk management system** — a continuous, documented process (this maps onto the
  RMF Govern/Map/Measure/Manage loop below).
- **Data governance** — quality, relevance, and handling controls for the data the
  system uses (the data-handling design, exercise-05).
- **Technical documentation + record-keeping (logging)** — automatic logging of events
  over the system's lifecycle (the audit-trail spec, exercise-04).
- **Human oversight** — the system must be designed so a human can intervene/override
  (HITL gates, mod-308; intervention paths, exercise-04).
- **Accuracy, robustness, cybersecurity** — measured and managed (eval harness mod-304;
  red-team and guardrails mod-306).
- **Transparency** — users informed they interact with AI; meaningful information about
  the system's operation.

### 2. ISO/IEC 42001 controls → components

| 42001 control area | What it requires (our words) | Component / boundary / log / gate / owner that satisfies it | Status |
| --- | --- | --- | --- |
| AI system impact assessment | A documented assessment of the system's impact on individuals and groups before and during operation. | The high-risk classification above + the affected-party analysis; the impact assessment is reviewed on material change. Owner: fleet governance lead. | partial — assessment exists; periodic-review cadence not yet wired (see exercise-04 stretch). |
| Data for AI systems | Governed acquisition, quality, and handling of data the system uses, with provenance. | Data-classification scheme + data-flow boundary map + residency map (exercise-05); lineage records data by class/source. Owner: data governance owner. | partial — classification designed; runtime residency-violation detector is an open finding (exercise-05 stretch). |
| AI lifecycle management | A managed lifecycle (design → deploy → operate → retire) with versioning and change control across the fleet. | Durable run history + rainbow/version retirement (mod-308); lineage records agent/model/prompt versions (exercise-04). Owner: platform/release owner. | satisfied — versions are tracked and retired; lineage captures the active version per run. |
| Transparency to users | Users are informed they are interacting with an AI system and given meaningful information about it. | Interaction-time disclosure at the agent's user boundary; system card describing scope/limits. Owner: product owner. | partial — disclosure specified; per-surface enforcement not yet audited. |
| Use of AI by third parties (tool surface) | Any tool/extension the agent can call is governed: evaluated, version-pinned, approved, monitored, revocable. | The extension registry + evaluate→version→approve→monitor lifecycle (exercise-03). Owner: extension-governance owner. | partial — lifecycle designed (exercise-03); runtime quality-drift monitoring not yet wired for all tools. |
| Logging / record-keeping | Consequential events are recorded immutably and retained per policy, queryable for audit. | Append-only, tamper-evident audit trail riding the durable history (exercise-04). Owner: governance lead. | partial — schema and immutability specified; hash-chaining anchor is a stretch finding. |
| Human oversight / accountability | A human can intervene/override; named accountability for the fleet's behavior. | HITL decision gates (mod-308); fleet RACI + kill-switch/disable/revoke intervention paths (exercise-04). Owner: accountable role per gate. | satisfied — gates and intervention paths are specified with named roles. |

An honest map has open findings. The `partial` and `open` rows above feed task 4.

### 3. NIST AI RMF functions instantiated for this agent

```text
   GOVERN  : The fleet governance lead is accountable for the agent's behavior;
             stated risk tolerance = no un-gated irreversible action on an
             individual; every consequential action is logged and attributable.
   MAP     : the top risks of THIS autonomous, tool-calling, non-deterministic
             agent (below — note half are on the ACTION surface).
   MEASURE : eval harness (mod-304), red-team, observability (mod-305).
   MANAGE  : guardrails (mod-306), HITL gates (mod-308), tool scoping (exercise-03).
```

**Map — top risks (action surface included, not just output):**

| # | Risk | On output or action? |
| --- | --- | --- |
| R1 | Harmful / incorrect output (hallucinated recommendation taken as fact). | output |
| R2 | Action beyond intent (agent takes a consequential action the user/owner did not authorize). | **action** |
| R3 | Tool misuse / over-broad scope (a tool used outside its approved purpose — e.g., messaging an arbitrary recipient). | **action** |
| R4 | Data leakage (identifiable data reaches a model prompt, tool, log, or region it should not). | data flow |
| R5 | Prompt injection (untrusted retrieved/ingested content hijacks the agent into an unintended action). | **action** (via input) |
| R6 | Non-determinism / drift (behavior shifts after a model or tool change, silently degrading). | output + action |

**Measure / Manage assignment for each top risk:**

| Risk | MEASURE (which component proves it) | MANAGE (which component contains it) |
| --- | --- | --- |
| R1 harmful output | eval harness with quality/factuality suites (mod-304); observability on output (mod-305) | output guardrail / content filter (mod-306); HITL gate on high-consequence claims (mod-308) |
| R2 action beyond intent | red-team scenarios exercising the action surface (mod-304/305) | HITL gate on consequential actions (mod-308); least-privilege tool scoping (exercise-03) |
| R3 tool misuse | per-tool eval + runtime scope checks (exercise-03 monitoring) | runtime scope enforcement at call time + revocation (exercise-03) |
| R4 data leakage | residency/minimization checks; trace inspection (mod-305) | minimization at every boundary + region pinning (exercise-05) |
| R5 prompt injection | adversarial/red-team injection suite (mod-304/306) | input guardrails + tool-call gating; untrusted-content quarantine (mod-306) |
| R6 drift | drift detection against an Evaluate baseline (mod-304/305) | re-enter Evaluate on version change; quarantine on regression (exercise-03) |

### 4. Open-findings list (the governance gap)

| Finding | Control/function it came from | Component that would close it |
| --- | --- | --- |
| OF-1 | Impact assessment (42001) | A scheduled impact-assessment **review cadence** with a trigger on material change (exercise-04 periodic-review). |
| OF-2 | Data for AI systems (42001) / R4 | A runtime **residency-violation detector** that catches a data class reaching a disallowed region (exercise-05 stretch). |
| OF-3 | Transparency to users (42001) | Per-surface **disclosure enforcement** check audited at each user boundary. |
| OF-4 | Use of AI by third parties (42001) / R3, R6 | **Runtime quality-drift monitoring** wired for *every* registered tool, not just the high-risk ones (exercise-03 monitoring). |
| OF-5 | Logging (42001) | A **tamper-evidence** mechanism (hash-chaining or external anchoring) so a privileged operator cannot rewrite history (exercise-04 stretch). |

This list *is* the governance gap. Every item names the component that closes it, so
the gap is a backlog, not a vague worry.

### Stretch: one control, three frameworks; certification order; over-governance

- **One control satisfying three frameworks.** *Human oversight of consequential
  actions* is simultaneously (a) an EU AI Act high-risk **obligation** (human
  oversight), (b) ISO/IEC 42001 **control** (human oversight / accountability), and (c)
  an RMF **Manage** action (HITL gate on R2). One HITL gate, evidenced by the audit
  record of each gate decision, discharges all three. This is why mapping horizontally
  first is efficient: a single well-placed control retires obligations across every
  framework at once.
- **Certification-readiness ordering.** To pursue a 42001 certificate, the blocking
  open findings, in close order: **OF-5 (tamper-evidence)** first — an audit log a
  privileged operator can rewrite undermines *every* other control's evidence; then
  **OF-1 (impact-assessment review cadence)** — a one-time assessment with no review is
  a documentation gap an auditor will flag; then **OF-4 (drift monitoring on all
  tools)** — the third-party-use control is only credible if monitoring is universal.
  OF-2 and OF-3 are improvements that strengthen but do not block the certificate.
- **Over-governance failure mode.** Applying a full **HITL gate to every notification**
  the agent sends would add latency and reviewer fatigue without reducing real risk:
  most notifications are low-consequence and reversible. The right control is *tiering*
  — gate the consequential, irreversible actions (record mutation, external-party
  notification with effect) and let low-consequence notifications flow with logging
  only. A control applied without tiering buys friction, not safety.

## Meeting the acceptance criteria

- **Tier classified and justified from properties, not sector; obligations named** —
  the high-risk classification table reasons purely from autonomy, action surface, and
  affected parties, and the obligation list is enumerated.
- **42001 map covers the listed areas; every control resolves or is honestly open** —
  the seven-row table includes impact assessment, data, lifecycle, transparency, and
  third-party use, each pointing at a component/owner or marked partial/open.
- **RMF instantiated; top risks include the action surface; each has Measure + Manage**
  — Govern/Map/Measure/Manage are stated for this agent, three of six top risks are on
  the action surface, and each risk has a distinct named Measure and Manage component.
- **Open-findings list complete; each names the closing component** — five findings,
  each traced to its source control and its closing component.
- **No sector statute appears** — the map is purely horizontal; HIPAA/GLBA/PCI/FERPA/
  COPPA appear nowhere in the artifact.

## Common pitfalls

- **Classifying from the sector.** "It's healthcare, so it's high-risk" is the wrong
  reasoning — it would fail the moment the same agent is sold elsewhere. Derive the tier
  from autonomy + action surface + affected parties so it holds in every sector.
- **An all-green map.** If every 42001 row is "satisfied," the map is not being honest.
  Real architectures have partial and open controls; surfacing them is the deliverable.
- **Mapping only the output risks.** Listing "hallucination" and stopping there misses
  the action surface — action-beyond-intent, tool misuse, injection-driven action. A
  tool-calling agent's worst failures are *actions*, not sentences.
- **Conflating Measure and Manage.** "We have guardrails" is a Manage answer; it does
  not tell you whether the risk is *measured*. Each risk needs both a thing that proves
  the risk (eval/red-team/observability) and a thing that contains it (guardrail/gate/
  scoping).
- **Reaching for a statute.** The instant FERPA or PCI appears, the map has stopped
  being horizontal. The whole value of this layer is that it is sector-independent.

## Verification

A completed submission is correct when:

- The EU AI Act tier is stated (high-risk) and every justification cited is a *property*
  of the system, with the attached obligation set listed and the transparency duty noted.
- The 42001 table covers impact assessment, data for AI, lifecycle, transparency, and
  third-party tool use, and each row resolves to a named component/boundary/log/gate/
  owner or is marked partial/open.
- The four RMF functions are instantiated for *this* agent; the Map risk list contains
  at least the action-surface risks (action-beyond-intent, tool misuse, injection); and
  each top risk has a distinct Measure component and Manage component.
- The open-findings list collects every partial/open control and names the closing
  component for each.
- No sector-specific statute appears anywhere in the artifact.
- `NOTES.md` answers the three prompts: the hardest 42001 control to point at (and the
  real gap it reveals), how much of the architecture the horizontal frame already
  mandates before any sector, and how one Manage guardrail would be *measured* working
  and where that signal comes from.
