# mod-310-agentic-developer-platforms/exercise-05-non-sdlc-platform-case — Solution

## Approach

This is the generalization proof: take the platform pattern designed across
exercises 01–04 for a software-development context and **re-bind it to a
non-developer internal domain** to show the architecture is domain-independent. The
deliverable is a side-by-side mapping of what *changed* (the nouns: inputs, tools,
systems, the irreversible-action list) versus what *stayed constant* (the
architecture: extension model, trust asymmetry, hydration, interface selection,
DX), ending in a one-sentence verdict.

The reference picks domain **(a) Operations / on-call** as the worked re-binding,
and maps a second domain (analytics) in the stretch to make the proof stronger.
The point is the proof, not the volume — a tight, honest mapping that reads "same"
down the architecture column beats a sprawling redesign.

The discipline throughout: SDLC was always **one binding of the pattern, never the
pattern itself**. So this exercise does not "port a coding tool to ops" — it
re-populates the nouns of a domain-free pattern. If the architecture column ever
reads anything but "same," the original design leaked a domain assumption, and that
is the bug this exercise exists to catch.

## Reference solution

### 1. The platform pattern, restated domain-free

The five elements with **no** domain-specific words — no code, commits, tickets, or
developers:

1. **Extension model.** Capabilities are placed by *kind*: an isolated role is an
   **agent**; runtime-enforced guarantees are **hooks/lifecycle**; external access
   across a trust boundary is an **MCP server**; procedural know-how is a **skill**;
   a callable pure function is a **tool**.
2. **Trust asymmetry.** Inputs are **untrusted** (they may carry injected
   instructions); tools are **privileged**. The model is **not** a boundary; a
   deterministic gate on the tool layer is. Actions sit on a **reversibility
   ladder** (read / sandbox-write / irreversible), and the irreversible tier is
   **gated**.
3. **Hydration.** Context is loaded in layers (standing / task / retrieved /
   tool-result), each budgeted; the rule is **pull, don't pre-load; distill, don't
   dump**; missing context **degrades safely** and never fabricates.
4. **Integration.** Interface by access shape: discrete write or shallow read →
   **REST**; deep related read in one round-trip → **GraphQL**; reaction to an
   external change → **event** (never poll the reactive trigger). Each binding sits
   behind an MCP server.
5. **Adoption + DX.** Two decisive moments — **activation** (first ten minutes) and
   **trust under failure** (first failure: legibility, reversibility by default,
   frictionless report path). Documentation **pays twice** (skills serve human and
   agent), versioning forbids **silent privileged-tier changes**, and a
   **feedback loop** turns every reported bad run into an **eval-gated** regression
   case.

Not one of those five sentences names a developer artifact. That is the test: the
pattern is stated before any domain touches it.

### 2. Re-populate the nouns — Operations / on-call

**Extension model (same placement rule applied to ops nouns):**

- **Agents:** a `triage` role (read-only over monitoring/logs, tight prompt) that
  proposes a next step.
- **Skills:** `how-we-run-an-incident` (the runbook convention), `how-we-write-a-status-update` (the external-comms convention) — procedural know-how.
- **Tools:** a local `parse-and-correlate-log-window` transform (no external call).
- **Hooks/lifecycle:** an always-run **action-logger** that records every
  privileged action taken during an incident — a runtime guarantee, no discretion.
- **MCP servers:** monitoring, logs, paging, ticketing, and status-page — each an
  external system behind its own server, credential in the server.

**Trust asymmetry (the reversibility ladder, irreversible tier gated):**

| Element | This domain's nouns |
| --- | --- |
| Untrusted inputs | Alert payloads, log lines, customer-reported symptoms, third-party status text |
| Privileged tools | Restart a service, page a person, post a public status update, acknowledge/annotate an incident, run a remediation runbook step |
| Irreversible (gated) actions | **Restart prod service**, **public status post**, **page an executive**, **trigger a destructive remediation** — model proposes, human/policy disposes |
| External systems (MCP) | Monitoring, logs, paging, ticketing, status-page (each behind its own MCP server) |

The reversibility ladder: read monitoring/logs (free, scoped) → annotate the
incident / open a draft status (sandbox, free) → **restart prod / public post /
page exec / destructive remediation (gated)**. The trust boundary sits on the tool
layer; an injected instruction in a log line or a customer-reported symptom can
make the agent *propose* a prod restart, but the gate — not the model — decides.

