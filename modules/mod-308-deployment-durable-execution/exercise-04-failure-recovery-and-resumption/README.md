# mod-308-deployment-durable-execution/exercise-04-failure-recovery-and-resumption — Solution

## Approach

The deliverable proves a single property across three failure scopes: **no in-flight
run is lost**. That holds only because progress is durable, so recovery everywhere is
"reschedule the unfinished step," not "restart the run" (Chapter 4). The artifact is
four pieces: the corrected scaling signal (with its three sizing numbers), the
worker/zone/dependency recovery matrix, the two incident fixes (the AZ outage and the
poison run), and the per-run blast-radius caps.

The choices that drive the solution:

- **Autoscale on ready-step backlog, not run count.** With 2% of runs sleeping for days
  on approvals, run count is dominated by runs consuming ~zero compute; it over-provisions
  on sleepers and misses the bursts. Ready-step backlog tracks real compute demand, and
  it is *bounded by downstream rate limits* because scaling past what the LLM/tool APIs
  can absorb just converts "slow" into "rate-limited errors."
- **Recovery is rescheduling, enabled by four conditions.** A multi-AZ HA durable store,
  workers spread across zones with no run affinity, idempotent steps (exercise-01), and
  dependency failures that *pause* the run durably rather than crash it. Name all four
  or the "reschedule the step" guarantee silently does not hold.
- **The AZ incident was caused by state in the workers.** The three fixes turn an AZ loss
  from a fleet-wide outage into surviving zones picking up orphaned runs.
- **Isolate misbehaving runs.** A per-run failure budget quarantines poison runs before
  they crash workers fleet-wide; per-run resource caps bound a runaway fan-out to one
  run's blast radius.

## Reference solution

### 1. Fix the scaling signal

