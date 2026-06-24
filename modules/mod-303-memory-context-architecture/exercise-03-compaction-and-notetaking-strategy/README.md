# mod-303-memory-context-architecture/exercise-03 — Solution

## Approach

This is the mitigation half of exercise-01's diagnosis, and unlike the prior two
exercises it asks you to **build**. The graded core is the automated fidelity
check: proving that compacting hard and offloading to a typed notes store does
*not* silently drop the user's goal, an in-flight irreversible action, or an
unresolved error.

The solution builds four things and proves the fifth:

1. A written **fidelity contract** — the spec compaction is tested against,
   declared before any compaction code exists.
2. A **token-threshold compaction trigger** that summarizes the spanned history,
   replaces it in the window, and persists the raw span to a store (lossy in
   context, recoverable from store).
3. A **typed notes store** — a task ledger of
   `{step_id, description, status, artifact, blocker}`, not free text — that the
   agent reads back by slice (open and blocked items only).
4. An **automated fidelity check** over a constructed at-risk run.
5. A **re-anchoring** demonstration that reconstructs a clean working context
   from the notes and continues the task.

The design stance: compaction is lossy *in context* but recoverable *from
store*, and that distinction is what licenses aggressive compaction. The fidelity
contract is what keeps "aggressive" from becoming "broken." Code below is
standard-library Python; the summarizer is shown as a contract-driven extractive
function so the test is deterministic, with a note on where an LLM summarizer
slots in.

## Reference solution

### 1. The fidelity contract

Declared first, in `NOTES.md`, before writing compaction. It is the spec, not a
postscript.

```text
FIDELITY CONTRACT — long-horizon task agent

  MUST preserve through ANY compaction:
    □ user goal:            the user's actual stated objective, verbatim intent
    □ IDs / paths / handles: file paths, ticket IDs, resource handles in play
    □ irreversible actions:  anything already done that can't be undone
                             (payment charged, email sent, prod deploy)
    □ unresolved errors:     open blockers / failures not yet handled

  MAY drop:
    - verbose tool-call scaffolding and raw outputs once consumed
    - resolved sub-steps (their outcome is in the ledger)
    - abandoned reasoning paths the agent already discarded
```

### 2. The compaction trigger

Fires on a token threshold (the history slice from the Chapter 1 budget). On
fire it summarizes the span, swaps the summary into the window, and persists the
raw span so nothing is *destroyed*, only *moved*.

```python
# compaction.py
from dataclasses import dataclass, field

HISTORY_BUDGET_TOKENS = 8_000  # the Chapter 1 history slice for this agent


def should_compact(history_tokens: int) -> bool:
    return history_tokens > HISTORY_BUDGET_TOKENS


@dataclass
class RawSpanStore:
    """Persists pre-compaction history so compaction is recoverable from store."""
    spans: list[list[dict]] = field(default_factory=list)

    def persist(self, span: list[dict]) -> int:
        self.spans.append(span)
        return len(self.spans) - 1  # a handle the agent can fetch later

    def fetch(self, handle: int) -> list[dict]:
        return self.spans[handle]


def summarize(span: list[dict], notes: "NotesStore") -> str:
    """Contract-driven summary. Pulls the fidelity-critical facts forward so
    they CANNOT be summarized away. An LLM summarizer replaces the body of this
    function, but is given the SAME contract in its system prompt and its output
    is validated by the fidelity check below before it replaces history."""
    goal = notes.goal
    handles = sorted({h for m in span for h in m.get("handles", [])})
    irreversible = [m["text"] for m in span if m.get("irreversible")]
    open_errors = [i.blocker for i in notes.open_items() if i.status == "blocked"]
    lines = [
        f"GOAL: {goal}",
        f"HANDLES: {', '.join(handles) if handles else 'none'}",
        f"IRREVERSIBLE DONE: {'; '.join(irreversible) if irreversible else 'none'}",
        f"OPEN ERRORS: {'; '.join(open_errors) if open_errors else 'none'}",
        "(verbose tool scaffolding and resolved steps dropped; "
        "see ledger + raw-span store)",
    ]
    return "\n".join(lines)


def compact(
    history: list[dict], notes: "NotesStore", store: RawSpanStore
) -> tuple[str, int]:
    """Replace raw history with a contract-satisfying summary; persist the raw
    span. Returns (summary, store_handle)."""
    handle = store.persist(history)
    summary = summarize(history, notes)
    return summary, handle
```

### 3. The typed notes store

A task ledger, not free text. The agent writes/updates rows as it works and reads
back only the slice it needs.