### 3. Hydration and integration carry over unchanged

**Hydration layers (ops):**

| Layer | What's in it (ops) | Pull/distill discipline |
| --- | --- | --- |
| L1 standing | Triage role, gate policy, sev definitions | Retained, small |
| L2 task | *This* incident: the firing alert, the affected service, current status | Fetched, scoped |
| L3 retrieved | A **distilled** slice: the one relevant runbook section + the recent deploys to the affected service — not the whole runbook, not the full deploy log | Pulled on demand, distilled |
| L4 tool-result | The last correlation/health-check result, pruned after the decision | Fetched, pruned |

**Concrete distillation (ops):** the affected service's runbook is 30+ pages; the
retrieval reduces it to the one section matching the firing alert's signature —
*"high-latency on service-X → check upstream cache, then the connection pool;
restart is last resort and requires sev-2 approval"* — ~80 tokens carrying the
decision, instead of dumping 30 pages. "Pull, don't pre-load; distill, don't dump"
holds identically.

**Interface selection (ops) — reactive trigger is an event, not a poll:**

| Ops access shape | Interface |
| --- | --- |
| A **new alert fires** | **Event** (webhook from monitoring) — never poll |
| Incident + related alerts + recent deploys, one round-trip | **GraphQL** (deep related read) |
| Acknowledge / annotate / update status | **REST** (discrete writes) |
| A **remediation job / runbook step completes** | **Event** |

The reactive trigger (new alert) is an **event** — polling monitoring would lag the
incident or burn the rate budget and still race the alert. The rule is unchanged.

### 4. Changed-vs-constant table

| Element | SDLC binding | Non-SDLC (ops) binding | Architecture |
| --- | --- | --- | --- |
| Extension model | review subagent, PR-convention skill, protected-path hook, VCS/CI MCP | triage agent, runbook/status skills, action-logger hook, monitoring/paging MCP | **same** |
| Trust asymmetry | untrusted ticket/file; gated merge/deploy | untrusted alert/log/symptom; gated prod-restart/status-post | **same** |
| Hydration | L1–L4, distill a spec/incident slice | L1–L4, distill a runbook section | **same** |
| Integration | event on change opened, GraphQL item+reviews, REST comment | event on alert, GraphQL incident+alerts+deploys, REST annotate | **same** |
| Adoption + DX | activation + trust, eval-gated feedback | activation (under incident pressure) + trust, eval-gated feedback | **same** |

Every cell in the architecture column reads **same**. Only the nouns in the SDLC
and ops columns differ.

### 5. Verdict

**Re-bind, not rebuild.** We **re-author the noun layer**: the domain MCP servers
(monitoring / logs / paging / ticketing / status-page), the domain skills (runbook,
status-update conventions), and the re-populated irreversible-action list
(restart-prod / public-post / page-exec / destructive-remediation, each gated).
Everything else — **the trust boundary placement and gating discipline, the
four-layer hydration with pull-and-distill, the REST/GraphQL/event interface-
selection rule, the activation-and-trust DX, and the eval-gated feedback loop** —
carries over **wholesale, unchanged**. The architecture is the asset; the domain is
a binding.

### Stretch: a second domain, a higher-stakes gate, and the shared core

**Second domain — Analytics / data (architecture column still "same"):**

| Element | Analytics binding | Architecture |
| --- | --- | --- |
| Extension model | query-author agent, "house metric definitions" skill, result-publish-logger hook, warehouse/BI MCP | **same** |
| Trust asymmetry | untrusted question text / raw dataset; gated **publish dashboard org-wide** / **run unbounded warehouse query** | **same** |
| Hydration | L1–L4, distill the relevant schema slice (not the whole catalog) | **same** |
| Integration | event on new data-request, GraphQL for related metrics/lineage, REST to publish | **same** |
| Adoption + DX | activation + trust, eval-gated feedback | **same** |

Two independent re-bindings both reading "same" down the architecture column is a
stronger proof than one.

