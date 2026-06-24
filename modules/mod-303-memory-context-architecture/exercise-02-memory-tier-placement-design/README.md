# mod-303-memory-context-architecture/exercise-02 — Solution

## Approach

The skill assessed here is *placement and boundary reasoning*, so the deliverable
is a placement table and a defense, not code. This solution works the
**multi-tenant research assistant** scenario — many users, a shared public
knowledge base, private per-user notes — because it is the hardest of the three
provided scenarios on the scope boundary: it has both a system-scoped store and
per-user stores in the same architecture, which is exactly where cross-tenant
leaks happen.

The method is the one the chapter prescribes:

1. Enumerate **every** piece of state exhaustively — if you can't name it, you
   can't place it, and the unnamed item is the one that leaks.
2. Place each into a tier with a scope key, lifetime, store, and promotion rule.
3. Answer all five boundary checks with specifics that point at the schema.
4. Draw the promotion/distillation flow (the dynamic view).
5. Find a concrete leak and show the specific mechanism that prevents it.

The load-bearing design decision throughout: **the scope key lives in the
storage schema (the namespace/partition), not in the application's query.** A
filter in code is one missing `WHERE` clause from an incident; a partition the
store enforces cannot leak even when the query is wrong. Everything below is
built so that a buggy query degrades to "no results," never to "another user's
results."

## Reference solution

### 1. Enumerate the state

Exhaustive list of everything the multi-tenant research assistant reads or
writes. The exercise warns that an incomplete enumeration is where leaks hide, so
this deliberately includes the easy-to-forget items (IDs, the embedding cache,
the retrieval index namespaces).

```text
STATE ENUMERATION — multi-tenant research assistant

  1.  System prompt + tool definitions
  2.  Current research conversation (this turn / task)
  3.  Active research plan / scratchpad for the current question
  4.  Chunks retrieved from the shared KB this turn
  5.  Chunks retrieved from the user's private notes this turn
  6.  user_id of the authenticated caller
  7.  Per-user summary of a finished research session
  8.  Per-user stable preferences ("cite in APA", "exclude pre-2015 sources")
  9.  Per-user private uploaded documents (raw + chunked + embedded)
  10. Shared public knowledge base (curated corpus, raw + embedded)
  11. Vector index — shared KB namespace
  12. Vector index — per-user private namespace
  13. Embedding cache (text hash → vector)
  14. Session/auth token for the current request
  15. Cross-session episodic log (which questions this user asked, when)
```

### 2. Place each item into a tier

Every item gets a tier, a scope key, a lifetime, a store, and a promotion rule.
The scope-key column is the one a reviewer reads first.

```text
MEMORY-TIER PLACEMENT — multi-tenant research assistant

State item                 Tier        Scope key   Lifetime     Store                  Promotion rule
─────────────────────────  ──────────  ──────────  ───────────  ─────────────────────  ──────────────────────────
1  system prompt + tools   working     system      ephemeral    none (front of window) none
2  current conversation    working     turn        ephemeral    none (window)          none
3  active research plan     working     turn        ephemeral    none (window / notes) none — or note-take to (7)
4  retrieved KB chunks      working     turn        ephemeral    none (JIT)             none — drop after use
5  retrieved private chunks working     turn        ephemeral    none (JIT)             none — drop after use
6  user_id                  working     turn        ephemeral    request context        none (it IS the scope key)
7  finished-session summary episodic    per-user    weeks        per-user KV (ns=uid)   write at session end
8  user preferences         long-term   per-user    indefinite   per-user store (ns=uid) on signal, VERSIONED
9  private uploaded docs     long-term   per-user    indefinite   per-user blob (ns=uid) on upload
10 shared knowledge base     long-term   system      indefinite   shared store           curated pipeline; NO user data
11 vector index: KB ns       long-term   system      indefinite   vector store ns=shared rebuilt from (10)
12 vector index: user ns     long-term   per-user    indefinite   vector store ns=uid    rebuilt from (9); deletable
13 embedding cache           long-term   system*     indefinite   KV (key = text hash)   evict by LRU; see pitfall
14 auth/session token        working     turn        seconds      request context only   NEVER persisted
15 episodic question log     episodic    per-user    weeks        per-user log (ns=uid)  append per session
```

`ns=uid` denotes a store-level namespace keyed by `user_id` — the partition the
store enforces, not a filter the query supplies. `system*` on row 13 is flagged
because a naive shared embedding cache is a subtle leak vector; the pitfall
section resolves it.

### 3. Defend the boundaries

The five boundary checks, answered with specifics that point at the schema.

