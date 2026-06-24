# mod-309-governance-compliance-domain — Solutions

Reference solutions for the Governance, Compliance & Regulated-Domain Architecture module
(~L48). Each exercise solution is a worked **governance artifact** — mappings,
comparison tables, lifecycle designs, specs, and data-handling designs, not code.

The module governs **horizontally first**: AI is made governable in general (ISO/IEC
42001, NIST AI RMF, EU AI Act) before any sector enters. Regulated sectors —
**healthcare (HIPAA)**, **finance (GLBA/SOX/PCI)**, **public sector**, and **edtech
(FERPA/COPPA)** — are then treated as **balanced peers**: one architecture,
parameterized per regime, with no sector as the default.

> **Not legal or audit advice.** These are teaching references. Framework citations
> (ISO/IEC 42001, NIST AI RMF, EU AI Act) and statutory citations (HIPAA, GLBA, SOX,
> PCI, FERPA, COPPA, and others) are stated for architectural mapping; verify against
> the primary sources and engage counsel for binding interpretation.

## Exercise index

- [exercise-01-horizontal-framework-controls-mapping](exercise-01-horizontal-framework-controls-mapping/README.md)
  — a sector-neutral controls map: an EU AI Act risk-tier classification justified from
  system properties, a seven-row ISO/IEC 42001 clause-to-component table, the NIST AI
  RMF functions instantiated for the agent (with Measure/Manage per top risk), and an
  open-findings list. Purely horizontal — no statute appears.
- [exercise-02-multi-regime-regulated-domain-architecture](exercise-02-multi-regime-regulated-domain-architecture/README.md)
  — one agent under four peer regimes: each decomposed into the five primitives at equal
  rigor, the convergent core, parameter-vs-structural divergence, the headline
  convergence/divergence comparison table, and stacking + conflict resolution.
- [exercise-03-extension-tool-governance-lifecycle](exercise-03-extension-tool-governance-lifecycle/README.md)
  — the evaluate → version → approve → monitor lifecycle for third-party and internal
  tools: a five-dimension rubric, an immutable registry, a risk-tiered approval contract,
  runtime monitoring + fast registry-based revocation, and two worked extension
  walk-throughs.
- [exercise-04-governance-accountability-spec](exercise-04-governance-accountability-spec/README.md)
  — fleet-level specs: a minimized lineage spec, an audit-trail spec (schema, immutability,
  per-regime retention, acid-test query), a human-accountability spec (gates, RACI,
  intervention, contestability), a risk-control catalog, and an incident walk-through.
- [exercise-05-data-handling-residency-design](exercise-05-data-handling-residency-design/README.md)
  — privacy/residency/retention built once and parameterized: a classification scheme, a
  data-flow boundary map with minimization at the prompt/tools/logs, an in-scope residency
  map, a per-class retention schedule, and a regime-profile object shown under two peer
  regimes.

## How to read these

Each solution follows the same shape: **Approach** (the method), **Reference solution**
(the worked artifacts), **Meeting the acceptance criteria**, **Common pitfalls**, and
**Verification**. The artifacts compose: the horizontal framework map and RMF risk list
(01) feed the risk-control catalog (04); the four-regime convergence map (02) supplies
the regime parameters consumed by the lineage/audit specs (04) and the data-handling
design (05); and the extension registry (03) supplies the tool-version pins that the
lineage spec (04) records. Throughout, the four regulated sectors are kept as equal
peers — none is the default frame.