**The one higher-stakes place.** In ops, an **irreversible action fires *during an
incident*** — a prod restart proposed by an agent reading injected log text, under
time pressure, is higher-stakes than an SDLC merge. Tighten that gate
specifically: the prod-restart / public-status gate requires a **named human
approver at the current incident severity** (not a generic policy auto-approve),
and the approval is itself logged by the always-run action-logger hook. Same
ladder, a tighter gate where the stakes are highest.

**Shared platform core vs. per-domain binding:**

```text
   SHARED CORE (host-agnostic, domain-free):
     • the trust-boundary + reversibility-ladder + gating discipline
     • the four-layer hydration engine (pull, distill, degrade, budget)
     • the interface-selection rule + the robust event-receiver framework
     • the DX spine: versioned-bundle install, legibility, eval-gated feedback loop
     • generic MCP client + hook/lifecycle runtime

   PER-DOMAIN BINDING (the noun layer, re-authored per domain):
     • the domain MCP servers (monitoring | warehouse | support-system | VCS/CI)
     • the domain skills (runbook | metric-defs | reply-style | PR-conventions)
     • the domain irreversible-action list (prod-restart | org-publish | refund | deploy)
     • the domain's curated first task
```

The core is built once and host-agnostic; each domain ships a thin binding of
nouns on top. That is the architectural payoff of designing the pattern instead of
wiring up a single-domain tool.

## Meeting the acceptance criteria

- **Five-element pattern restated with no domain-specific vocabulary.** Task 1
  states all five elements without code, commits, tickets, or developers.
- **Chosen domain's nouns fully re-populated.** Task 2 names the ops agents,
  skills, tools, hooks, and MCP servers, plus untrusted inputs, privileged tools,
  gated irreversible actions, and external systems.
- **Hydration and integration carry over unchanged, with a concrete distillation
  and an event-based reactive trigger.** Task 3 gives the ops L1–L4 layers, the
  runbook-section distillation, and the new-alert event (not a poll).
- **Changed-vs-constant table reads "same" down the architecture column, nouns
  differing.** Task 4's table is "same" in every architecture cell.
- **Verdict is re-bind, not rebuild, with re-authored nouns and carried-over
  architecture named.** Task 5 names the re-authored noun layer (domain MCP
  servers, domain skills, irreversible-action list) and the wholesale-carried
  architecture.

## Common pitfalls

- **Rebuilding instead of re-binding.** Redesigning the trust boundary or the
  hydration layers for ops means the original design leaked an SDLC assumption.
  The architecture column must read "same"; only nouns change.
- **Leaving SDLC vocabulary in the domain-free restatement.** "Commit," "PR," or
  "developer" in task 1 fails the test — the pattern must be statable before any
  domain.
- **Polling the new-alert trigger.** Ops tempts a monitoring poll; the reactive
  trigger is still an event. The interface-selection rule does not change with the
  domain.
- **Dropping the gate because "ops needs speed during incidents."** Incident
  pressure is the argument for a *tighter* gate on prod-restart/status-post, not a
  looser one.
- **Calling a new noun a new architecture element.** A status-page MCP server is a
  new *noun*, not new architecture — it is the same "external access → MCP"
  placement.
- **Padding the proof.** A sprawling ops redesign is weaker than a tight mapping;
  the value is the "same" column, not the page count.

## Verification

A completed submission is correct when:

- The five-element pattern in task 1 contains **no** domain-specific vocabulary.
- The chosen domain's nouns are **fully** re-populated: agents, skills, tools,
  hooks, MCP servers, untrusted inputs, privileged tools, gated irreversible
  actions, and external systems.
- Hydration and integration are shown to carry over with **one concrete
  distillation** and the reactive trigger correctly designed as an **event**.
- The changed-vs-constant table reads **"same"** in every architecture-column cell,
  with only the nouns differing.
- The verdict is **re-bind, not rebuild**, naming the re-authored noun layer and
  the carried-over architecture explicitly.
- `NOTES.md` answers the three prompts: which element felt least obviously
  domain-independent before mapping and what convinced you it carried over (the
  hydration distillation and the event trigger are common answers), where (if
  anywhere) the non-SDLC domain genuinely needed something new and whether that is
  a new architecture element or just a new noun (the higher-stakes incident gate is
  a tighter gate, not new architecture), and what would force a rebuild had the
  SDLC platform been built with no architecture (a wired-up single-domain tool
  re-implements everything; the pattern-based design re-binds).
