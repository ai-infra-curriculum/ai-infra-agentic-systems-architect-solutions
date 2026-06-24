# mod-302-multi-agent-orchestration/exercise-03 — Solution

## Approach

Every edge in this system gets run through one test from Chapter 3: *could the other
side be a deterministic function?* If yes, it is a capability — **MCP**. If it has to
reason, plan, or use its own tools to satisfy the request, it is a delegation —
**A2A**. The exercise is built so four of the five edges pass the function test
cleanly (metrics, logs, tickets, DB-health are all "call this, get data back") and
exactly one fails it (the security-analysis agent, owned by another team, is a black
box that reasons over the incident). That asymmetry is the whole point: A2A earns its
weight at the *organizational* boundary, and nowhere else here.

The design choices that drive the spec:

- **Four MCP edges, one A2A edge.** The metrics store, logging system, ticketing
  read/comment, and DB health check are all capabilities the assistant invokes — each
  could be a deterministic function (a query, a read, a scoped write, a health probe).
  They are MCP. The security-analysis agent cannot be a function: it plans its own
  investigation with its own tools and returns a judgment, not a fixed result. It is
  A2A, and it is A2A specifically because it is *another team's* agent — a contract
  that must survive them rewriting their internals.
- **Draw both seams on the delegated agent.** The security agent reaches *its* tools
  over MCP while the assistant delegates *to it* over A2A. The diagram shows both,
  because conflating "the thing I delegate to" with "the things it uses" is the
  Chapter 3 mistake the exercise is testing.
- **Least privilege per edge, with the cross-team peer as the highest-risk one.**
  Metrics/logs/DB-health run read-only; ticketing runs scoped write (comment only, no
  close/delete); the A2A peer runs autonomously on the assistant's behalf and is
  assumed to be able to fail, stall, or return garbage. The human gate before any
  remediation is the backstop that keeps the autonomous peer from causing irreversible
  action without review.
- **Trace IDs cross both seams.** A single `incident_id` / `trace_id` is injected at
  the assistant and propagated as a tool-call argument on every MCP call and as task
  metadata on the A2A task, so one incident request is traceable end-to-end —
  otherwise a failure inside the delegated agent is invisible from the caller
  (Chapter 3, observability across the seam).

The tempting-but-wrong classification, surfaced in the reflection, is the DB health
check: "it's a database, and the security agent talks to a database, so maybe it's an
agent too." No — the DB health check is a fixed read-only probe with a known output
shape. It could be a function. It is MCP. The thing that makes the security edge A2A
is *autonomous reasoning behind a team boundary*, not the presence of a database.

## Reference solution

### Edge inventory and classification

```text
| Edge                   | MCP or A2A | "Could be a function?"            | Contract artifact | Trust scope        |
| ---------------------- | ---------- | -------------------------------- | ----------------- | ------------------ |
| metrics store          | MCP        | yes — a query returns a series   | tool schema       | read-only          |
| logging system         | MCP        | yes — a query returns log lines  | tool schema       | read-only          |
| ticketing read/comment | MCP        | yes — read + append a comment    | tool schema       | scoped write       |
|                        |            |   (no close/delete/transition)   |                   |  (comment only)    |
| DB health check        | MCP        | yes — fixed read-only probe      | tool schema       | read-only          |
| security-analysis agent| A2A        | NO — plans + reasons w/ own tools| capability card   | cross-team (other  |
|                        |            |   behind a team boundary         |                   |  team's autonomy)  |
```

### Two-axis architecture

```text
                          ┌──────────────────────────────┐
   A2A (delegation)       │  incident-response assistant  │       MCP (capabilities)
   ┌──────────────────────┤            (you)              ├──────────────────────────┐
   │                      └──────────────────────────────┘                          │
   ▼                                                                                 ▼
┌────────────────────┐                                          ┌──────────────────────────┐
│ security-analysis  │  task{incident, scope, trace_id}         │  metrics store     (RO)  │
│ agent (other team) │  ──submitted→working→                    │  logging system    (RO)  │
│                    │     input-required?→completed/failed     │  ticketing read/comment  │
│   │ MCP to ITS     │  ◀── result{findings, severity}          │       (scoped write: RW*)│
│   ▼ own tools      │                                          │  db-health check   (RO)  │
│  (its threat intel,│                                          └──────────────────────────┘
│   its sandbox, ...)│
└────────────────────┘            ┌──────────────────────────────────────────────┐
                                  │  HUMAN GATE before ANY remediation action      │
                                  │  (no MCP write beyond ticket comments, and no  │
                                  │   A2A-recommended action, executes ungated)    │
                                  └──────────────────────────────────────────────┘

   trace_id is injected once at the assistant and propagated on every MCP call
   argument AND as A2A task metadata — one incident is traceable across both seams.
```

