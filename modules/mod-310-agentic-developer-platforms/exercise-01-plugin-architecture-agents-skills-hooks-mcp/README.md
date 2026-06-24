# mod-310-agentic-developer-platforms/exercise-01-plugin-architecture-agents-skills-hooks-mcp — Solution

## Approach

The exercise asks for a single architectural move repeated six times: take a
platform requirement and route it onto the Claude Code extension point whose
semantics match it, using one rule — **hooks for guarantees, MCP for access,
skills for know-how, subagents for isolated specialized work.** The trap is
treating the four extension points as interchangeable. They are not. They split
on *who triggers them and whether it is deterministic*:

- Agents and skills are **model-invoked** — the model decides when, based on a
  description. Good for judgment and know-how; useless as a guarantee.
- Hooks are **deterministic lifecycle events** — they fire every time the event
  fires, by configuration, not by the model's choice. This is the only place a
  guarantee can live.
- MCP is **external access** — invoked as tools, but implemented and governed
  outside the agent, where the credential is held.

The deliverable is the *partition* plus the real config artifacts that make it
concrete. I work it in three passes:

1. **Partition** each requirement, calling out the two that decompose into a
   "describe" skill plus an "enforce" hook (commit style + commit hook; PR
   format skill is paired with the security-review subagent gate).
2. **Author the five artifacts** the exercise names — `plugin.json`, the
   `security-reviewer` subagent, the `pr-conventions` skill, `hooks/hooks.json`,
   and the `.mcp.json` Atlassian entry — as complete, well-formed files, not
   sketches.
3. **Defend the invariants**: read-only review tools, deterministic protected-path
   block, and zero credentials in agent-visible context.

All config shapes are verified against the current Claude Code docs at
[code.claude.com/docs](https://code.claude.com/docs). The protected-path hook in
particular uses the current `PreToolUse` JSON-output contract
(`hookSpecificOutput.permissionDecision: "deny"`), with exit code 2 noted as the
simpler scripted alternative.

## Reference solution

### 1. The partition table

| # | Requirement | Extension point(s) | Justification |
| - | ----------- | ------------------ | ------------- |
| 1 | Open PRs in Acme's PR format | **Skill** (`pr-conventions`) + **MCP** (`github.create_pr`) | The *format* is know-how the agent applies when opening a PR (skill); the *act of opening* the PR is access to an external system (MCP). Describe vs. do. |
| 2 | Never write to `infra/`, `*.tf`, `db/migrations/` without human confirmation | **Hook** (`PreToolUse`) | A guarantee that must hold on *every* edit, deterministically — never the model's discretion. Guarantees are hooks. |
| 3 | Run a focused security review on any diff before a PR is finalized | **Agent** (`security-reviewer` subagent), optionally backstopped by a **Hook/CI gate** | Specialized, isolated work in a clean context with a read-only persona (subagent). If the review must be a *hard* block, add a CI gate — the subagent is judgment, the gate is the guarantee. |
| 4 | Read/transition Jira tickets; read Confluence pages | **MCP** (`atlassian` server) | Access to external systems. The credential lives in the server, never in context. |
| 5 | Run `make lint` after every file edit and surface failures | **Hook** (`PostToolUse`) | Must run on *every* edit, deterministically — a guarantee, not a suggestion the model may skip. |
| 6 | Apply Acme's house style for commit messages | **Skill** (`commit-style`) + **Hook** (`commit-msg`-style `PreToolUse` guard) | The *style* is know-how (skill the agent loads when committing); enforcing the format on every commit is a guarantee (hook). The classic describe-plus-enforce split. |

Two requirements correctly split across two extension points: **#1** (skill
describes the PR format, MCP opens the PR) and **#6** (skill describes commit
style, hook enforces it). Requirement #3 is a third candidate — a subagent for
judgment with a CI gate when a hard block is required.

Why the splits are not gold-plating: a skill that *describes* the PR format
gives you nothing if the model decides not to load it; an MCP tool that opens a
PR gives you the action but not the convention. You need both, in their
respective layers. Same for commit style — the skill teaches the format, the
hook guarantees it.

### 2. Plugin manifest

`.claude-plugin/plugin.json`:

```json
{
  "name": "acme-sdlc-platform",
  "version": "1.0.0",
  "description": "Acme's internal agentic SDLC platform: PR-convention and commit-style skills, a read-only security-review subagent, protected-path and lint guards, and Jira/Confluence MCP access.",
  "author": {
    "name": "Developer Platform Team",
    "email": "platform@acme.example"
  },
  "homepage": "https://github.com/acme/sdlc-platform-plugin",
  "license": "Apache-2.0"
}
```

`name` is the install identifier (kebab-case, unique in the marketplace);
`version` is semantic; `author` is an object with `name` and optional `email`.
The directory layout the manifest sits at the root of:

```text
acme-sdlc-platform/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json        # stretch goal, below
├── agents/
│   └── security-reviewer.md
├── skills/
│   ├── pr-conventions/
│   │   └── SKILL.md
│   └── commit-style/
│       └── SKILL.md
├── hooks/
│   └── hooks.json
├── scripts/
│   └── guard-protected-paths.sh
└── .mcp.json
```

### 3. The security-reviewer subagent

`agents/security-reviewer.md` — frontmatter scopes tools to **read-only** so the
reviewer inspects but cannot write:

```markdown
---
name: security-reviewer
description: >-
  Reviews a proposed code diff for security issues — injection, hardcoded
  secrets, unsafe deserialization, SSRF, authz gaps, and path traversal.
  Invoke before finalizing any PR, on the diff only. Returns a structured
  verdict. Read-only: it inspects, it never edits.
tools: Read, Grep, Glob
model: inherit
---

You are a focused security reviewer for Acme. You are given a proposed code
diff and nothing else. You do not have write tools and must not request them.

Procedure:

1. Read the diff. Pull in adjacent file context with Read/Grep only when a
   finding requires it to confirm or rule out.
2. Evaluate against, at minimum: injection (SQL, command, template), hardcoded
   secrets or credentials, unsafe deserialization, SSRF and unvalidated
   outbound requests, missing authorization checks, and path traversal.
3. Report each finding on one line as:
   `SEVERITY (critical|high|medium|low) — file:line — issue — fix`.
4. If you find nothing, say so explicitly; do not invent findings to look busy.

End with exactly one verdict line:

- `VERDICT: BLOCK` if any critical or high finding is present.
- `VERDICT: PASS` otherwise (note medium/low findings as advisory).

Do not modify files. Do not open or finalize the PR. You produce a verdict;
the platform and the human decide what to do with it.
```

`tools` is a comma-separated string and is restricted to the three read-only
tools — there is no `Edit`, `Write`, or `Bash`, so the subagent *structurally
cannot* change the tree or run a shell. `model: inherit` keeps it on the
session's model.

### 4. The pr-conventions skill

`skills/pr-conventions/SKILL.md` — the `description` says *when* to load it so
the model pulls it in at the right moment, not always:

````markdown
---
name: pr-conventions
description: >-
  Acme's required pull-request format — title prefix, description template, and
  linked-ticket rule. Load this whenever opening, editing, or reviewing a PR
  body, or when asked to "open a PR" / "follow our PR format".
---

# Acme Pull-Request Conventions

Apply this format to every PR you open or edit.

## Title

`<type>(<scope>): <summary>  [PROJ-1234]`

- `type` is one of: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`,
  `ci`.
- `scope` is the affected module (e.g. `billing`, `auth`), lowercase.
- `summary` is imperative, lowercase, no trailing period, <= 70 chars.
- The trailing `[PROJ-1234]` is the **required** Jira key. If you do not have a
  ticket key, stop and ask — do not open a PR without one.

## Description template

Fill every section. Do not delete headings; write "N/A" if truly empty.

```text
## What
<one paragraph: what this change does and why>

## Linked ticket
Closes PROJ-1234

## How to test
<steps a reviewer runs to verify>

## Risk / rollback
<blast radius and how to revert>
```

## Linked-ticket rule

- Every PR links exactly one Jira ticket via `Closes PROJ-NNNN` in the
  description and the `[PROJ-NNNN]` suffix in the title.
- If the change spans multiple tickets, split it into multiple PRs.

## What this skill does NOT cover

- It does not *open* the PR — that is the `github.create_pr` MCP tool.
- It does not enforce commit-message style — that is the `commit-style` skill
  plus the commit guard hook.
````

The skill is pure know-how. It deliberately disclaims the action (MCP) and the
enforcement (hook) so the boundaries between extension points stay crisp.

### 5. The hooks

`hooks/hooks.json` — a `PreToolUse` guard that blocks protected paths
deterministically, and a `PostToolUse` hook that runs `make lint`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PLUGIN_ROOT/scripts/guard-protected-paths.sh"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "make lint"
          }
        ]
      }
    ]
  }
}
```