```text
□ PROMOTION — is every persisted item justified?
  Persisted: (7) session summaries, (8) preferences, (9) private docs,
  (10) shared KB, (12)(15) per-user indexes/logs. Each is justified:
  summaries enable cross-session continuity; preferences shape every answer;
  docs and the shared KB are the corpus; the log powers "what did I ask before".
  DELIBERATELY NOT PERSISTED: (2) raw conversation, (3) active plan, (4)(5)
  retrieved chunks. These die with the window on purpose — persisting raw
  turns is the promote-everything anti-pattern that builds retrievable noise
  the agent later rots on. Only the distilled summary (7) crosses, not the
  transcript.

□ SCOPE — is every per-user item partitioned by a key in the SCHEMA?
  Yes. Rows 7, 8, 9, 12, 15 all carry ns=uid. The partition is the store's
  namespace, so a read is store.get(ns=uid, key) — there is no code path that
  reads per-user data without supplying the namespace, because the namespace
  is a required positional argument of the store API (see §5 mechanism).

□ PRIVACY — does the system-scoped store contain ZERO user-private data?
  Yes. The system stores are (10) the curated KB and (11) its vector index,
  both fed ONLY by a curation pipeline that never runs inside a user session.
  The look-alike trap: (5) a private chunk and (4) a KB chunk are both
  "retrieved chunks" and structurally identical — but (5) lives in ns=uid and
  (4) in ns=shared. They are on opposite sides of the boundary by namespace,
  not by content inspection. Resolved embedding cache (13) in pitfalls.

□ CONSISTENCY — does each tier meet its freshness/durability need?
  working (2–6,14): no durability, must be fresh — retrieved chunks are JIT
    re-fetched, never cached stale.
  episodic (7,15): per-session durable + read-your-writes within a user thread
    — a summary written this session is visible next session. Cross-region
    eventual consistency is acceptable; intra-thread staleness is not.
  long-term (8,9,10,12): durable; preferences (8) are VERSIONED upserts so
    "cite in APA" replacing "cite in MLA" is a new version, not a silent
    overwrite — the agent never reasons over a contradiction.

□ FORGETTING — trace "delete user U".
  Reaches: drop per-user KV namespace ns=U (7,8,15) + per-user blob ns=U (9)
  + DELETE the per-user vector namespace ns=U (12) + purge any embedding-cache
  entries derived from U's private docs (13). System stores (10,11) are
  untouched — they provably contain no U data (privacy check above). Because
  every per-user row shares the ns=U partition key, this is one fan-out over a
  known key set, not an archaeology project.
```

### 4. The promotion / distillation flow

The dynamic view the static table can't show — what crosses each boundary and on
what trigger.

```text
PROMOTION / DISTILLATION FLOW — multi-tenant research assistant

  WORKING[turn]
    │  trigger: session end / explicit checkpoint
    │  what crosses: a distilled session summary (decisions, sources used,
    │                open threads) — NOT the raw transcript
    ▼
  EPISODIC[user]  (ns=uid: session summaries + question log)
    │  trigger: a preference observed stably (stated, or inferred N≥3 times)
    │  what crosses: the durable fact only, as a VERSIONED upsert
    ▼
  LONG-TERM[user]  (ns=uid: preferences, private docs + index)

  SEPARATE INGEST PATH (never from a session):
    curation pipeline ──▶ LONG-TERM[system]  (shared KB + shared vector index)

  SCOPE INVARIANT: nothing flows WORKING[turn] → LONG-TERM[system]. A user
  session can write only into ns=uid. The system tier has no session-facing
  write path at all.
```

That last invariant is the architecture's spine: there is physically no edge
from a user session into the system tier, so a coding bug cannot route private
data there — the write API into the shared store is callable only by the
offline curation job, not by the request handler.

### 5. Find the leak

```text
PLAUSIBLE LEAK
  Mechanism of failure: the agent answers user A's question by retrieving from
  the private-notes vector index. If retrieval filtered by user_id in
  APPLICATION CODE — search(query, filter={"user_id": A}) — then a refactor
  that drops or mistypes the filter returns the global nearest neighbours,
  surfacing user B's private notes in A's session. One missing filter = a
  cross-tenant data breach.

MECHANISM THAT PREVENTS IT
  The private index is partitioned at the STORE level by namespace. The
  retrieval call is user_index(ns=A).search(query) — the namespace is a
  required argument that selects the physical partition; there is no global
  index spanning users to fall back to. A bug that forgets to pass ns=A fails
  closed (no namespace → error / empty), it does not fall through to B's data.
  The scope key is in the schema, not the query. (See the §Stretch prototype
  below for an executable proof.)
```

### Stretch — executable scope-boundary proof

A tiny per-user store where retrieval **cannot** return another user's data even
when called with a wrong query, plus a deletion test.