The delegated security agent shows **both** seams: the assistant reaches it over A2A,
and it reaches its own tools (threat intel, sandbox, whatever it owns) over MCP. The
assistant never sees those tools — only the capability card and the task result.

### Contracts

```text
MCP edges (tool/resource contracts)

  metrics.query
    inputs:  { metric, window, filters, trace_id }
    outputs: { series[], unit }
    errors:  TIMEOUT, BAD_QUERY, UPSTREAM_DOWN
    side-effects: none (read-only)
    scope:   read-only credential, metrics namespace only

  logs.search
    inputs:  { query, window, limit, trace_id }
    outputs: { lines[], truncated: bool }
    errors:  TIMEOUT, BAD_QUERY, RATE_LIMITED
    side-effects: none (read-only)
    scope:   read-only credential, logs index only

  tickets.read / tickets.comment
    inputs:  read{ ticket_id, trace_id } ; comment{ ticket_id, body, trace_id }
    outputs: read{ ticket } ; comment{ comment_id }
    errors:  NOT_FOUND, FORBIDDEN, RATE_LIMITED
    side-effects: comment APPENDS only — cannot close, transition, or delete
    scope:   scoped write: comment permission only, on the incident's project

  db.health_check
    inputs:  { target_db, trace_id }
    outputs: { replication_lag, connection_count, slow_query_count, status }
    errors:  TIMEOUT, UNREACHABLE
    side-effects: none (read-only probe; runs no DDL/DML)
    scope:   read-only health role, no data access

A2A edge (capability card + task lifecycle)

  peer: security-analysis-agent (other team)
  capability card advertises:
    - "analyze a suspected-security incident given scope + evidence pointers"
    - inputs it accepts, output schema it returns, its SLA / typical latency
    - it does NOT advertise its internal tools (opaque by design)
  task payload:
    task{ incident_summary, evidence_pointers[], scope, trace_id }
  task lifecycle the caller depends on:
    submitted → working → [input-required → working]* → completed | failed
  result schema:
    { findings[], severity, recommended_actions[], confidence }
    (recommended_actions are RECOMMENDATIONS — they pass through the human gate,
     they never auto-execute)
```

### Trust boundaries and blast radius

```text
| Edge                | Least-privilege scope        | If it fails / stalls / returns garbage          |
| ------------------- | ---------------------------- | ----------------------------------------------- |
| metrics store       | read-only, metrics ns        | retry once; then proceed w/ "metrics unavailable"|
| logging system      | read-only, logs index        | retry once; then proceed w/ "logs unavailable"  |
| ticketing           | comment-only, this project   | surface to human; never silently drop a comment |
| db health check     | read-only health role        | mark health "unknown"; do not block triage      |
| security agent (A2A)| cross-team; autonomous       | HIGHEST RISK — see below                        |
```

The cross-team A2A peer is the highest-risk edge and is handled explicitly:

- **Bounded task wait.** The assistant sets a deadline on the A2A task. If the peer
  stays in `working` past the deadline (or `input-required` goes unanswered), the
  assistant abandons the task, records "security analysis incomplete," and steps up
  the escalation ladder (Chapter 4) — it does **not** wait forever and it does **not**
  fabricate the security verdict.
- **Garbage tolerance.** The peer's result is validated against the advertised output
  schema; a malformed or low-`confidence` result is treated as "no usable security
  finding," not as a clean negative.
- **No auto-action.** Every `recommended_action` from the peer is a recommendation
  that passes through the human gate. The blast radius of a compromised or buggy peer
  is therefore bounded to "produces bad recommendations a human reviews," never
  "executes remediation."