The block is deterministic because it is a hook on the `PreToolUse` event with a
matcher on the write tools — it fires on *every* matching tool call, before the
edit lands, regardless of what the model "decides." `$CLAUDE_PLUGIN_ROOT`
resolves to the installed plugin's root directory, so the script path is stable
wherever the plugin is installed.

`scripts/guard-protected-paths.sh` — reads the tool input on stdin, denies the
call when the target matches a protected path, using the current `PreToolUse`
JSON-output contract:

```bash
#!/usr/bin/env bash
# Block edits to protected paths. Fires on every Edit|Write|MultiEdit via the
# PreToolUse hook. The model cannot bypass this; it is configuration, not a
# suggestion.
set -euo pipefail

# The hook receives the tool call as JSON on stdin.
input="$(cat)"

# Extract the target path (file_path for Edit/Write; first path for MultiEdit).
target="$(printf '%s' "$input" | jq -r '
  .tool_input.file_path
  // .tool_input.path
  // (.tool_input.edits[0]?.file_path)
  // empty
')"

# Protected path patterns from requirement 2.
protected_regex='(^|/)(infra/|db/migrations/)|\.tf$'

if [[ -n "$target" && "$target" =~ $protected_regex ]]; then
  # Current Claude Code PreToolUse contract: deny with a reason the agent sees.
  jq -n --arg p "$target" '{
    hookSpecificOutput: {
      hookEventName: "PreToolUse",
      permissionDecision: "deny",
      permissionDecisionReason: ("Protected path: \($p). Edits to infra/, *.tf, and db/migrations/ require explicit human confirmation. Ask the requesting engineer to apply this change directly or approve an exception.")
    }
  }'
  exit 0
fi

# Not protected — allow by emitting nothing and exiting 0.
exit 0
```

`exit 0` plus the `permissionDecision: "deny"` JSON is the explicit, current
contract and lets us return a reason the agent can relay. The simpler
alternative — `exit 2` with the reason written to stderr — also blocks the call,
but the JSON form is preferred here because it carries a structured,
agent-readable reason.

### 6. The Atlassian MCP server

`.mcp.json` — Jira + Confluence access for requirement 4, with the credential
held by the server:

```json
{
  "mcpServers": {
    "atlassian": {
      "command": "npx",
      "args": ["-y", "@acme/atlassian-mcp-server"],
      "env": {
        "ATLASSIAN_BASE_URL": "https://acme.atlassian.net",
        "ATLASSIAN_EMAIL": "${ACME_ATLASSIAN_EMAIL}",
        "ATLASSIAN_TOKEN": "${ACME_ATLASSIAN_TOKEN}"
      }
    }
  }
}
```

The token is interpolated from the host environment (`${ACME_ATLASSIAN_TOKEN}`)
into the **server's** environment, where the server process holds it. The model
calls tools like `jira.get_issue` and `confluence.get_page`; it never sees the
token, because the secret is in the integration layer, not the context window.
The Jira token should be scoped to the projects the bot needs and to the
transitions it is allowed to perform — least privilege at the credential, not
just at the prompt.

### Stretch: marketplace entry and rollout

`.claude-plugin/marketplace.json` makes the plugin installable org-wide:

```json
{
  "name": "acme-platform-marketplace",
  "owner": {
    "name": "Developer Platform Team",
    "email": "platform@acme.example"
  },
  "plugins": [
    {
      "name": "acme-sdlc-platform",
      "source": "./",
      "description": "Acme's internal agentic SDLC platform plugin."
    }
  ]
}
```

Rollout/rollback story: engineers add the marketplace
(`/plugin marketplace add acme/sdlc-platform-plugin`) and install
`acme-sdlc-platform`. Because the plugin is a versioned Git repo, a rollout is a
tagged release; a rollback is pinning to the previous tag. A bad hook or a
broken skill is reverted centrally in one PR, not chased across 50 laptops.

### Stretch: a second subagent

