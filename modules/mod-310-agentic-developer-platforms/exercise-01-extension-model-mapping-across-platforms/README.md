# mod-310-agentic-developer-platforms/exercise-01-extension-model-mapping-across-platforms — Solution

## Approach

This is a placement-and-grading exercise, not a build. The work is to take one
capability set and decide, for each capability, *which kind of extension point it
is* on the generic model (agent / skill / tool / hook-lifecycle, with MCP as the
external-access mechanism), then bind that placement across **two or more host
platforms treated as peers** and grade how much each binding ports versus locks
in. The reference treats **Claude Code, Cursor, GitHub Copilot, LangGraph
Platform, and a custom in-house host as five interchangeable hosts** — the
placement decision is upstream of all of them, and no single host is canonical.

The chosen peer set for the worked mapping is **LangGraph Platform (Platform A),
a custom in-house host (Platform B), and Claude Code (Platform C, optional
column)**. This deliberately satisfies the constraint "at least one of your two
must be a custom host *or* a platform other than Claude Code" by making the two
primary columns a framework host and a self-built host — Claude Code is present
only as a third peer to show the mapping is host-shaped, not vendor-shaped.

The design choices that drive everything below:

- **Capability *kind* decides the extension point, not the platform.** A
  guarantee is a hook; external access is MCP; procedural know-how is a skill; a
  callable pure function is a tool; an isolated role with its own prompt and
  permissions is an agent. This classification is done once, host-free, in task 1.