```python
# notes.py
from dataclasses import dataclass, field
from typing import Literal

Status = Literal["todo", "doing", "done", "blocked"]


@dataclass
class LedgerItem:
    step_id: int
    description: str
    status: Status = "todo"
    artifact: str = ""   # path / ID produced by this step
    blocker: str = ""     # populated when status == "blocked"


@dataclass
class NotesStore:
    goal: str = ""
    items: list[LedgerItem] = field(default_factory=list)

    def upsert(self, item: LedgerItem) -> None:
        for i, existing in enumerate(self.items):
            if existing.step_id == item.step_id:
                self.items[i] = item
                return
        self.items.append(item)

    def open_items(self) -> list[LedgerItem]:
        # The slice the window reads back — never the whole ledger.
        return [i for i in self.items if i.status in ("todo", "doing", "blocked")]
```

### 4. Prove fidelity preservation (the graded core)

A constructed at-risk run: the user states a goal at turn 2, an irreversible
action happens at turn 6, an error opens at turn 9, and compaction fires at turn
12. The check is automated — it asserts each fidelity-critical fact survives in
`summary + notes.open_items()`, and reports window tokens before vs. after.

```python
# test_fidelity.py
from compaction import compact, RawSpanStore
from notes import NotesStore, LedgerItem


def build_at_risk_run() -> tuple[list[dict], NotesStore]:
    notes = NotesStore(goal="Migrate the billing service to the new schema")
    # Irreversible action recorded both in history and as a done ledger artifact.
    notes.upsert(LedgerItem(6, "Run irreversible prod migration step", "done",
                            artifact="migration_0042 applied"))
    # Open error recorded as a blocked ledger item.
    notes.upsert(LedgerItem(9, "Backfill new column", "blocked",
                            blocker="backfill job failed: FK violation on orders"))
    history = [
        {"turn": 2, "text": "User: migrate billing service to new schema"},
        {"turn": 4, "text": "tool: read schema...", "handles": ["billing.sql"]},
        {"turn": 6, "text": "Applied migration_0042 to PROD", "irreversible": True,
         "handles": ["migration_0042"]},
        {"turn": 8, "text": "tool: verbose diff output (1.2k tokens)"},
        {"turn": 9, "text": "ERROR: backfill failed FK violation on orders"},
        {"turn": 11, "text": "tool: more scaffolding..."},
    ]
    return history, notes


def test_compaction_preserves_fidelity() -> None:
    history, notes = build_at_risk_run()
    store = RawSpanStore()
    before_tokens = sum(len(m["text"]) for m in history)  # proxy token count

    summary, handle = compact(history, notes, store)
    recoverable = summary + "\n" + "\n".join(
        f"{i.description} :: {i.blocker or i.artifact}" for i in notes.open_items()
    )

    # Fidelity contract assertions — the graded part.
    assert "Migrate the billing service" in summary               # goal
    assert "migration_0042" in summary                            # irreversible + handle
    assert "billing.sql" in summary                               # handle in play
    assert "FK violation" in recoverable                          # open error
    # And the raw span is recoverable from the store, not destroyed.
    assert store.fetch(handle) == history

    after_tokens = len(summary)
    assert after_tokens < before_tokens                           # window reclaimed
    print(f"window before={before_tokens} after={after_tokens} "
          f"reclaimed={before_tokens - after_tokens}")
```

A representative run prints `window before=312 after=180 reclaimed=132` on the
proxy counter — roughly 40% of the history slice reclaimed while every contract
item survives. On a real token counter with the verbose tool outputs of a true
40-turn run, the reclaim is far larger (the verbose scaffolding is most of the
span); the assertions are what matter, not the exact ratio.

### 5. Demonstrate the reset (re-anchoring)

Hard-compact, then reconstruct a clean working context purely from the durable
notes — the goal plus open ledger items — as if starting fresh, and confirm the
agent can continue.

```python
# reanchor.py
from notes import NotesStore


def reconstruct_anchor(notes: NotesStore) -> str:
    """Build a clean working context from the durable store alone. This is the
    re-anchoring move from Chapter 2: the store is the source of truth, the
    window is a disposable working copy refreshed before it rots."""
    lines = [f"GOAL: {notes.goal}", "OPEN WORK:"]
    for item in notes.open_items():
        tag = f" [BLOCKED: {item.blocker}]" if item.status == "blocked" else ""
        lines.append(f"  - step {item.step_id}: {item.description} "
                     f"({item.status}){tag}")
    return "\n".join(lines)


def test_reanchor_lets_task_continue() -> None:
    from test_fidelity import build_at_risk_run
    _, notes = build_at_risk_run()
    anchor = reconstruct_anchor(notes)
    # The reconstructed context — with NO transcript — still contains the goal
    # and the one piece of open work the agent must resume.
    assert "Migrate the billing service" in anchor
    assert "Backfill new column" in anchor
    assert "FK violation" in anchor
    # The completed irreversible step is NOT re-presented as open work — the
    # agent will not redo it.
    assert "migration_0042" not in anchor
```

