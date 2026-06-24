# mod-310-agentic-developer-platforms/exercise-03-workflow-toolchain-integration — Solution

## Approach

This is an integration-contract exercise: choose the right interface (REST,
GraphQL, or event-driven) **per integration** by access shape, shape one GraphQL
read that collapses an N+1 cascade, and design a webhook receiver that survives
retries, duplicates, and out-of-order delivery. The contract must be
**vendor-neutral by construction** — design to the toolchain *category*, then note
a product as an example.

The worked solution names the four categories generically and gives a peer set of
example products for each, so no single vendor anchors the design:

- **Version-control / work source** — e.g. GitHub, GitLab, or an internal VCS.
- **Issue / work tracker** — e.g. Jira, Linear, or an internal tracker.
- **Knowledge base** — e.g. Confluence, Notion, or an internal wiki.
- **CI/CD or pipeline** — e.g. GitHub Actions, GitLab CI, or an internal pipeline.

The non-SDLC mapping (monitoring / ticketing / runbooks / automation) is carried
in the stretch section to prove the interface-selection *rule* is unchanged.

The design choices that drive everything below:

- **Access shape decides the interface, not the vendor.** A discrete write or a
  shallow read → REST; a deep related read in one round-trip → GraphQL; a reaction
  to an external change → event. This rule is applied per integration point.
- **The reactive trigger is an event, never a poll.** Polling is either too slow
  (long interval) or too expensive and rate-limited (short interval), and it still
  races the change. An event delivers the change once, immediately.
- **The receiver is the hard part.** Signature verification, fast-return +
  async work, dedupe on delivery ID, and tolerate ordering/loss are the four
  non-negotiables — a naive inline receiver violates the second and fourth and
  causes duplicate agent runs.
- **Every integration is bound behind an MCP server** so the binding stays
  host-portable (exercise-01) and the credential stays out of the model's context
  (exercise-02).

## Reference solution

### 1. Interface-selection table

| Integration point | Access shape | Interface | Why this interface (not the others) |
| --- | --- | --- | --- |
| Learn a new unit of work appeared (change opened / ticket created) | React to external change | **Event** | The platform must learn *immediately*; polling either lags or hammers the API and still races. An event delivers the change once, on occurrence. |
| Gather the work item + its related items (reviews/checks/linked issue) | Deep related read, one round-trip | **GraphQL** | One query returns the item and all linked items with exactly the fields needed; REST would be an N+1 cascade, and an event carries no rich read. |
| Comment on the item | Discrete write | **REST** | A single, idempotent mutation to one resource. No deep read, no reaction — REST is the simplest correct fit. |
| Transition the item's state | Discrete write | **REST** | Same: one discrete mutation; GraphQL is overkill, an event is the wrong direction. |
| React when a downstream step finishes (CI completes / dependent task closes) | React to external change | **Event** | The completion happens outside the agent; the agent must be *told*, not poll. |

**All three styles used:** REST (comment, transition), GraphQL (the deep related
read), event (new-work trigger and downstream-completion trigger).

**Why the reactive point is NOT polling.** To learn about a new change/ticket by
polling, you choose an interval. A *long* interval means the agent reacts minutes
late — the work sits idle and the user notices. A *short* interval means thousands
of "anything new?" calls per hour against the tracker, burning the rate budget for
no result on the vast majority of polls, and you *still* race the moment of
creation. An event is delivered exactly when the change occurs, once, with the
payload — strictly better on latency, cost, and rate budget. Polling is only
defensible as a *backstop* when the event channel is down, never as the primary.

### 2. Shape the GraphQL read

The "gather rich related context" step in **one** round-trip — fetch the work item
and its reviews, checks, and linked issue, selecting only the fields the agent
needs:

```graphql
query WorkItemContext($id: ID!) {
  workItem(id: $id) {
    id
    title
    state
    author { login }
    reviews(first: 20) {
      nodes { reviewer { login } state submittedAt }
    }
    checks(first: 20) {
      nodes { name conclusion completedAt }
    }
    linkedIssue {
      id
      title
      state
      labels(first: 10) { nodes { name } }
    }
  }
}
```

**The N+1 REST cascade it replaces:**