`agents/migration-writer.md` deserves its own isolated context because schema
migrations are high-blast-radius, low-frequency work with a distinct persona:
it should reason only about the schema diff and the migration's
forward/backward safety, not carry the main agent's whole feature context. Its
toolset is tightly scoped — `Read`, `Grep`, `Glob`, and a single
`Write` limited to `db/migrations/` (paired with a `PreToolUse` hook that
*allows* that path for this subagent while the global guard still gates it for
the main agent). Isolation keeps a migration mistake from leaking into, or being
contaminated by, unrelated feature reasoning.

## Meeting the acceptance criteria

- **Every requirement mapped with a one-sentence justification, ≥1 split.** The
  partition table covers all six; #1 (skill + MCP) and #6 (skill + hook) are
  explicit two-point splits, with #3 noted as subagent-plus-optional-gate.
- **Partition obeys the routing rule.** Guarantees (#2 protected paths, #5 lint,
  #6 enforcement) are hooks, not skills; access (#1 open-PR, #4 Atlassian) is
  MCP, not skills; the protected-path block is a `PreToolUse` hook that fires on
  every write tool call — deterministic, not model-discretionary.
- **All five artifacts present and well-formed.** `plugin.json`,
  `agents/security-reviewer.md`, `skills/pr-conventions/SKILL.md`,
  `hooks/hooks.json` (with the real guard script), and `.mcp.json` are complete
  and valid against the current docs.
- **Read-only reviewer, no leaked credentials.** The subagent's `tools` line is
  `Read, Grep, Glob` — no write or shell tool. The only place a credential
  appears is the Atlassian server's `env`, interpolated from the host
  environment; it never enters any agent-visible context.

## Common pitfalls

- **Putting a guarantee in a skill.** "Add a skill that tells the agent not to
  edit `infra/`." A skill is model-invoked know-how; the model can decline to
  load it or override it under pressure. Protected-path enforcement *must* be a
  `PreToolUse` hook, or it is not a guarantee.
- **Putting access in a skill.** Writing a `jira-howto` skill and assuming the
  agent can now read Jira. The skill describes; it cannot *call* anything. Jira
  access is an MCP server. Describe-vs-do is the recurring split.
- **Giving the security reviewer write tools.** Adding `Edit` or `Bash` "so it
  can fix what it finds" collapses the isolation. A reviewer that can write is
  no longer a reviewer; scope it to `Read, Grep, Glob` and let a separate,
  gated step apply fixes.
- **Leaking the token into context.** Pasting the Atlassian token into a skill,
  a prompt, or an `args` value instead of the server's `env`. Anything in
  context can be exfiltrated by a prompt-injection in a ticket; the secret must
  live only in the server process.
- **Using the stale hook-block contract.** Emitting `{"decision":"block"}` from
  the guard script. The current `PreToolUse` contract is
  `hookSpecificOutput.permissionDecision: "deny"` (or exit code 2); verify the
  exact field names against the hooks docs before relying on the block.

## Verification

- **Lint the configs.** `jq . .claude-plugin/plugin.json`,
  `jq . hooks/hooks.json`, `jq . .mcp.json`, and
  `jq . .claude-plugin/marketplace.json` all parse without error.
- **Validate the plugin loads.** Install the plugin locally
  (`/plugin marketplace add <path>` then `/plugin install acme-sdlc-platform`)
  and confirm Claude Code lists the `security-reviewer` agent, the
  `pr-conventions` and `commit-style` skills, and the hooks with no parse
  warnings.
- **Exercise the protected-path guard both ways.** Pipe a synthetic tool call
  into the script and assert the decision:

  ```bash
  echo '{"tool_input":{"file_path":"infra/main.tf"}}' \
    | scripts/guard-protected-paths.sh | jq -r '.hookSpecificOutput.permissionDecision'
  # -> deny

  echo '{"tool_input":{"file_path":"src/app.ts"}}' \
    | scripts/guard-protected-paths.sh
  # -> (no output, exit 0): allowed
  ```

- **Confirm the read-only invariant.** Grep the subagent frontmatter and assert
  no write tools: `grep -E '^tools:' agents/security-reviewer.md` shows only
  `Read, Grep, Glob`.
- **Confirm no secret in context-visible files.** `grep -rn 'ATLASSIAN_TOKEN'
  .` finds it only as `${ACME_ATLASSIAN_TOKEN}` inside `.mcp.json`'s `env`, never
  as a literal value and never in a skill or agent file.
- **Markdownlint** this solution: one H1, fenced blocks tagged and blank-line
  separated, dash bullets, trailing newline.
