# mod-310-agentic-developer-platforms/exercise-04-devops-toolchain-integration — Solution

## Approach

This exercise architects the platform's **integration layer** for one SDLC loop:
*when a Jira ticket is assigned to the bot, implement it and open a PR; when CI
fails, debug; when green, comment back.* The governing rule is the interface
selection rule: **REST for writes and shallow reads; GraphQL for deep/related
reads in one round-trip; events for reacting to change without polling.** Most
real platforms use all three, and the architect's job is choosing correctly per
integration and justifying each by the shape of the data and the trigger.

I work it in five passes mapped to the tasks:

1. An interface choice per integration, justified in one line.
2. The event-driven backbone diagram, labeling each arrow REST / GraphQL / event.
3. Webhook robustness: enqueue-and-return-fast, verify signatures, deduplicate
   on delivery ID.
4. Rate limits as a first-class design input, with named mitigations.
5. The tie-back to exercise-02: every integration is reached through an MCP
   tool, and every webhook payload is untrusted input.

The two required highlights — one GraphQL query collapsing an N+1 REST cascade,
and one event replacing a polling loop — are called out explicitly. The chosen
toolchain is **GitHub + Jira + Confluence + GitHub Actions**.

## Reference solution

### 1. Interface choice per integration

| Integration | Interface | Why (selection rule) |
| ----------- | --------- | -------------------- |
| Ticket assigned to bot | **Jira webhook (event)** | React to a state change; polling Jira for "anything assigned yet?" is the anti-pattern. |
| Read ticket | **Jira REST** (`get_issue`) | A simple, shallow read. |
| Read linked design page | **Confluence REST** (`get_page`) | A simple, shallow read. |
| Gather code context | **GitHub GraphQL** | Deep, related read — repo + relevant files + the linked issue + its status — in **one** round-trip; avoids the N+1 REST cascade. |
| Open PR | **GitHub REST** (`POST /pulls`) | A write; REST is the right default for writes. |
| CI finished / failed | **GitHub Actions webhook (event)** | React to completion; **replaces a polling loop** ("did CI finish yet?"). |
| Read failing job logs | **GitHub Actions REST** (`get_job_log`) | A shallow read of one job's output. |
| Comment back on ticket | **Jira REST** (`add_comment`) | A write. |
| Transition ticket status | **Jira REST** (transition) | A write along an allowlisted transition. |

**The GraphQL win (N+1 → 1).** Gathering code context the REST way is a cascade:
`GET /repos/{r}` → `GET /repos/{r}/contents/...` per file → `GET /issues/{n}` for
the linked issue → `GET /issues/{n}` again for its status/labels. That is N+1
calls and N+1 round-trips. One GitHub GraphQL query asks for exactly the
repository, the named blobs, the linked issue, its state, and its labels in a
single request:

```graphql
query GatherContext($owner: String!, $repo: String!, $issue: Int!) {
  repository(owner: $owner, name: $repo) {
    name
    defaultBranchRef { name }
    object(expression: "HEAD:src/billing/charge.ts") {
      ... on Blob { text }
    }
    issue(number: $issue) {
      title
      body
      state
      labels(first: 10) { nodes { name } }
    }
  }
}
```

One round-trip, exactly the fields needed, no over- or under-fetching.

**The event win (polling → push).** Detecting CI completion by calling
`GET /actions/runs` in a loop wastes rate budget and adds latency between "CI
done" and "agent reacts." The `workflow_run` / `check_run` **completed** webhook
pushes the result the instant it happens — zero polling.

### 2. The event-driven backbone

```text
  Jira: ticket assigned to @agent-bot
        │ (event: webhook)
        ▼
  ┌───────────────────────────────────────────────┐
  │  [verify signature]                            │   ← untrusted input (ch.2)
  │  [dedup on delivery id]                        │
  │  [enqueue task, return 200 fast]               │
  └───────────────────┬───────────────────────────┘
                      │  worker pulls task off queue
                      ▼
              run the SDLC agent (boundaries from exercise-02)
                      │
        ┌─────────────┼───────────────────────────────┐
        │ (REST)      │ (GraphQL)        │ (REST)       │
        ▼             ▼                  ▼              ▼
  Jira get_issue  GitHub gather    GitHub POST     Confluence
  (read ticket)   code context     /pulls          get_page
                  (one query)      (open PR)        (read design)
                      │
                      ▼  PR opened, CI starts
  GitHub Actions: "workflow_run completed"
        │ (event: webhook)
        ▼
  ┌───────────────────────────────────────────────┐
  │  [verify sig] [dedup] [enqueue, return 200]    │
  └───────────────────┬───────────────────────────┘
                      ▼
        conclusion == failure ──▶ enqueue debug task (read logs via REST)
        conclusion == success ──▶ Jira add_comment (REST): "PR green: <url>"
```