```text
   REST cascade (sequential, N+1):
     1  GET /work-items/{id}                  → the item
     2  GET /work-items/{id}/reviews          → review list
     3..  GET /reviews/{rid}  (per review)    → N calls for reviewer/state
     k  GET /work-items/{id}/checks           → check list
     k+1  GET /work-items/{id}/linked-issue   → the linked issue
     k+2  GET /issues/{iid}/labels            → its labels
   → 1 + 1 + N(reviews) + 1 + 1 + 1 round-trips ≈ 5 + N
```

With, say, 8 reviews, the REST path is **~13 round-trips**; the GraphQL path is
**1**. Round-trips saved: **~12**, collapsed to a single request.

**Point/cost budget — no over-selection.** The query selects only the fields the
downstream decision uses (state, reviewer + review state, check name + conclusion,
linked-issue state + labels). It does *not* pull review bodies, full diffs, commit
histories, or comment threads the agent will not read. This keeps the GraphQL
point cost low **and** is a context-hydration win (exercise-02): the field
selection *is* the L3 distillation — nothing over-fetched lands in the model's
context.

### 3. Robust event receiver

| Requirement | Your design | What breaks without it |
| --- | --- | --- |
| **Signature verification** | Verify the HMAC signature on every inbound event against the shared secret *before* parsing the body; reject (401) unverified senders | Anyone who learns the URL can forge events and drive the agent — a spoofed "CI passed" could trigger a privileged action |
| **Fast return + async work** | Verify, enqueue `{delivery_id, payload}`, return `200` in milliseconds; a separate worker runs the agent | Running the agent inline holds the connection for seconds–minutes; the sender times out, marks delivery failed, and **retries** → duplicate runs |
| **Dedupe on delivery ID** | The worker checks the delivery ID against a seen-set (with TTL) and drops duplicates before running the agent | Retries and network duplicates each fire a full agent run — double comments, double transitions |
| **Tolerate ordering / loss** | Treat the event as a *hint to re-check current state* via the GraphQL read, not as truth; never act on the payload's stale snapshot | Out-of-order or lost events make the agent act on a state that no longer holds (e.g. "CI failed" after a re-run already passed) |

**The flow — and where a naive inline design duplicates runs:**

```text
   event in ─▶ verify signature ─▶ enqueue {delivery_id, payload} ─▶ return 200
                                              │
                                              ▼
                              worker ─▶ dedupe(delivery_id)? ──seen──▶ drop
                                              │ new
                                              ▼
                              re-check current state (GraphQL) ─▶ run agent

   NAIVE (inline) — causes duplicate runs:
     event in ─▶ verify ─▶ RUN AGENT (seconds–minutes) ─▶ 200
                                    ▲
                 sender times out waiting for 200 ─▶ RETRIES ─▶ second full run
                 (duplicate comment / duplicate transition)
```

The duplicate-run hazard lives precisely at "run the agent *before* returning
`200`": the slow handshake provokes the sender's retry, and with no dedupe each
retry is a fresh run. Fast-return + enqueue + dedupe-on-delivery-ID closes it.

### 4. Tie back to portability and credentials

Each integration is bound **behind an MCP server**, one per category:

```text
   vcs-mcp        → version-control/work-source API   (token in the server)
   tracker-mcp    → issue/work-tracker API            (token in the server)
   kb-mcp         → knowledge-base API                (token in the server)
   ci-mcp         → CI/pipeline API + webhook source  (secret in the server)
```

- **Host-portable (exercise-01):** the MCP servers are self-owned and speak an
  open protocol, so any host (LangGraph, a custom host, Claude Code) binds them
  with one client-registration line. The interface choice (REST/GraphQL/event)
  lives *inside* the server, invisible to the host.
- **Credential out of context (exercise-02):** every API token and webhook signing
  secret lives in its MCP server's config, never in the model's prompt. The model
  sees only scoped tool schemas (`comment`, `transition`, `workItemContext`).

### Stretch: rate budget, idempotent write, and the non-SDLC re-bind

**Rate-budget analysis.** The GraphQL query has a point cost roughly proportional
to the connections traversed (item + ~20 reviews + ~20 checks + 1 linked issue +
its labels) — call it a few dozen points on a typical points-per-hour budget. The
REST cascade is **~5 + N discrete calls**, each counted against a calls-per-hour
limit. For trackers that bill per-call (the common case), the single GraphQL query
is far cheaper than 13 REST calls; only on a backend with an unusually punitive
per-field GraphQL cost and generous REST limits would the cascade win — so the
shaped query is the right default and the analysis names the one exception.

