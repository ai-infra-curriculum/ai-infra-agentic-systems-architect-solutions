# mod-310-agentic-developer-platforms/exercise-03-context-hydration-strategies — Solution

## Approach

The discipline this exercise enforces is **pull, don't pre-load; distill, don't
dump.** A maximal context is not a safe default — it is slower, costlier, and
*less* accurate, because the relevant signal drowns in noise (context rot). The
job is to assign every candidate piece of context to a layer, name how each
layer is delivered, budget the tokens, decide what to *exclude*, and design what
the agent does when context is missing.

I pick the **"debug a failing CI test"** task because it has a clean four-layer
shape and a sharp baseline contrast: the naive "load the whole repo" approach is
catastrophic here, which makes the budget argument concrete.

The four layers and their delivery mechanisms:

- **L0 standing** — conventions/guardrails, delivered by a plugin **skill**.
  Small, stable, always present.
- **L1 task** — the specific failure, delivered by the **CI event payload** that
  triggered the run.
- **L2 retrieved-on-demand** — the bulk, pulled **lazily** through **MCP tools**
  (`search_code`, `git_log`). This is where most context lives, and the whole
  point is that it is fetched, not pre-loaded.
- **L3 isolated/distilled** — a big sub-task run in a **subagent** that returns a
  summary, keeping the main window clean.

Then I budget the layers against a load-everything baseline with real numbers,
list the exclusions and why each *improves* accuracy, and write degradation
rules for two realistic missing-context cases.

## Reference solution

### 1. Candidate-context inventory (unfiltered)

Everything the agent *could* be handed for "debug a failing CI test":

- The goal: "make this test pass without disabling it."
- Debug conventions and guardrails (don't delete the assertion, don't `xfail`).
- The failing test's name and its assertion error.
- The CI log tail (the traceback, the last lines before failure).
- The test file itself.
- The source module(s) under test.
- The last few commits that touched the test or the source.
- The full repo (all packages, all modules).
- All open Jira tickets.
- The full CI history for the branch.
- The design doc the feature came from.
- The full git history.
- The dependency lockfile and CI config.

Inventory first, filter second — the next step is where most of this gets
excluded or pushed lazy.

### 2. Layer assignment with delivery mechanism

| Item | Layer | Delivery mechanism |
| ---- | ----- | ------------------ |
| Debug conventions / "don't disable tests" guardrail | **L0** | plugin **skill** (`debug-conventions`) |
| Failing test name + assertion error | **L1** | **CI event payload** |
| CI log tail (traceback) | **L1** | **CI event payload** |
| The test file | **L2** | **MCP** `read_file` (pulled on demand) |
| Source module(s) under test | **L2** | **MCP** `search_code` → `read_file` |
| Last 3 commits touching test/source | **L2** | **MCP** `git_log` (on demand) |
| CI config / lockfile (only if dependency-related) | **L2** | **MCP** `read_file` (conditional pull) |
| Cross-module call-path trace (only if failure spans modules) | **L3** | **subagent** → returns 5-line root cause |

**Why the bulk lives in L2, not L0/L1.** The agent does not know *which* three
files matter until it has read the failing assertion. Pre-loading the test file
and source into L0/L1 would mean guessing — and for most failures, guessing
wrong, because the failing assertion points at code the always-present layers
couldn't have known to include. Lazy L2 retrieval lets the agent read the log
*first*, then fetch exactly the files the traceback implicates. The model only
ever sees what it explicitly fetched.

### 3. Context budget for one run

Order-of-magnitude estimates (tokens), this plan vs. the naive baseline:

| Layer | Content | Budget (this plan) |
| ----- | ------- | ------------------ |
| L0 | debug skill (conventions) | ~0.5k (standing) |
| L1 | test name + assertion + log tail | ~1k (per task) |
| L2 | test file + 1–2 source files + 3 commit diffs | ~6k (bounded, lazy) |
| L3 | subagent root-cause summary (only if needed) | ~0.3k (distilled) |
| | **Total (typical run)** | **~8k** |