Solid arrows into external systems are labeled REST or GraphQL; the two inbound
triggers (`ticket assigned`, `workflow_run completed`) are events. The agent
runs *off the queue*, never inline in the webhook handler.

### 3. Webhook robustness

Three properties make the backbone survive production:

- **Enqueue and return `200` fast.** The webhook handler does signature
  verification, dedup, and an enqueue — then returns `200` in milliseconds. It
  does **not** run the multi-minute agent inline. Webhook senders (Jira, GitHub)
  time out and retry on slow responses and have delivery limits; absorbing the
  work into a queue decouples the slow agent from the fast HTTP handshake.
- **Verify signatures.** GitHub signs each delivery with
  `X-Hub-Signature-256` (HMAC-SHA256 over the raw body using the webhook
  secret); Jira webhooks are verified by a shared secret / JWT depending on
  config. The handler recomputes and constant-time-compares before trusting the
  payload — the event is an untrusted input (exercise-02).
- **Deduplicate on delivery ID.** At-least-once delivery means duplicates
  *will* arrive (sender retries, network re-delivery). Key on the delivery ID
  (`X-GitHub-Delivery` / Jira's event ID) in a short-TTL dedup store and drop
  repeats — otherwise the agent opens the same PR twice.

```text
on webhook:
  1. read raw body
  2. verify signature  -> fail => 401, do not enqueue
  3. delivery_id = header; if seen(delivery_id) => 200, drop (dedup)
  4. mark seen(delivery_id, ttl=24h)
  5. enqueue {task, payload}
  6. return 200            # fast; agent runs off the queue
```

### 4. Rate limits as a design input

The platform is chattiest at **code-context gathering** (potentially many file
reads per task) and under **fan-out** (many tickets assigned at once). Treat the
rate budget as a constraint up front:

- **Batch with GraphQL.** The Section 1 GraphQL query collapses the N+1 file/
  issue cascade into one request, cutting REST call count dramatically. GitHub
  GraphQL uses a **point budget** (each query costs points based on the nodes it
  requests); the gather query above is a handful of points, well within budget.
- **Cache stable reads.** The repo's default branch, the CI config, and
  rarely-changing files are cached across tasks (invalidated on a new commit),
  so repeated tasks don't re-fetch them.
- **Back off on `429`.** Honor `Retry-After` and the `X-RateLimit-Reset` header
  with exponential backoff; the worker pauses rather than hammering. The queue
  absorbs the delay without dropping work.
- **Bound concurrency.** The worker pool has a fixed size, so a burst of 40
  assigned tickets becomes a paced drain on the rate budget, not a thundering
  herd.

### 5. Trust-boundary connection

- Every integration is reached through an **MCP tool** (`jira.get_issue`,
  `github.gather_context`, `github.create_pr`, `ci.get_job_log`), never a raw
  HTTP call from the agent. The MCP server wraps each interface with the
  least-privilege credential and the boundary controls of exercise-02 — the
  agent never holds a token.
- Every **webhook payload is untrusted input.** A ticket body or PR comment in
  the payload can carry injection; it is signature-verified at the edge and then
  treated as adversarial context inside the agent (exercise-02), never as
  trusted instructions.

### 6. The spec

| Trigger / action | Interface | Calls | Why this interface |
| ---------------- | --------- | ----- | ------------------ |
| Ticket assigned to bot | Jira **webhook** | enqueue task | react to change, don't poll |
| Read ticket + design | Jira / Confluence **REST** | `get_issue`, `get_page` | shallow reads |
| Gather code context | GitHub **GraphQL** | repo + files + linked issue + status, one query | deep related read, no N+1 |
| Open PR | GitHub **REST** | `POST /pulls` | a write |
| CI finished / failed | GitHub Actions **webhook** | enqueue debug task | react to completion, no polling |
| Read failing logs | GitHub Actions **REST** | `get_job_log` | shallow read |
| Comment back on ticket | Jira **REST** | `add_comment` | a write |

### Stretch: stale-task race, dead-letter, GraphQL point budget

**"Reassigned away mid-run."** If the ticket is reassigned off the bot while a
task is in flight, a Jira `issue_updated` webhook fires with the new assignee.
The worker checks the ticket's *current* assignee (a cheap REST read) at each
durable step (before opening the PR); if it is no longer the bot, the task is
**cancelled** and the branch discarded. Tasks carry the assignment's version, so
a stale task that slips through is recognized and dropped before it writes.

**Dead-letter handling.** An event that fails processing repeatedly (3 attempts
with backoff) goes to a **dead-letter queue** with the original payload, the
error, and the attempt count. A platform-team alert fires on any DLQ arrival;
the on-call triages, fixes the cause, and replays the event. Nothing fails
silently into the void.

**GraphQL point budget.** The Section 1 gather query requests one repository
node, one blob, one issue, and up to 10 labels — on GitHub's point formula this
is on the order of a single-digit point cost per call, far under the hourly
budget even at high task fan-out. Compared to the REST cascade (N+1 separate
rate-limited requests), the GraphQL path is both cheaper in point terms and
faster in round-trips.

## Meeting the acceptance criteria

- **Every integration's interface justified in one line by the rule.** Section 1
  and the Section 6 spec give each row a one-line justification keyed to "writes/
  shallow reads → REST, deep reads → GraphQL, react-to-change → events."
- **≥1 GraphQL-replaces-N+1 and ≥1 event-replaces-polling, called out.** The
  code-context GraphQL query (with the explicit cascade it replaces) and the CI
  `workflow_run` webhook (replacing the `GET /runs` poll loop) are both named.
- **Backbone enqueues-and-returns-fast, verifies signatures, dedups on delivery
  ID.** Section 3 specifies the handler steps and the dedup keying.
- **Rate limits named as a constraint with concrete mitigations.** Section 4
  identifies the chattiest points and lists GraphQL batching, caching, `429`
  backoff, and bounded concurrency.
- **Each integration via MCP; payloads untrusted.** Section 5 confirms both.

## Common pitfalls

- **Polling for completion.** Looping `GET /actions/runs` to detect CI finishing
  wastes rate budget and adds latency. Subscribe to the `workflow_run`/
  `check_run` completed webhook instead.
- **Running the agent inline in the webhook handler.** A 5-minute agent inside
  the HTTP handler times the webhook out; the sender retries, and now you have
  *two* runs opening *two* PRs. Enqueue and return `200` fast; run the agent off
  the queue.
- **Skipping signature verification.** An unsigned/unverified webhook is an open
  door for forged events. Verify the HMAC before trusting any field.
- **No dedup.** At-least-once delivery guarantees duplicates; without dedup on
  the delivery ID the agent acts twice. Key and drop repeats.
- **N+1 REST where GraphQL fits.** Fetching repo, then each file, then the
  linked issue, then its status as separate REST calls burns round-trips and
  rate budget. Collapse related reads into one GraphQL query.

## Verification

- **Interface-rule audit.** For every row in the Section 6 spec, confirm the
  interface matches the rule (writes → REST, deep related read → GraphQL,
  react-to-change → event); flag any row where a polling REST loop or an N+1
  cascade crept back in.
- **GraphQL query check.** Run the gather query against a real repo via the
  GitHub GraphQL explorer and confirm it returns the repo, blob, linked issue,
  and issue status in one response, and inspect the reported point cost.
- **Webhook robustness test.** Send a duplicate delivery (same `X-GitHub-
  Delivery`) and confirm the second is dropped; send a payload with a bad
  signature and confirm it is rejected with `401` and never enqueued; send a
  valid one and confirm the handler returns `200` before the agent runs.
- **Backpressure test.** Assign a burst of tickets and confirm the worker pool
  paces the rate budget (no `429` storm) and that `Retry-After` is honored.
- **Stale-task test (stretch).** Reassign a ticket mid-run and confirm the task
  is cancelled before it opens a PR; force three processing failures and confirm
  the event lands in the DLQ with an alert.
- **Markdownlint** clean: one H1, every fence tagged (`text`/`graphql`), blank
  lines around blocks, dash bullets, trailing newline.