**Idempotent discrete write.** "Comment on the item" must not double-post when an
event is re-driven. Give each agent-generated comment a deterministic idempotency
key — `hash(work_item_id, run_id, comment_purpose)` — and either (a) pass it as the
tracker's client-request-id where supported, or (b) before posting, query for an
existing comment carrying that key marker and skip if present. A re-driven event
then re-computes the *same* key and the second post is a no-op. This reuses the
at-least-once discipline from durable execution: design the write so running it
twice equals running it once.

**Non-SDLC re-bind (rule unchanged).** Mapping the four categories to ops:

| SDLC category | Non-SDLC (ops) category | Interface (by access shape) |
| --- | --- | --- |
| New change/ticket appears | New **alert** fires | **Event** (not poll) |
| Item + reviews/checks/linked issue | Incident + related alerts + recent deploys | **GraphQL** (deep related read) |
| Comment / transition | Acknowledge / annotate / update status | **REST** (discrete writes) |
| CI completes | Remediation job / runbook step completes | **Event** |

The interface-selection rule — discrete write → REST, deep related read → GraphQL,
react → event, never poll the reactive trigger — is **identical**. Only the nouns
(alert, incident, runbook) changed.

## Meeting the acceptance criteria

- **Interface-selection table covers every point, uses all three styles, justifies
  by access shape, reactive point not polled.** Task 1's table maps five points,
  uses REST (comment, transition), GraphQL (related read), and event (two
  triggers), justifies each by access shape, and argues the reactive point against
  polling explicitly.
- **Shaped GraphQL fetches item + related items in one round-trip; N+1 cascade
  shown with a count; no over-selection.** Task 2 gives the single query, shows the
  ~5+N (≈13) REST cascade it replaces (~12 round-trips saved), and constrains field
  selection to what the decision uses.
- **Event receiver satisfies all four requirements; flow marks the duplicate-run
  point.** Task 3's table covers signature verification, fast-return + async,
  dedupe-on-delivery-ID, and tolerate-ordering/loss, and the diagram marks the
  inline "run before 200" point as the duplicate-run hazard.
- **Each integration bound behind an MCP server with the credential out of
  context.** Task 4 binds one MCP server per category, keeping the binding portable
  and every token/secret in the server, not the prompt.

## Common pitfalls

- **Polling the reactive trigger.** The most common error: a cron that asks "any
  new work?" Either it lags or it burns the rate budget, and it still races. The
  trigger is an event.
- **Running the agent inline before returning `200`.** This is the duplicate-run
  generator: the slow handshake triggers sender retries, and without dedupe each
  retry is a full run. Verify → enqueue → `200` → worker.
- **Skipping dedupe because "retries are rare."** Retries are *guaranteed* under
  at-least-once delivery; network duplicates happen even without timeouts. Dedupe
  on the delivery ID is not optional.
- **Trusting the event payload as truth.** Out-of-order or lost events mean the
  payload's snapshot may be stale. Treat the event as a hint and re-check current
  state via the shaped read.
- **Over-selecting GraphQL fields.** Pulling diffs, bodies, and histories the agent
  never reads inflates point cost *and* pollutes the context. Select only the
  decision fields.
- **Choosing the interface by vendor habit instead of access shape.** "We always
  use REST" turns a one-round-trip read into an N+1 cascade. Let the access shape
  decide.

## Verification

A completed submission is correct when:

- The interface-selection table covers **every** integration point, uses **all
  three** styles at least once, justifies each by access shape, and explicitly
  argues the reactive point is an event, not a poll.
- The GraphQL query fetches the item **and** its related items in **one**
  round-trip, the replaced REST cascade is shown with a round-trip count
  (~5 + N → 1), and the field selection is limited to decision-relevant fields.
- The receiver design satisfies **all four** requirements, and the flow diagram
  marks where a naive inline ("run before 200") design causes duplicate runs.
- Every integration is bound behind an **MCP server**, with each token/secret in
  the server and **none** in the model's context.
- `NOTES.md` answers the three prompts: the genuinely ambiguous point and what
  tipped it (the "gather context" read can look REST-able until the linked-item
  depth forces GraphQL), the three-retry trace without dedupe (three full agent
  runs → fixed to one by dedupe-on-delivery-ID), and where polling was "good
  enough" but the event won anyway (the latency/rate-budget argument for the
  new-work trigger).