Naive "load the whole repo + all tickets" baseline:

| Source | Budget (baseline) |
| ------ | ----------------- |
| Whole repo (even a small 200-file service) | ~400k–2M+ |
| All open tickets | ~50k+ |
| Full CI history | ~30k+ |
| **Total** | **~500k–2M+** |

**The gap: ~8k vs. ~500k+ — roughly two orders of magnitude.** The baseline
either overflows the window outright or, where it fits, buries the failing
assertion's three relevant files under hundreds of irrelevant ones. The lazy
plan is also *faster* (no megabyte of retrieval and prefill per run) and
*cheaper* (you pay for ~8k input tokens, not 500k+).

### 4. What to exclude, and why exclusion improves accuracy

| Excluded | Why excluding it makes the agent *more* accurate |
| -------- | ------------------------------------------------ |
| The rest of the repo | The failing test implicates a small neighborhood; the other 99% of files are pure noise that dilutes attention and invites the model to "fix" unrelated code. |
| Unrelated open tickets | They describe other work; in-context they tempt scope creep and false associations between this failure and unrelated features. |
| Full git history | Only the commits that touched the test/source are causally relevant; the rest is distraction with a non-trivial token cost. |
| Full CI history | The *current* failure's log tail is the signal; prior runs add volume without pointing at the bug. |
| The design doc | Useful for *building* the feature, irrelevant to *why this assertion fails now*; including it shifts the agent toward redesign instead of debugging. |

Exclusion is not deprivation — it is **noise reduction**. Less irrelevant
context means the relevant context dominates the model's attention, which is
exactly what raises accuracy.

### 5. Safe degradation for missing context

**Case A — the L1 source is thin (a one-line ticket / no acceptance criteria,
applied to a sibling "implement a small ticket" framing of the same plan):**

```text
If get_ticket() returns no acceptance criteria:
  → do NOT invent requirements
  → state explicitly: "ticket PROJ-1234 is underspecified: no acceptance
     criteria, no expected behavior given"
  → ask ONE clarifying question, or escalate to the requesting human
  → do not open a PR against guessed requirements
```

**Case B — an L2 MCP fetch errors (Jira down, repo unreachable, `search_code`
returns an error):**

```text
If an MCP fetch returns an error (not an empty result):
  → surface it as a tool failure the agent reports:
     "could not retrieve source under test: search_code errored (repo
      unreachable). Cannot debug without the source."
  → treat error != "nothing relevant exists"
  → stop and report; do NOT proceed as if the file were empty or absent
```

The shared rule across both: **make sparseness legible, prefer asking/surfacing
over hallucinating.** A missing fetch is never silently coerced into "no
results"; an underspecified task is never silently filled with invented
requirements.

### 6. The plan

| Layer | Content | Source / mechanism | Budget |
| ----- | ------- | ------------------ | ------ |
| L0 | debug conventions; "don't disable tests to make them pass" | plugin skill | ~0.5k, standing |
| L1 | failing test name, assertion error, CI log tail | CI event payload | ~1k, per task |
| L2 | test file + source under test + last 3 commits that touched them | MCP (`search_code`, `read_file`, `git_log`), on demand | ~6k, bounded/lazy |
| L3 | cross-module failure → subagent traces call path, returns 5-line root cause | subagent | ~0.3k, distilled |

**Degradation rules:** underspecified L1 → state what's missing, ask one
question or escalate, never invent. L2 fetch error → surface as a tool failure
and stop, never treat an error as an empty result.

### Stretch: caching, a budget guardrail, and permission-scoped L2

**Caching.** L2 reads that are stable across runs — the CI config, the
lockfile, a rarely-changing utility module — can be cached keyed on the file's
content hash (or the commit SHA). **Invalidation trigger:** a new commit on the
branch (or a hash mismatch) busts the cache for the touched files. The failing
test file and the source under test are *not* good cache candidates mid-debug,
because the whole point of the run is that they're changing.