- **The two classic misplacements are the always-run linter and the internal
  service.** Both *look* like skills ("run the linter," "here is how to call the
  deploy API") and both are wrong as skills — a skill is model-discretionary
  know-how the model *may* follow, which is exactly what neither of these can be.
- **Portability tracks the openness of the binding's substrate.** An MCP-backed
  capability rides an open protocol and ports nearly unchanged; a hook rides each
  host's proprietary lifecycle and is the least portable.
- **Lock-in is measured as exit cost**, not as a feeling — what would have to be
  re-authored, re-reviewed, or re-guaranteed if the host were swapped.

## Reference solution

### 1. Place each capability on the generic extension model

| Capability | Generic extension point | Why this point (not the others) |
| --- | --- | --- |
| 1. Reviewer (read-only, tight prompt) | **Agent** | An isolated *role* with its own constrained prompt and a read-only permission profile. Not a tool (it reasons, it is not a single callable function); not a skill (it is a bounded actor, not know-how loaded into another actor). |
| 2. Security linter that must run on **every** artifact | **Hook / lifecycle** | A non-negotiable *guarantee* that fires on a lifecycle event with **no model discretion**. A hook is enforced by the runtime; the model cannot choose to skip it. |
| 3. Access to an internal service (deploy system / internal API) | **MCP (external access)** | *External access* across a trust boundary to a system the model does not own. MCP is the mechanism that puts the call — and its credential — outside the model's context. |
| 4. Org convention for a recurring procedure (cut a release / write a reply) | **Skill** | *Procedural know-how* the agent should follow when relevant — discretionary, model-applied, exactly what a skill is for. |
| 5. Local helper that transforms data the agent already has | **Tool** | A *callable pure function* with no external call and no side effect — a deterministic transform the model invokes by name. |

**Why capability 2 is NOT a skill.** A skill is *advisory* — it is loaded into the
model's context and the model decides whether and how to apply it. "Must run on
every single artifact, no exceptions, no model discretion" is the precise
negation of advisory. If you author the linter as a skill, a single distracted or
adversarially-prompted turn can skip it and the guarantee is gone. The guarantee
has to live in the **runtime lifecycle**, where the host (or an out-of-loop gate)
runs it regardless of what the model decides. Hook, not skill.

**Why capability 3 is NOT a skill.** Authoring "here is how to call the deploy
API, here is the token" as a skill puts the *integration logic and the credential*
into the model's context — the credential is now exfiltratable by any injected
instruction, and the call is the model's to make or fabricate. External access
belongs behind an **MCP server** that holds the credential in the integration
layer and exposes only a scoped tool surface. MCP, not skill. (The trust-boundary
and credential design for this is exercise-02; the interface choice is
exercise-03. Here we only note that capability 3's MCP server hands off to both.)

### 2. Map the architecture across named platforms (peers)

Rows are the five capabilities; columns are three peer hosts. Each cell names that
host's concrete mechanism.

| Capability | Generic point | LangGraph Platform (A) | Custom in-house host (B) | Claude Code (C, optional) |
| --- | --- | --- | --- | --- |
| 1. Reviewer | Agent | A sub-graph node with a read-only tool set and its own system prompt | A `reviewer` runtime role with a read-only credential profile | A subagent definition with `tools` restricted to read-only + a tight prompt |
| 2. Always-run linter | Hook / lifecycle | **No first-class per-artifact hook** — enforce in the graph edge + an out-of-loop CI gate | A lifecycle interceptor on the "artifact produced" event in your own runtime | A `PostToolUse` / lifecycle hook that runs the linter and can block |
| 3. Internal service | MCP | MCP client tool bound into the graph (credential in the MCP server) | MCP server fronting the internal API (credential in the server) | `.mcp.json` server entry; credential in the server, not the prompt |
| 4. Release/reply convention | Skill | A prompt/instruction module injected when the node runs | A skill document loaded from the bundle into the role's context | A skill in `skills/` loaded by relevance |
| 5. Local transform helper | Tool | A Python tool function registered on the graph | A registered callable in the host's tool registry | A tool definition (function the model calls by name) |

**The cell with no clean host equivalent — and the out-of-loop fallback.**
The **hook/lifecycle row for LangGraph Platform** has no first-class "run on every
artifact, block on failure" primitive the way a lifecycle-rich host does — graph
edges can call the linter, but a graph is model-and-author-controlled flow, so
"the model can never skip it" is not *guaranteed* by the runtime alone. The
fallback is to enforce the guarantee **out-of-loop**: a CI/policy gate on the
artifact's destination (the branch, the deploy queue, the publish endpoint) that
runs the same linter and **rejects the artifact if it did not pass**, independent
of whatever the agent did in-loop. The in-loop edge becomes a fast-feedback
convenience; the out-of-loop gate is the actual guarantee. This is the general
move for any host whose lifecycle cannot host the guarantee: push it to a fronting
service the artifact must pass through.

### 3. Portability per capability

| Capability | Portability (H/M/L) | What survives a host switch unchanged | What must be re-authored |
| --- | --- | --- | --- |
| 3. Internal service (MCP) | **High** | The MCP server, its tool schemas, and the credential placement — an open protocol any host can speak | Only the per-host client registration (one config line) |
| 5. Local transform helper (Tool) | **High** | The function body — pure logic, no host coupling | The host-specific tool registration wrapper |
| 1. Reviewer (Agent) | **Medium** | The role's intent, prompt, and read-only permission *policy* | The host's agent/subagent/node definition format and how read-only is expressed |
| 4. Convention (Skill) | **Medium** | The procedure prose itself | The host's skill/instruction-injection format and trigger model |
| 2. Always-run linter (Hook) | **Low** | The linter binary and the *intent* of "always run" | The entire enforcement mechanism — each host's lifecycle is proprietary, and one host (LangGraph) needs a wholly different out-of-loop construction |

The MCP capability grades **High** and the hook capability grades **Low**, exactly
as the chapter predicts: the MCP binding rides an open protocol, while the hook
binding rides each host's bespoke lifecycle and, on the weakest host, cannot be
expressed in-loop at all.

### 4. Lock-in register

| Capability | Host | Lock-in surface | Exit cost if we leave this host |
| --- | --- | --- | --- |
| 1. Reviewer | A / B / C | The host's agent/role definition format + how read-only is enforced | **Low–Medium**: re-express the role in the new host's format; the prompt and policy port |
| 2. Always-run linter | A / B / C | The host's lifecycle event model — *the guarantee itself is captured by the runtime* | **High**: re-author the enforcement per host; on a host without the lifecycle, build the out-of-loop gate from scratch |
| 3. Internal service | A / B / C | Almost nothing — the MCP server is yours and host-agnostic | **Low**: change one client registration line |
| 4. Convention | A / B / C | The host's skill format + injection/trigger semantics | **Low–Medium**: reformat the doc; re-tune when it loads |
| 5. Transform helper | A / B / C | The host's tool-registration wrapper | **Low**: re-wrap the same function |

**One-sentence verdict.** As designed, this platform is **cheap to re-host for
four of five capabilities and expensive to leave only on the always-run linter** —
which is the trade we intend, because we deliberately concentrated the lock-in in
the one capability whose value *is* a runtime-enforced guarantee, and we hedge
even that with an out-of-loop gate that any host can sit behind.

### 5. Defend a host decision

**Recommendation: a deliberate split, not a single host.**

- **Host-agnostic core:** capability 3 (internal-service MCP server) and
  capability 5 (transform tool) live as portable, self-owned artifacts that any of
  the three peer hosts binds with one line. We never let a vendor capture these.
- **Enforcement layer:** capability 2 (the always-run linter) is enforced
  **out-of-loop** in CI/policy as the source of truth on *every* host, with the
  in-loop hook as fast feedback where the host supports it (custom host, Claude
  Code) and as a graph edge where it does not (LangGraph). The guarantee never
  depends on a single vendor's lifecycle.
- **Role/convention layer:** capabilities 1 and 4 are authored once as
  prompt+policy and convention prose, then re-expressed per host — Medium
  portability we accept because the content ports even when the wrapper does not.

**Argued against the rejected alternative — "standardize on one rich-lifecycle
host (e.g., Claude Code) for everything."** That maximizes in-loop ergonomics for
the hook but **buys lock-in we do not need on the four portable capabilities** and
makes the always-run guarantee depend on one vendor's lifecycle continuing to
behave as documented. If that vendor changes its hook semantics or we are pushed
to a second host (a team already on LangGraph, a regulated workload that needs the
custom host), we re-author the guarantee under pressure. The split costs a little
more upfront (we build the out-of-loop gate even where the in-loop hook exists)
but keeps the guarantee vendor-independent and the portable capabilities free — so
the split wins.

### Stretch: fourth host build-cost and the packaged bundle

**Custom in-house host — build cost to reproduce each extension point yourself
versus inheriting it from a vendor:**

| Extension point | Inherit from a vendor host | Build yourself (custom host) |
| --- | --- | --- |
| Agent / role isolation | Cheap (definition format provided) | **Moderate** — you build prompt scoping + per-role permission profiles |
| Skill loading | Cheap (loader + trigger provided) | **Moderate** — you build relevance loading + a context budget |
| Tool registry | Cheap | **Low** — a function registry is small |
| Hook / lifecycle | Cheap *if the host has it* | **High** — a guaranteed, blocking, per-artifact lifecycle is the hardest thing to build correctly |
| MCP client | Cheap (client provided) | **Low** — speak an open protocol you do not own |

The lesson the table teaches: the **hook/lifecycle** is both the least portable
(task 3) and the most expensive to self-build — which is exactly why concentrating
deliberate lock-in there, and hedging it out-of-loop, is the right call.

**Packaged bundle layout** (the four-folder structure), with per-host portability:

```text
   platform-bundle/
     agents/      reviewer role            (Medium-portable: re-express per host)
     skills/      release-and-reply convs  (Medium-portable: reformat per host)
     tools/       local transform helper   (High-portable: re-wrap only)
     mcp/         internal-service server  (High-portable: one client line per host)
     hooks/       always-run linter        (LOW-portable: per-host lifecycle
                                            + an out-of-loop CI manifest)
     manifest/    per-host packaging:
       langgraph.toml   custom-host.json   claude-code.mcp.json
```

`tools/` and `mcp/` are host-portable as-is. `agents/` and `skills/` carry
portable content behind a per-host wrapper. `hooks/` is the only folder that needs
a genuinely different construction per host plus the out-of-loop manifest — so the
`manifest/` directory exists precisely to absorb the per-host packaging differences
the hook (and the agent/skill wrappers) require.

**One grounded MCP tool definition** (capability 3, host-neutral JSON Schema the
MCP server exposes — credential lives in the server, never here):

```json
{
  "name": "deploy.request_release",
  "description": "Open a release request in the internal deploy system. Read-and-propose only; does not promote to production.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "service": { "type": "string", "description": "Target service name." },
      "version": { "type": "string", "description": "Build/version identifier to stage." },
      "notes":   { "type": "string", "description": "Human-readable release notes." }
    },
    "required": ["service", "version"],
    "additionalProperties": false
  }
}
```

The same definition binds into LangGraph's MCP client, the custom host's MCP
client, or a `.mcp.json` entry — proving the capability-3 High portability grade
concretely: the schema is the contract, the host is just a client.

## Meeting the acceptance criteria

- **All five capabilities placed with a kind-based justification; both
  misplacement traps defended.** Task 1's table places each capability by *kind*,
  and the two prose paragraphs defend the always-run linter as a **hook (not a
  skill, because a skill is discretionary)** and the internal service as **MCP
  (not a skill, because a skill leaks the credential into context)**.
- **Mapping names two-plus hosts as peers, Claude Code at most one, and a
  no-clean-equivalent cell with an out-of-loop fallback.** Task 2 maps across
  LangGraph, a custom host, and Claude Code as peers (Claude Code only the third
  column); the LangGraph hook cell is flagged as having no first-class equivalent
  with a CI/policy out-of-loop fallback.
- **Portability graded per capability with MCP High and hook Low.** Task 3 grades
  all five; capability 3 (MCP) is High and capability 2 (hook) is Low, matching
  the chapter.
- **Lock-in register gives an exit cost per capability and a one-sentence
  verdict.** Task 4 gives an exit cost per row and concludes cheap-to-re-host for
  four, expensive-to-leave on the linter — by intent.
- **Host recommendation argued against a rejected alternative.** Task 5 recommends
  a deliberate split and argues it down against "standardize on one rich-lifecycle
  host," not by assertion.

## Common pitfalls

- **Authoring the always-run linter as a skill.** The single most common error:
  it reads like an instruction, so it looks like a skill. A skill is advisory; an
  always-run, no-discretion guarantee must be a runtime hook (or an out-of-loop
  gate), never something the model may skip.
- **Authoring internal-service access as a skill with the token in the prose.**
  Puts the credential in the model's context and makes the call fabricatable. It
  must be an MCP server holding the credential in the integration layer.
- **Treating one host as canonical and the others as "ports of it."** The
  placement is host-free; every host is a peer binding of the same generic point.
  A mapping that grades against "how close is host X to host Y" has missed the
  point — grade against the generic model.
- **Grading portability by familiarity instead of substrate openness.** MCP is
  High because the protocol is open, not because it is well-known; the hook is Low
  because each host's lifecycle is proprietary, not because it is hard to write.
- **Asserting the host decision.** A recommendation with no rejected alternative
  is unfalsifiable. Name the alternative (single rich-lifecycle host) and show
  what it costs.

## Verification

A completed submission is correct when:

- Every capability in task 1 is placed on **one** generic point with a *kind*
  justification, and capability 2 is a hook and capability 3 is MCP — each with an
  explicit "why not a skill" sentence.
- The mapping table names **two or more hosts as peers**, Claude Code appears at
  most once and is never described as the standard, and at least one cell is
  flagged as having no clean host equivalent with a named out-of-loop fallback.
- Portability is graded for all five capabilities; the MCP capability is **High**
  and the hook capability is **Low** (or any exception is defended on substrate,
  not preference).
- The lock-in register has an exit cost per capability and a one-sentence
  cheap-to-re-host vs. expensive-to-leave verdict.
- The host recommendation is argued **against a named rejected alternative**.
- `NOTES.md` answers the three prompts: the hardest capability to place (the
  internal service and the always-run linter are the two designed traps — either
  is a strong answer, with the *kind* test as the tiebreaker), what breaks if you
  force the always-run linter in-loop on the weakest-lifecycle host (a skipped or
  injected turn silently drops the guarantee, closed by the out-of-loop gate), and
  which capability hurts most to re-author on a forced host switch (the hook, per
  the Low portability grade — mitigated now by building the out-of-loop gate
  everywhere so the switch does not change the source of truth).
