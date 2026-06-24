# mod-308-deployment-durable-execution/exercise-01-durable-execution-design — Solution

## Approach

The exercise is a decomposition, not an implementation: take the vendor-onboarding
agent and split it cleanly into a **durable workflow** (deterministic control flow)
and **ephemeral activities** (every non-deterministic or side-effecting step), then
prove that the run survives a process death and that no irreversible action can fire
twice. Four artifacts carry the weight: the durable/ephemeral diagram, the
decomposition table, the replay/determinism rule (with one wait shown as a durable
signal), and the idempotency register.

The design choices that drive everything below:

- **The workflow holds only orchestration and recorded results.** Steps 1 through 5
  are sequenced by the workflow, but the workflow never *does* anything observable —
  it dispatches activities and branches on their recorded results. This is the single
  rule that keeps replay valid (Chapter 1: "the agent loop's control flow is the
  deterministic Workflow; every LLM call, tool call, and retrieval is an Activity").
- **Every external touch is an activity**, because every external touch is
  non-deterministic: an API read returns different bytes on replay, an LLM call is
  non-deterministic by construction, and `now()` advances. Recording the *result* of
  each is what lets replay fast-forward without re-hitting the network.
- **Waits are durable signals/timers, not in-process loops.** "Wait until the vendor
  uploads documents" and "wait for compliance sign-off" are the two multi-day waits;
  both are expressed as the workflow sleeping on a signal in the durable plane, so the
  worker is freed and the process may die without losing the wait.
- **Idempotency keys are derived from stable run/step identity**, never from
  wall-clock time or a fresh UUID — because the whole point is that a *re-run* of the
  same step computes the *same* key and dedupes against the recorded result.

The hardest classification call is step 3 (create ERP + billing records): it is the
clearest "irreversible write" but it is also the place a naive implementation hides
non-determinism — see the trap called out in the decomposition table.

## Reference solution

### 1. The durable/ephemeral split

```text
┌──────────────────────────────────────────────────────────────────────────┐
│  DURABLE PLANE — VendorOnboardingWorkflow (deterministic; replayable)      │
│                                                                            │
│   orchestrate steps 1->5 as recorded control flow:                         │
│     - dispatch A1 request_documents                                        │
│     - await SIGNAL documents_uploaded   (durable wait — days)              │
│     - dispatch A2 sanctions_check, A3 credit_check (recorded results)      │
│     - branch on RECORDED A2/A3 results only                                │
│     - dispatch A4 create_erp_record, A5 create_billing_record             │
│     - await SIGNAL compliance_signoff   (durable wait — gate, ex-02)       │
│     - dispatch A6 send_welcome_email                                       │
│     - mark onboarding_complete (recorded)                                   │
│                                                                            │
│   state persisted to durable store: step history, A2..A6 results,          │
│   pending timers, signal/approval state                                    │
└───────────────────────────────────┬────────────────────────────────────────┘
                                     │  dispatch step / replay from history
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  EPHEMERAL PLANE — Activities (stateless workers; may run >once)           │
│                                                                            │
│   A1 request_documents(vendor)     side effect: open portal task           │
│   A2 sanctions_check(vendor)        external API read  (non-deterministic)  │
│   A3 credit_check(vendor)           external API read  (non-deterministic)  │
│   A4 create_erp_record(vendor)      IRREVERSIBLE write  * idempotency key   │
│   A5 create_billing_record(vendor)  IRREVERSIBLE write  * idempotency key   │
│   A6 send_welcome_email(vendor)     IRREVERSIBLE send   * idempotency key   │
│                                                                            │
│   workers hold nothing that must survive a crash                           │
└──────────────────────────────────────────────────────────────────────────┘
```

The control flow (the sequencing of 1->5, the two waits, the branch on screening
results) lives entirely in the durable plane. Every LLM/API/side-effect lives in an
activity whose result is recorded. A worker can die at any instant; the engine
reschedules the unfinished activity onto a fresh worker and the workflow resumes from
the last recorded step.

### 2. Decomposition table

| Step | Workflow or Activity? | Deterministic? | Why |
| --- | --- | --- | --- |
| Orchestrate steps 1->5 (sequencing, branching) | Workflow | Yes (must be) | Pure control flow over recorded results; replay must reproduce the same step order. |
| 1. Request documents (open portal task) | Activity (A1) | No | A side effect on an external portal; the dispatch is recorded, the effect is not replayed. |
| 1. Wait for upload | Workflow wait on `documents_uploaded` signal | Yes | The *waiting* is durable state; the arrival event is recorded, not polled. |
| 2. Sanctions check | Activity (A2) | No | External API read — different bytes on replay; record the result once. |
| 2. Credit check | Activity (A3) | No | External API read; same reasoning as A2. |
| Branch on screening results | Workflow | Yes | Branches on the *recorded* A2/A3 results, which are deterministic on replay. |
| 3. Create ERP record | Activity (A4) | No | Irreversible external write; recorded `record_id` is what replay returns. |
| 3. Create billing record | Activity (A5) | No | Irreversible external write; same as A4. |
| 4. Wait for compliance sign-off | Workflow wait on `compliance_signoff` signal | Yes | Durable HITL interrupt (designed in exercise-02); the decision is a recorded signal. |
| 5. Send welcome email | Activity (A6) | No | Irreversible external send; idempotency key prevents a double send. |
| 5. Mark onboarding complete | Workflow | Yes | A recorded state transition; deterministic. |

**The non-determinism-in-the-workflow trap.** The seductive mistake is to compute a
risk tier *inside the workflow* by calling the model directly, or to stamp the ERP
record with `created_at = now()` inside the workflow before dispatching A4. Both read
non-recorded values (the model output, the wall clock) in deterministic code. On
replay the model returns something different and `now()` advances, so the workflow
takes a different branch or builds a different payload than history recorded — the
engine raises a non-determinism error and the run is poisoned. The fix: the model
call is an activity whose result is recorded; the timestamp is captured *inside* the
activity (or via a durable timer), never read in workflow code.

### 3. The replay/determinism boundary

Stated as a rule the implementing engineer can hold in their head:

> **Workflow code may not read the wall clock, generate randomness, perform I/O, or
> branch on any value it did not receive from a recorded activity result or signal.**
> Everything it needs that violates this comes from the durable plane instead:
>
> - current time -> a **durable timer**, not `now()`;
> - randomness / a UUID -> generated **inside an activity** and recorded, or seeded
>   from `(run_id, step_id)`;
> - any network read or write -> an **activity** whose result is recorded;
> - a wait for an external event -> a **signal/timer** the workflow sleeps on, never a
>   polling loop in process memory.

**One wait, expressed durably.** "Wait until the vendor uploads documents" is *not* a
loop:

```text
   WRONG (in-memory poll — dies with the process):
     while not portal.documents_ready(vendor):
         sleep(60)                     # holds a worker for days; lost on crash

   RIGHT (durable signal — survives process death):
     workflow:
       dispatch A1 request_documents(vendor)        # opens the portal task
       await signal "documents_uploaded"            # YIELD worker; persist wait
       # ... process may be killed / deployed / drained here; the wait is durable
       # portal webhook -> engine delivers signal -> workflow resumes on ANY worker
       proceed to A2/A3
```

The workflow records "waiting on `documents_uploaded`" in the durable store and yields
its worker. The process is now free to die. When the portal fires its webhook, the
engine delivers the signal and resumes the workflow on whatever worker is available —
no compute was held during the multi-day wait.

### 4. Idempotency register

Every irreversible side effect, each with a stable key and dedup strategy:

| Side effect | Idempotency key | Dedup strategy | What "ran twice" must equal |
| --- | --- | --- | --- |
| A4 create ERP record | `hash(run_id, "A4_create_erp", vendor_id)` | ERP API accepts the key as a client-request-id and returns the same `record_id` on retry; if the ERP lacks that, record `key -> record_id` locally and return the recorded id on re-run. | Exactly one ERP record; the same `record_id` returned both times. |
| A5 create billing record | `hash(run_id, "A5_create_billing", vendor_id)` | Billing API idempotency-key header (most billing APIs support one); else local record-and-replay keyed by the key. | Exactly one billing record; same billing id both times. |
| A6 send welcome email | `hash(run_id, "A6_welcome_email", vendor_id)` | Email provider idempotency key (or a local "sent" marker keyed by the key, checked before send). | Exactly one email delivered; the second attempt is a no-op returning the recorded send id. |
| A1 open portal task (side effect) | `hash(run_id, "A1_portal_task", vendor_id)` | Portal upsert keyed by the key — re-running re-attaches to the existing task rather than creating a duplicate. | One portal task, not two. |

The shape every key obeys, so a re-run computes the *same* key and dedupes:

```text
   key = stable_hash(run_id, step_id, payload_digest)   # NEVER now(), NEVER fresh UUID
   on activity call:
     if store.has(key):  return store.get(key)     # replayed — return recorded result
     result = do_side_effect()
     store.put(key, result)                        # record BEFORE returning
     return result
```

A2/A3 (the screening reads) do not strictly need idempotency keys because they are
reads with no external mutation — re-running them is harmless and the *recorded
result* is what replay uses. They are still recorded so replay does not re-hit the
vendor APIs unnecessarily.

### 5. Substrate justification

Against the spectrum from Chapter 1:

| Substrate | Verdict for this scenario |
| --- | --- |
| Roll-your-own DB checkpoints | **Insufficient.** You would hand-write resume logic, durable timers for multi-day waits, signal delivery for the two human/vendor waits, and version-safe replay — i.e. re-implement a workflow engine, badly, on a regulated workload. |
| Durable functions (DBOS / Inngest / Restate) | **Borderline.** They persist step results and are lighter, but the multi-day signal-driven waits, the worker-versioning needs for in-flight-safe deploys (exercise-03), and the audit-grade event history push past their comfortable range. |
| **Workflow engine (Temporal / Cadence)** | **The choice.** Full event-history replay, first-class durable timers and signals (for the document-upload and compliance-sign-off waits), and worker versioning (the foundation for exercise-03's rainbow deploy). The strongest guarantees, which a compliance workload with irreversible writes needs. |
| Managed step orchestrator (Step Functions) | **Possible but awkward.** Expressing a days-long, signal-driven, human-gated agent as a provider state machine is more contortion than the Temporal model, and the audit-replay story is weaker. |

The decision is driven by Chapter 1's three questions, all "yes" here: runs live for
**days**, take **irreversible side effects** (account creation, the email), and **wait
on external signals** (document upload, human sign-off). The further down the table you
belong, and this scenario belongs at the workflow-engine row. The cheaper options are
insufficient precisely because they do not give you durable signals + replay + worker
versioning together, and this workload needs all three.

### Crash-survivability sequence (where a double effect would occur without a key)

```text
   workflow        activity (A4: create_erp_record)        ERP API
   --------        --------------------------------        -------
      |  dispatch A4  |                                        |
      |--------------▶|  key = hash(run_id,"A4",vendor)        |
      |               |  store.has(key)? NO                    |
      |               |  create record -----------------------▶|  (record created)
      |               |  ◀---------------- 200 + record_id ----|
      |               |  ✗ worker dies BEFORE store.put + report
      |  (engine sees no completion -> reschedules A4 on new worker)
      |  dispatch A4  |                                        |
      |--------------▶|  key = hash(run_id,"A4",vendor)  <- SAME KEY
      |               |  ERP dedupes on key (or store.has after the
      |               |  put committed) ----------------------▶|  returns SAME record_id
      |               |  ◀---------------- same record_id -----|
      |  ◀-- one ERP record; replay returns the same id --     |
```

The load-bearing detail: the key is derived from `(run_id, step_id, vendor_id)`, so the
*rescheduled* A4 computes the identical key and the ERP (or the local store) returns the
already-created `record_id` instead of creating a second record. If the worker dies
after the ERP write but before `store.put`, the dedup must live **server-side at the
ERP** (idempotency-key header) — a local-only store cannot help if it never committed.
That is why the register prefers ERP/billing/email APIs that accept the key natively,
with local record-and-replay only as the fallback.

### Stretch: compensation, versioning, and write volume

- **Saga / compensation.** If A3 (credit check) fails *after* A4 created the ERP
  record, the workflow runs a compensating activity `A4c void_erp_record`, keyed by the
  same `(run_id, "A4")` identity so it targets exactly the record A4 created.
  Compensations fire in reverse order of the writes they undo (void billing before
  voiding ERP if both ran), and each is itself idempotent. The workflow owns the saga
  control flow; the undo effects are activities.
- **Workflow-version change (add a fourth check).** A run already past the screening
  branch must *not* suddenly grow a fourth check on replay — that is a non-determinism
  divergence. Old runs pin to the version they started on (finish on three checks);
  new runs start on the version with four. If the fourth check *must* reach in-flight
  runs, guard it with a version marker (`GetVersion`/patch) so old runs replay the
  3-check path and new runs take the 4-check path. This is exactly the
  pin-vs-patch decision designed in [exercise-03](../exercise-03-agent-fleet-deployment-strategy/README.md).
- **Write volume.** Roughly one durable event per dispatch plus one per completion per
  activity, plus signal and timer events: on the order of ~20–30 events per run (6
  activities, 2 signals, a handful of timers/markers). At thousands of runs/day that is
  well within a workflow engine's budget, and the auditability payoff — a tamper-evident,
  replayable record of every screening result and account creation — is exactly what a
  *compliance* workload needs, so the write volume is justified here.

## Meeting the acceptance criteria

- **Every step placed in durable or ephemeral plane with justification; all
  non-determinism in activities** — the split diagram and decomposition table place
  every step, and every API read, model call, side effect, and `now()` is an activity
  (or a durable timer), never workflow code.
- **Decomposition table complete; one naive-non-determinism trap identified** — the
  table covers all steps and waits, and the "risk tier computed inside the workflow /
  `now()` in the payload" trap is called out with the replay-corruption it causes.
- **Replay rule stated; one wait expressed as a durable signal/timer** — the four-part
  rule is written, and the document-upload wait is shown as a signal the workflow
  sleeps on, contrasted against the wrong in-memory poll.
- **Idempotency register covers every irreversible side effect with a stable key and
  dedup strategy** — A4, A5, A6 (and the A1 portal side effect) each have a
  `(run_id, step, vendor)` key and a server-side-or-record-and-replay dedup strategy,
  with "ran twice = ran once" stated per row.
- **Substrate argued against alternatives** — the four substrates are each weighed and
  the cheaper three rejected on concrete grounds (no durable signals + replay + worker
  versioning together), not asserted.

## Common pitfalls

- **Calling the model or reading `now()` inside the workflow.** The most common
  determinism break: any non-recorded value read in workflow code diverges on replay.
  Push it into an activity and record the result; get time from a durable timer.
- **Polling for the document upload / sign-off in process memory.** A `while not
  ready: sleep()` loop holds a worker for days and dies on the first deploy. The wait
  must be a durable signal the workflow yields on.
- **Idempotency keys built from `uuid4()` or `now()`.** A fresh key every call defeats
  dedup — the re-run computes a *different* key and the side effect fires again. Keys
  must be a pure function of stable run/step identity.
- **Recording the result *after* returning (or only locally).** If the worker dies
  between the side effect and the local `store.put`, a local-only dedup never sees the
  first run. Prefer server-side idempotency at the downstream API for the truly
  irreversible writes; record-and-replay is the fallback, not the primary.
- **Forcing durable execution onto the cheap reads.** A2/A3 are reversible reads; they
  do not need idempotency keys, only result recording. Over-keying everything is noise
  that hides the four effects that actually matter.

## Verification

A completed submission is correct when:

- The control flow (sequencing 1->5, the branch on screening, both waits) is in the
  durable plane and *nothing else* is — scan the workflow box for any API call, model
  call, or `now()`; there should be none.
- The decomposition table marks every API read, side effect, and model call as a
  non-deterministic activity, and every wait as a workflow signal/timer.
- At least one naive-non-determinism trap is written out with the replay error it
  would cause (the risk-tier-in-workflow / `now()`-in-payload trap qualifies).
- The idempotency register lists **A4, A5, A6** at minimum, each with a key that is a
  function of `(run_id, step, vendor_id)` and a stated dedup strategy; trace one key
  through the crash sequence and confirm the re-run returns the recorded id.
- The substrate section rejects roll-your-own and durable-functions on the
  multi-day-signal + versioning grounds, not on preference.
- `NOTES.md` answers the three prompts: the hardest workflow-vs-activity call (the ERP
  write — irreversible *and* a non-determinism magnet), the crash trace showing the
  key dedupes the rescheduled A4, and what a future `time.time()` inside the workflow
  would do (a non-determinism error on the next replay).