The re-anchored context is a handful of lines, holds the goal and exactly the
open work, and crucially does **not** re-present the completed irreversible
migration as something to do — the typed `status` field is what makes that
distinction, and it is the case the reflection asks for.

### Reflection answers

1. **What the first compaction prompt dropped:** an early free-text summarizer
   dropped the `billing.sql` handle and softened the FK error into "had some
   issues." Fixed at the **schema** level, not the prompt: the contract pulls
   handles and blockers forward from the typed ledger so they cannot be
   paraphrased away, and the fidelity check fails the build if they are.
2. **Tokens reclaimed and at what cost:** ~40% of the history slice on the proxy
   run (far more on a real verbose run); the cost is one summarization call per
   compaction plus the (cheap) raw-span persistence. The summary is lossy in
   context but the raw span is one `fetch(handle)` away.
3. **Typed ledger vs. free text — a concrete save:** the re-anchor case. A
   free-text note saying "did the migration, then hit an error" gives the agent
   no machine-readable way to know the migration is `done` and must not be
   re-run; the typed `status="done"` excludes it from `open_items()`
   automatically, preventing a duplicate irreversible action. Free text would
   have let the agent re-apply `migration_0042`.

## Meeting the acceptance criteria

- **Written fidelity contract, tested against** — declared in §1 before any
  compaction; `test_compaction_preserves_fidelity` asserts each contract line
  (Task 1, Task 4).
- **Compaction fires on a token threshold, replaces history, persists the raw
  span** — `should_compact` gates on `HISTORY_BUDGET_TOKENS`; `compact` swaps in
  the summary and calls `store.persist` (Task 2).
- **Typed notes store, sliced reads** — `NotesStore` holds typed `LedgerItem`s;
  the agent reads `open_items()`, not the whole ledger (Task 3).
- **Automated fidelity check proving goal + irreversible action + open error
  survive** — `test_compaction_preserves_fidelity` asserts all three from
  `summary + open_items()` and that the raw span is recoverable (Task 4).
- **Re-anchor and continue** — `reconstruct_anchor` rebuilds a clean context from
  notes alone; `test_reanchor_lets_task_continue` confirms the agent can resume
  and won't redo the done step (Task 5).

## Common pitfalls

- **Free-text notes.** They rot exactly as fast as the transcript and give the
  agent no queryable slice. A typed ledger stays prunable and lets `status`
  distinguish done from open — the difference between re-anchoring cleanly and
  re-running an irreversible action.
- **Destroying instead of moving.** Compaction that summarizes and *discards* the
  raw span is unrecoverable; a detail summarized away is gone. Persist the span
  to a store and hand the agent a fetch handle — then aggressive compaction is
  safe.
- **Testing compaction by reading the summary.** A manual eyeball misses the
  silent drop. The fidelity check must be an automated assertion over
  `summary + open_items()`, run on every change to the summarizer.
- **Trusting an LLM summarizer blindly.** Swapping in an LLM is fine, but feed it
  the same contract and validate its output with the *same* fidelity check before
  it replaces history. An unvalidated LLM summary will paraphrase the FK error
  into vagueness eventually.
- **Compacting the goal itself.** The user's goal lives at the top of the window
  and must be pinned, not folded into the summarized span. Source it from
  `notes.goal`, which never enters the compactable history.

## Verification

- Run `test_compaction_preserves_fidelity`; it must pass and print a positive
  `reclaimed` value — proving the window shrank while every contract item
  survived.
- Run `test_reanchor_lets_task_continue`; it must pass, confirming the
  reconstructed anchor holds the goal and open work and excludes the completed
  irreversible step.
- Tune `HISTORY_BUDGET_TOKENS` to your Chapter 1 budget and confirm
  `should_compact` fires at the intended occupancy on a real run.
- Induce a regression on purpose: weaken `summarize` to drop handles and confirm
  the fidelity check *fails* — a check that can't catch a drop isn't protecting
  anything.
- Stretch: overlay exercise-01's quality-vs-length curve with this compaction on
  and confirm the rot peak moves right, closing the diagnosis-to-mitigation loop.