```python
# scope_store.py — the scope key is structural, not a query filter
class PerUserStore:
    def __init__(self) -> None:
        self._ns: dict[str, dict[str, str]] = {}  # user_id -> {key: value}

    def put(self, user_id: str, key: str, value: str) -> None:
        self._ns.setdefault(user_id, {})[key] = value

    def search(self, user_id: str, query: str) -> list[str]:
        # Reads are confined to the caller's namespace by construction.
        # There is no global index to leak from.
        partition = self._ns.get(user_id, {})
        return [v for v in partition.values() if query in v]

    def forget(self, user_id: str) -> None:
        self._ns.pop(user_id, None)  # one op reaches the whole per-user partition


def test_no_cross_tenant_read() -> None:
    s = PerUserStore()
    s.put("alice", "note1", "alice secret plan")
    s.put("bob", "note1", "bob secret plan")
    # Even with a query that matches Bob's data, Alice's namespace cannot see it.
    assert s.search("alice", "secret plan") == ["alice secret plan"]
    assert s.search("bob", "secret plan") == ["bob secret plan"]


def test_forget_one_user_survives_the_other() -> None:
    s = PerUserStore()
    s.put("alice", "n", "alice data")
    s.put("bob", "n", "bob data")
    s.forget("alice")
    assert s.search("alice", "data") == []         # gone from every partition
    assert s.search("bob", "data") == ["bob data"]  # untouched
```

### Reflection answers

1. **Hardest item to place:** the embedding cache (13). It feels system-scoped
   (vectors are just math), but a cache keyed only by text hash leaks: if user A
   uploads a private document, user B uploading the *same* document gets a cache
   hit that confirms A possessed it (a membership-inference leak). Resolved by
   scoping cache entries derived from private docs to `ns=uid`, and only sharing
   cache entries for the curated KB. Tier-ambiguous because the *data* is shared
   but the *fact of possession* is private.
2. **Tempted to promote everything:** the raw conversation (2) and the question
   log detail. Persisting every turn "so the agent remembers" would build a
   per-user noise pile the agent retrieves and rots on (Chapter 2) and enlarge
   the forgetting surface. Cost avoided: retrieval rot plus a bigger GDPR blast
   radius. Only the distilled summary crosses.
3. **First consistency boundary to bite multi-region:** episodic
   read-your-writes (7, 15). A user finishing a session in region X and
   immediately reconnecting to region Y must see the just-written summary;
   cross-region replication lag breaks that. Fix: pin a user's episodic writes
   and reads to a home region (sticky routing), or use a session-durable store
   with synchronous replication for the per-user thread, accepting eventual
   consistency only for the system KB.

## Meeting the acceptance criteria

- **Placement table covers every state item** — all 15 enumerated items appear
  in the table with tier, scope key, lifetime, store, and promotion rule
  (Tasks 1–2).
- **All five boundary checks answered with specifics** — each check points at a
  schema artifact (`ns=uid`, the curation-only write path, versioned upserts),
  not an assertion (Task 3).
- **Promotion/distillation flow with named triggers** — session-end, stable
  preference (N≥3), and the separate curation ingest path, with the scope
  invariant stated (Task 4).
- **Plausible cross-user leak + preventing mechanism** — the dropped-filter leak
  and the namespace-as-schema defense, proven executable in the stretch
  (Task 5).
- **Forgetting path reaches every per-user partition** — "delete user U" fans out
  over `ns=U` across all per-user KV, blob, vector, and cache entries; system
  stores provably untouched (Task 3 forgetting check + stretch test).

## Common pitfalls

- **Scope key in the query, not the schema.** `search(query, filter={"user_id":
  A})` is one refactor from a breach. Make the namespace a required argument of
  the store so a forgotten scope fails closed (empty), never open (another
  user's data).
- **Promote-everything.** Writing raw turns to a vector store "for memory" builds
  retrievable noise and a larger forgetting surface. Promote only distilled
  outcomes.
- **A shared embedding cache keyed only by content.** It leaks possession of
  private documents via cache hits. Scope cache entries derived from per-user
  data to the user's namespace.
- **Forgetting that misses the index.** Deleting a user's rows but leaving their
  vectors in the retrieval index means deleted content is still retrievable — a
  privacy failure. The forgetting path must reach every per-user index
  namespace, not just the source store.
- **A system store that look-alike data sneaks into.** A private chunk and a KB
  chunk are structurally identical; only the namespace separates them. Never let
  a session write to the system tier — give it no write path at all.

## Verification

- Run `test_no_cross_tenant_read` and `test_forget_one_user_survives_the_other`;
  both must pass — the first proves a wrong query cannot cross tenants, the
  second proves forgetting is surgical.
- Walk the placement table and confirm every per-user row carries `ns=uid` and
  every system row has no session-facing write path.
- Trace "delete user U" by hand against the table and confirm it touches all
  per-user stores (KV, blob, vector, cache) and no system store.
- For each long-term per-user fact, confirm the update model is a versioned
  upsert, not an overwrite — re-set a preference and check the prior version is
  still auditable.
- Sanity-check completeness: if any boundary check is hard to answer for an item,
  that item is probably mis-enumerated or mis-placed — revisit Task 1.