**Why run count is wrong.** The fleet is 50,000 runs/day; ~2% (1,000+ at any time) are
sleeping for days on human approvals, consuming *zero* worker compute — they are rows in
the durable store. Autoscaling on concurrent run count counts those sleepers as load and
over-provisions, while *missing* the real spikes (a run's bursty parallel tool fan-out).
Run count and compute demand are decoupled by the durable/ephemeral split, so scaling on
run count scales on the wrong number.

**The correct signal** is **ready-step backlog** (the queue depth of steps ready to
execute) or equivalently **task schedule-to-start latency**, bounded by downstream rate
limits:

```text
   run_count            ──▶ misleads (sleeping approvals inflate it)
   ready_steps_backlog  ──▶ tracks real compute demand   ◀── autoscale on THIS
   dependency_rate_lim  ──▶ caps useful worker count     ◀── bound by THIS

   autoscale_target ≈ min( ready_steps/sec demand,
                           dependency_rate_limit / steps_per_call )
```

Scaling past `dependency_rate_limit / steps_per_call` adds workers that only generate
rate-limited errors, so the dependency limit is a hard ceiling on useful workers.

**Three sizing artifacts:**

- **Worker pool floor / ceiling.** *Floor* = steady-state ready-step throughput +
  burst headroom (so a sudden fan-out is not cold-started from zero) — sized from the
  median active-step rate, e.g. enough workers for the 90% of runs that finish in <1
  min plus the 8% hour-long runs' active steps. *Ceiling* = the binding downstream
  rate limit divided by steps-per-call, capped further by budget (mod-307). Autoscale
  between floor and ceiling on ready-step backlog.
- **Per-dependency concurrency caps.** A global in-flight limit per LLM/tool (e.g. "at
  most C concurrent calls to the sanctions API") so a fleet-wide burst cannot blow a
  shared rate limit. When a cap is hit, ready steps **queue in the durable plane**
  (safe — they are persisted), they do not fail. This is fleet-level backpressure.
- **Tail budget.** A max run age / approval TTL (the 7-day ceiling / 72h-24h gate TTLs
  from exercise-02) so the tail is *bounded*, plus **reserved capacity** for an
  approval-wave resume — a slice of the ceiling held back so that when a batch of
  sleeping runs resumes at once they do not starve fresh runs (see reflection 3).

### 2. Recovery matrix

| Scope | Failure | Detection signal | Recovery action | Resulting guarantee |
| --- | --- | --- | --- | --- |
| Worker | pod OOM / crash / spot reclaim | missing worker heartbeat; step exceeds schedule-to-start timeout | engine reschedules the in-flight step onto another worker; run resumes from last recorded step; idempotency (ex-01) makes the re-run safe | no progress lost; at most one step re-executed (safely) |
| Zone / AZ | whole AZ goes dark | health checks fail for all workers in the zone; durable-store replica in that AZ drops | workers in surviving zones pull the orphaned runs (durable store is multi-AZ replicated, no zone affinity on runs); runs resume | AZ loss = "surviving zones pick up the runs," not a fleet outage |
| Dependency | LLM/tool down, rate-limited, or timing out | activity errors / timeouts; rate-limit responses; circuit-breaker trips | activity-level retry with backoff; circuit-break to a fallback model/path if configured; if unrecoverable the run **PAUSES durably** (cheap wait) and resumes when the dependency returns or a human intervenes | a down dependency *pauses* runs, never loses or crashes them |

Every row's recovery is **reschedule the unfinished step, not restart the run**. That
holds because, and only because, the four enabling conditions are true:

```text
   because progress is DURABLE:
     recovery = "reschedule the unfinished step"   (NOT "restart the run")
   true at every scope IF:
     - durable store is multi-AZ HA           (the one thing you can't make stateless)
     - workers span zones, runs have NO zone affinity
     - steps are idempotent (exercise-01)     (re-run ≠ double side effect)
     - dependency failure PAUSES the run durably (cheap wait), not crashes it
```

### 3. Fix the AZ-outage incident

**Why it caused total errors.** Run state lived *in the workers*. When the AZ went dark
and a third of the workers vanished, the runs those workers held vanished with them —
there was no durable copy of their progress to resume from, so they all errored out. The
failure was architectural: state coupled to compute, and compute is exactly what an AZ
outage destroys.

**The three changes** that convert an AZ loss into a non-event:

1. **Multi-AZ durable store.** Run state lives in a store replicated across at least
   three AZs, so losing one AZ loses no state — the surviving replicas hold the full run
   history.
2. **Workers spread across zones.** The worker pool is distributed across AZs, so losing
   one AZ removes a fraction of capacity, not the whole pool; surviving-zone workers
   remain to pick up work.
3. **No zone affinity on runs.** A run is not pinned to a zone or a worker; any worker in
   any surviving zone can pull and resume any orphaned run.

With all three, an AZ loss becomes: the durable store keeps every run's state, the
engine detects the dead-zone workers' missing heartbeats, and workers in the surviving
zones reschedule those runs' unfinished steps. Capacity dips and recovery takes minutes;
**zero runs are lost.**

### 4. Fix the poison-run incident

**The failure mode.** A run with a malformed input (or a replay-divergence bug)
deterministically crashes whatever worker executes its next step. The engine, seeing the
step fail, reschedules it onto a fresh worker — which also crashes. Repeat: one bad run
walks through the fleet crashing worker after worker, degrading capacity for every other
run. Endless rescheduling of a deterministically-failing step is the trap.

**The fix — a per-run failure budget.** After **N = 5** consecutive step failures for a
single run, the engine stops rescheduling it and moves it to a **quarantine / dead-letter
state** for manual review, emitting an alert. N = 5 (not 1) tolerates transient failures
(a flaky network, a one-off OOM) while still catching a deterministic crasher quickly;
the destination is a dead-letter queue / manual-review state, *not* the active pool, so
the poison run can no longer touch a worker.

```text
   per-run failure budget : N=5 consecutive step failures -> QUARANTINE (dead-letter)
                            -> stop rescheduling; alert; await manual review
   -> one bad run can no longer crash workers fleet-wide
```

### 5. Bound blast radius

Per-run resource caps so a runaway fan-out is contained to a single run (connecting to
the cost-incident theme of mod-307):

```text
   per-run resource caps:
     max_concurrent_activities : e.g. 20   (caps parallel tool fan-out per run)
     max_total_steps           : e.g. 500  (caps an unbounded loop / recursion)
     max_spend                 : e.g. $50   (hard budget ceiling per run; mod-307)
   on breach -> pause the run for review (or fail it into quarantine), do NOT let it
                keep consuming the pool
```

A run that tries to spawn unbounded sub-runs or hammer tools in parallel hits its
concurrency or step or spend cap and is paused/quarantined — its blast radius is bounded
to *itself*, and the rest of the fleet is unaffected. Quarantine (item 4) and per-run
caps (item 5) are the two isolation mechanisms: together they guarantee a single
misbehaving run can neither crash workers fleet-wide nor exhaust the pool.

### Stretch: degradation, chaos drill, and deploy-as-failure

- **Graceful degradation on a partial LLM outage.** Two responses, chosen by *how
  long* the outage is expected to last and whether a fallback preserves correctness.
  **Circuit-break to a fallback model** (mod-307) when a degraded-but-correct answer is
  acceptable and the outage may be long — keep runs moving on the fallback. **Pause runs
  durably** until the primary returns when only the primary model is acceptable (a
  quality/compliance constraint) or the outage is expected to be brief — the durable
  wait is cheap, so pausing costs almost nothing. Pick fallback for availability,
  pause for correctness.
- **Chaos drill.** Experiment A: kill a random **one-third of workers**; success
  criterion = **zero lost runs** and recovery time (all orphaned steps rescheduled and
  progressing) under a target (e.g. 5 minutes). Experiment B: black-hole a **whole AZ**;
  same success criterion. The proving metric is **lost-run count == 0** (every run that
  was in flight before the fault is either complete or progressing after it) plus
  **schedule-to-start latency** returning under threshold — the same signal you autoscale
  on, now doubling as the recovery-time proof.
- **Is a deploy just a planned worker failure?** Mostly yes — a rolling worker
  replacement is exactly the "worker crash" row your recovery design already handles, so
  rainbow's drain is recovery applied deliberately. The *difference*: a deploy can also
  change the *code* (a replay-incompatibility), which a crash never does. So a deploy is
  a planned worker failure **plus** a version-compatibility concern — the part exercise-03's
  rainbow/pinning handles that plain recovery does not.

## Meeting the acceptance criteria

- **Replaces run-count autoscaling with ready-step backlog bounded by dependency rate
  limits, with floor/ceiling, per-dependency caps, and a tail budget** — the signal
  section gives the `min(...)` bound and all three sizing artifacts.
- **Recovery matrix complete across worker/zone/dependency, each recovery is "reschedule
  the step," four enabling conditions named** — the matrix fills all three rows and the
  throughline lists the multi-AZ store, cross-zone no-affinity workers, idempotent steps,
  and pause-not-crash dependency handling.
- **AZ-outage fix names the multi-AZ store, cross-zone workers, and no-affinity, and
  traces how an AZ loss becomes a non-event** — the three changes are listed and the
  surviving-zones-pick-up trace is written.
- **Poison-run fix specifies a per-run failure budget with a quarantine destination** —
  N = 5 consecutive failures -> dead-letter / manual-review, with the destination off the
  active pool.
- **Per-run resource caps bound a runaway run's blast radius to itself** —
  max_concurrent_activities, max_total_steps, and max_spend, with the on-breach action.

## Common pitfalls

- **Autoscaling on run count or CPU.** Sleeping approvals inflate run count and CPU lags
  bursty fan-out; both over-provision or thrash. Scale on ready-step backlog.
- **Scaling workers past the dependency ceiling.** Extra workers beyond
  `rate_limit / steps_per_call` only manufacture rate-limited errors. The dependency
  limit is a hard cap; excess demand must queue in the durable plane, not fail.
- **Leaving run state in the workers.** This is the exact cause of the AZ outage. State
  must live in a multi-AZ durable store; workers hold nothing that must survive a crash.
- **Endlessly rescheduling a failing step.** Without a per-run failure budget, one poison
  run crashes worker after worker. Cap consecutive failures and quarantine.
- **No per-run caps.** A runaway fan-out with no concurrency/step/spend cap exhausts the
  pool (and the budget). Caps bound the blast radius to a single run.

## Verification

A completed submission is correct when:

- The autoscaling signal is ready-step backlog (or schedule-to-start latency) bounded by
  `dependency_rate_limit / steps_per_call`, and the three sizing artifacts
  (floor/ceiling, per-dependency caps, tail budget) are all present.
- The recovery matrix has all three rows filled, each recovery action is "reschedule the
  step / resume the run," and the four enabling conditions are named explicitly.
- The AZ fix lists the multi-AZ store, cross-zone workers, and no zone affinity, and
  traces an AZ loss to "surviving zones pick up the orphaned runs, zero lost."
- The poison-run fix states an N and a quarantine/dead-letter destination off the active
  pool.
- Per-run caps include max concurrent activities, max total steps, and max spend, with
  an on-breach action that stops the run consuming the pool.
- `NOTES.md` answers the three prompts: the worker-OOM-after-ERP-write trace (exercise-01's
  idempotency key makes the rescheduled step return the same record_id), the durable
  store's HA/backup design and its blast radius if it dies (it is the single source of
  truth — the one component you cannot make stateless), and how the tail budget / reserved
  capacity stops a 5,000-approval Monday-morning wave from starving fresh runs.