### Observability across seams

A single `trace_id` (== `incident_id`) is generated at the assistant when the incident
opens. It is threaded as an explicit argument on every MCP tool call and attached as
metadata on the A2A task. The security agent is contractually expected to propagate it
into its own MCP calls so its internal spans roll up under the same incident. Result:
one incident request produces one connected trace spanning the assistant's MCP calls
and the delegated A2A task, so a failure *inside* the delegated agent surfaces in the
assistant's incident timeline instead of vanishing behind the team boundary.

## Meeting the acceptance criteria

- **Every edge classified MCP/A2A with the function test** — four MCP (each "could be
  a function"), one A2A (the security agent fails the test: autonomous reasoning
  behind a team boundary). The classification table states the test outcome per edge.
- **Diagram shows both axes and the delegated agent's own MCP** — the A2A axis on the
  left, the MCP axis on the right, and the security agent drawn with *its own* MCP
  seam to its internal tools.
- **MCP edges have full contracts; A2A edge has card + lifecycle** — each MCP tool
  lists inputs/outputs/errors/side-effects/scope; the A2A edge lists the capability
  card summary, task payload, and the `submitted → working → input-required →
  completed/failed` lifecycle.
- **Least-privilege + failure-coping per edge, A2A peer highest-risk** — the trust
  table gives a scope and a failure response for every edge, and the cross-team peer
  gets bounded-wait, schema validation, and no-auto-action coping called out
  explicitly.
- **Trace-ID propagation across both seams** — one `trace_id` injected at the
  assistant, carried on every MCP argument and as A2A task metadata, propagated by the
  peer into its own calls.

## Common pitfalls

- **Modeling a capability as an agent.** Wrapping the DB health check or the metrics
  query as a "monitoring agent" buys an autonomous black box and a heavier task
  lifecycle for a deterministic probe (Chapter 3: "don't reach for A2A when MCP
  suffices"). If it could be a function, it is MCP. Reserve A2A for the team boundary.
- **Forgetting the delegated agent's own MCP seam.** A diagram that shows only the
  A2A edge to the security agent and stops there hides half the architecture. The peer
  uses its own tools over MCP; draw both seams or you have mismodeled the system.
- **Over-scoping the ticketing credential.** "Scoped write" must mean comment-only.
  A credential that can close, transition, or delete tickets gives the assistant (and
  anything that compromises it) far more blast radius than the task needs. Scope to
  the single operation the workload requires.
- **Trusting the A2A result like an MCP return.** An MCP tool you own returns a known
  shape; an autonomous cross-team peer can stall or return garbage. Validate its
  result against the advertised schema, bound the wait, and route its recommendations
  through the human gate — never auto-execute them.
- **Losing the trace at the seam.** If the `trace_id` is dropped when crossing into
  the A2A peer, a failure inside the peer is invisible from the assistant's timeline.
  Make trace propagation a contractual term of the capability card, not a hope.

## Verification

A completed submission is correct when:

- The classification table marks exactly the security-analysis agent as A2A and the
  other four edges as MCP, each with an explicit could-this-be-a-function judgment —
  and the A2A justification is the *team boundary + autonomous reasoning*, not merely
  "it's complicated."
- The diagram has two distinct axes and draws the delegated agent's *own* MCP seam to
  its internal tools.
- Every MCP edge lists inputs, outputs, errors, side-effects, and a least-privilege
  scope; the ticketing edge is comment-only and the read paths are read-only.
- The A2A edge lists a capability-card summary, a task payload, the full lifecycle,
  and a failure-coping plan (bounded wait → step up the ladder; schema validation; no
  auto-action) that names it the highest-risk edge.
- A single `trace_id` is shown injected once and propagated across both MCP calls and
  the A2A task.
- `NOTES.md` answers the three prompts: the edge that tempts an A2A label but is MCP
  (the DB health check — a database is not an agent); what in the contract survives the
  peer rewriting its internals (the capability card — you depend on the card, not the
  implementation) vs. what breaks (a change to its advertised output schema or
  lifecycle); and what happens if the peer stalls in `working` forever (bounded wait →
  abandon → escalate per Chapter 4, never fabricate the security verdict).