**Context-budget guardrail.** A check that warns at ~12k and blocks at ~20k of
hydrated context for a single run. It lives **tool-side** (in the MCP layer that
assembles retrieval), not in a `PreToolUse` hook, because the hook sees one tool
call at a time and cannot sum the running total across an L2 fetch sequence; the
retrieval layer can track cumulative tokens and refuse the fetch that would blow
the budget, emitting a legible "hydration budget exceeded — narrow the query."

**Permission-scoped L2 (reconciling with exercise-02).** Every L2 `search_code`
/ `read_file` runs against the **requesting human's** permissions in the MCP
server, so lazy hydration cannot become a data-leak path — the agent can only
pull files the requester could read. Just-in-time retrieval and least-privilege
retrieval are the same discipline applied to tokens and to access.

## Meeting the acceptance criteria

- **Every item assigned to a layer with a named mechanism; bulk in L2.** Section
  2's table places each inventory item in L0–L3 with a concrete delivery
  mechanism, and the test/source/commits — the bulk — sit in L2, pulled on
  demand, with an explicit justification for laziness.
- **Budget shows a clear token + latency advantage with numbers.** Section 3
  contrasts ~8k (this plan) against ~500k–2M+ (baseline) — roughly two orders of
  magnitude — and notes the latency and cost wins.
- **Exclusion list explains how leaving context out improves accuracy.** Section
  4 gives five exclusions, each tied to noise reduction (attention dilution,
  scope creep, redesign drift), not just token savings.
- **Degradation makes missing context legible, prefers asking/surfacing.**
  Section 5 covers both required cases — underspecified L1 and an erroring L2
  fetch — with rules that ask or surface and never hallucinate.

## Common pitfalls

- **Pre-loading "to be safe."** Putting the test file and source into L1 because
  "the agent will need them." It doesn't know *which* files until it reads the
  assertion; pre-loading guesses wrong and pays the token cost regardless. Keep
  them L2 and let the traceback drive the fetch.
- **Treating a fetch error as an empty result.** A `search_code` that errors is
  not "no relevant code exists." Silently proceeding produces a confident,
  wrong debug. Surface the error and stop.
- **Dumping the repo and trusting the model to filter.** "More context can't
  hurt." It does: relevant signal drowns, attention dilutes, and the model
  starts editing unrelated files. Less is more accurate, not just cheaper.
- **L3 distillation that drops the needed detail.** A subagent that returns "the
  bug is in the auth module" without the file:line the main agent needed has
  lost information. Specify what the summary *must* carry (root-cause location +
  the failing path), and have the main agent re-pull if the summary is too thin.
- **Caching volatile files.** Caching the file under active debug means the
  agent reasons over a stale version. Cache only stable reads (config,
  lockfile), invalidate on commit/hash change.

## Verification

- **Layer-coverage check.** Confirm every inventory item in Section 1 appears in
  exactly one row of the Section 2 table with a named mechanism — nothing is
  unassigned, nothing is double-counted.
- **Budget sanity.** Re-estimate L0–L3 for a real failing test in a sample repo
  (count tokens with the provider's tokenizer on the actual files) and confirm
  the total lands in the single-digit-thousands range, two orders below a
  whole-repo load.
- **Degradation dry-run.** Feed the agent a one-line ticket and confirm it
  states the underspecification and asks rather than inventing; simulate an MCP
  `search_code` error and confirm it surfaces the failure rather than proceeding
  on an empty result.
- **Exclusion audit.** For each excluded source, articulate the specific failure
  it would have caused in-context (scope creep, redesign drift, window
  overflow) — if you can't, reconsider whether it belongs in L2.
- **Guardrail test (stretch).** Drive an L2 query that would exceed the budget
  threshold and confirm the tool-side guardrail warns/blocks with a legible
  message.
- **Markdownlint** clean: one H1, every fence tagged (`text`), blank lines around
  blocks, dash bullets, trailing newline.
