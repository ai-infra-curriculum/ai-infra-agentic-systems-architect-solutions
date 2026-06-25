# AI Infrastructure Agentic Systems Architect — Solutions Repository

<!-- aicg:site-banner -->
> 🎓 Part of the free, open-source **AI Career Curriculum** ecosystem — [Infrastructure](https://github.com/ai-infra-curriculum) · [ML Engineering](https://github.com/ml-engineering-curriculum) · [AI Engineering](https://github.com/ai-engineering-curriculum) · [Governance](https://github.com/ai-governance-curriculum). Live cohorts &amp; team programs: **[ai-infra-curriculum.github.io](https://ai-infra-curriculum.github.io/)**.
<!-- /aicg:site-banner -->

> **Worked reference solutions for the Agentic Systems Architect track (~L48)**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Modules: 10](https://img.shields.io/badge/modules-10-326CE5.svg)](#-module-solutions)
[![Exercise Solutions: 42](https://img.shields.io/badge/exercise%20solutions-42-blue.svg)](#-module-solutions)
[![Status: Complete](https://img.shields.io/badge/reference%20solutions-complete-success.svg)](#-overview)

## 🎯 Overview

This repository contains the reference solutions for the paired
[**Agentic Systems Architect Learning Repository**](https://github.com/ai-engineering-curriculum/agentic-systems-architect-learning) —
the top rung of the agentic track, pitched at the **L48 architect** level.

At this altitude the deliverable is a **design**, not a service. So the solutions are
**worked design artifacts**, not application code:

- 🧮 **Decision matrices** — workflows-vs-agents, handoffs-vs-manager-as-tools,
  durable-vs-ephemeral, cost/latency/quality budgets, with the trade-offs defended
- 🏗️ **Reference architectures** — orchestrator–worker topologies, memory-tier
  placement, MCP/A2A integration specs, fleet deployment and recovery designs
- 🛡️ **Threat models & security specs** — prompt-injection threat models, guardrail
  placement, least-privilege tool permissions, OAuth/RBAC scoping, sandboxing
- 📋 **Governance artifacts** — horizontal-framework control mappings (ISO/IEC 42001,
  NIST AI RMF, EU AI Act), multi-regime regulated-domain designs, accountability specs
- 💻 **Code where it earns its place** — runnable evaluation harnesses (mod-304),
  OTel GenAI span instrumentation (mod-305), and platform extension/config examples
  across multiple agentic platforms (mod-310) ship real, verified artifacts rather than prose

Each exercise solution is a self-contained walkthrough: the approach, the worked
artifact, the acceptance-criteria mapping, the failure modes named, and the
trade-offs made explicit — what a strong architect would actually defend in review.

> **Status**: ✅ Reference solutions complete for every module exercise.
> AI-assisted content under ongoing human review.

## 📁 Repository Structure

```text
ai-infra-agentic-systems-architect-solutions/
├── modules/
│   ├── mod-301-agentic-systems-foundations/      # module README + per-exercise solutions
│   ├── mod-302-multi-agent-orchestration/
│   ├── ...                                        # one directory per module
│   └── mod-310-agentic-developer-platforms/
│       ├── README.md                              # module index + rationale
│       └── exercise-NN-*/README.md                # one worked solution per exercise
├── projects/
│   ├── project-301-production-agentic-reference-architecture/
│   ├── project-302-regulated-domain-agent-architecture/
│   └── project-303-extensible-agent-platform/
├── SOLUTIONS_INDEX.md                             # inventory + completion map
└── README.md                                      # this file
```

Every module directory has a `README.md` index, and every `exercise-NN-*/`
directory holds a single `README.md` — the worked solution for that exercise.

## 📚 Module Solutions

| Module | Solutions |
|--------|-----------|
| [mod-301 — Agentic Systems Foundations](./modules/mod-301-agentic-systems-foundations/) | 4 exercises: workflows-vs-agents decision matrix, orchestration pattern catalog, tool-vs-subagent decomposition, reference-architecture teardown |
| [mod-302 — Multi-Agent Orchestration](./modules/mod-302-multi-agent-orchestration/) | 4 exercises: orchestrator–worker topology, handoffs-vs-manager-as-tools, MCP/A2A integration spec, coordination & escalation design |
| [mod-303 — Memory & Context Architecture](./modules/mod-303-memory-context-architecture/) | 4 exercises: context budget & rot analysis, memory-tier placement, compaction & note-taking, RAG architecture for agents |
| [mod-304 — Evaluation Harnesses](./modules/mod-304-evaluation-harnesses/) | 4 exercises (runnable): trajectory eval, tool-call correctness harness, LLM-as-judge rubric, eval-gated release pipeline |
| [mod-305 — Observability & Tracing](./modules/mod-305-observability-tracing/) | 3 exercises: OTel GenAI span design, quality & drift signals, observability-platform evaluation |
| [mod-306 — Guardrails, Safety & Security](./modules/mod-306-guardrails-safety-security/) | 6 exercises: guardrail placement, prompt-injection threat model, least-privilege tool permissions, OAuth/RBAC token management, CLI agent sandboxing, excessive-agency controls |
| [mod-307 — Cost & Latency Architecture](./modules/mod-307-cost-latency-architecture/) | 3 exercises: token-economics cost model, caching & routing architecture, cost/latency/quality budgets |
| [mod-308 — Deployment & Durable Execution](./modules/mod-308-deployment-durable-execution/) | 4 exercises: durable-execution design, HITL approval architecture, agent-fleet deployment, failure recovery & resumption |
| [mod-309 — Governance, Compliance & Regulated-Domain Architecture](./modules/mod-309-governance-compliance-domain/) | 5 exercises: horizontal-framework controls mapping (ISO 42001 / NIST AI RMF), multi-regime regulated-domain architecture (healthcare/finance/public-sector/edtech as peers), extension & tool governance lifecycle, governance & accountability spec, data-handling & residency design |
| [mod-310 — Agentic Developer & Internal Platforms](./modules/mod-310-agentic-developer-platforms/) | 5 exercises (real config across multiple platforms): extension-model mapping across platforms (Agents/Skills/Hooks/MCP), secure tool-use & context hydration, workflow-toolchain integration, platform adoption & DX, non-SDLC platform case |

**Totals:** 10 modules · 42 exercise solutions · 3 capstone projects.

The three capstone projects under [`projects/`](./projects/) integrate the modules
into end-to-end design walkthroughs: a
[production agentic reference architecture](./projects/project-301-production-agentic-reference-architecture/),
a [regulated-domain agent architecture](./projects/project-302-regulated-domain-agent-architecture/),
and an [extensible agent platform](./projects/project-303-extensible-agent-platform/).

## 📖 How to Use This Repository

These are reference answers — get the most from them by trying the exercise first.

1. **Start in the [learning repository](https://github.com/ai-engineering-curriculum/agentic-systems-architect-learning).**
   Read the module brief and attempt the exercise's design artifact yourself.
2. **Produce your own artifact first** — your own decision matrix, threat model, or
   reference diagram. The value is in defending the trade-offs, not matching the text.
3. **Compare against the reference solution** in the matching `exercise-NN-*/README.md`.
   Note where your trade-offs differed and *why* — divergence is often defensible.
4. **Read the failure modes and acceptance-criteria mapping.** Each solution names the
   common mistakes a grader sees and ties the artifact back to the exercise's criteria.
5. **For mod-304, mod-305, and mod-310**, the solutions include runnable harnesses,
   OTel instrumentation, and platform extension/config examples across multiple agentic
   platforms — run and adapt them.
6. **Work the capstone projects** once the module solutions click, to see the pieces
   compose into a single architecture.

See [`SOLUTIONS_INDEX.md`](./SOLUTIONS_INDEX.md) for the full inventory and
completion map.

## 🔗 Paired Learning Repository

- **[ai-infra-agentic-systems-architect-learning](https://github.com/ai-engineering-curriculum/agentic-systems-architect-learning)**
  — module briefs, exercise specs, and project stubs. Start there; return here to
  check your work.

## 📞 Contact & Support

- **Email:** ai-infra-curriculum@joshua-ferguson.com
- **GitHub Organization:** [ai-infra-curriculum](https://github.com/ai-infra-curriculum)
- **Issues:** [Report bugs or request features](https://github.com/ai-engineering-curriculum/agentic-systems-architect-solutions/issues)

## 📄 License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

---

<!-- aicg:maintained-by -->
Maintained by [VeriSwarm.ai](https://veriswarm.ai)
