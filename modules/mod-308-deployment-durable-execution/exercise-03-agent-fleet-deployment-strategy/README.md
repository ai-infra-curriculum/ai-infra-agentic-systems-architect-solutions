# mod-308-deployment-durable-execution/exercise-03-agent-fleet-deployment-strategy — Solution

## Approach

The deliverable is a deploy *process*, not CI/CD YAML: diagnose why the team's rolling
deploy keeps paging them, pick the strategy that never severs an in-flight run, write
the versioning policy that bounds how many versions can coexist, and hand on-call an
ordered runbook with a real rollback path. The whole exercise hangs on one realization
from Chapter 2 — **drain means seconds for a web service and days for an agent** — and
on the fix that follows: rainbow (never-replace, version-pinned) deploys.

The choices that drive the solution:

- **Diagnose before prescribing.** The rolling deploy fails for a precise mechanical
  reason (a 30-second drain timeout cannot wait out a multi-day run, so it kills the
  run's worker), and blue-green fails worse (it mass-kills every BLUE run at cutover).
  Raising the timeout is not a fix — you cannot hold a deploy open for three days.
- **Rainbow / never-replace is the only strategy that fits.** Deploy each version
  alongside the others, pin new runs to the newest, let old versions drain to zero
  before retiring them. Temporal Worker Versioning / Build IDs is the concrete
  mechanism; the contract is identical without an engine (version-tagged deployments +
  run pinning at the router).
- **Pin by default; patch only when forced.** Old runs finish on the code they started
  on (zero replay risk). Patching — forcing a change into in-flight runs via a
  version-guarded branch — is reserved for security fixes and broken downstream
  contracts that cannot wait for runs to drain.
- **Bound the rainbow.** A version-retention policy (max N live versions) plus a
  run-age TTL (the approval TTL from exercise-02) stop versions accreting without end
  when 3–5 deploys/day meet slow-draining runs.

## Reference solution

### 1. Diagnosing the current failure

**Why rolling with a 30s drain severs the multi-day runs.** A rolling deploy replaces
pods one at a time: drain a pod, kill it, start a v2 pod, repeat. "Drain" means "wait
for in-flight work on this pod to finish, up to the drain timeout, then kill it." For a
web service in-flight work is HTTP requests that finish in milliseconds, so a 30-second
drain is generous. For an agent, the in-flight work on a pod may be a run that is *three
days* from finishing (sleeping on a document upload or a human approval). The drain
timer expires after 30 seconds, the pod is killed, and the run's worker dies. If that
run's state lived only in the worker it is lost; even with durable execution it is now
*stranded* — its state survives, but every v1 worker is being replaced by v2, so there
may be no compatible worker to resume it on, and if v2 changed the step sequence it
cannot validly replay on v2 either.

**Why raising the timeout is not a fix.** To drain a three-day run you would hold the
deploy open for three days — during which you ship 9–15 more deploys (3–5/day). The
deploy would never complete; the queue of pending deploys would grow without bound. The
drain-timeout knob assumes work is short-lived; the workload violates that assumption,
so no value of the knob is correct.

**Why blue-green is worse.** Blue-green stands up a full GREEN v2 environment, cuts all
traffic over, and tears down BLUE v1:

```text
   before cutover            after cutover
   --------------            -------------
   [ BLUE  v1 ] ◀-traffic    [ BLUE  v1 ]  ◀- TORN DOWN  (kills every long run on v1)
   [ GREEN v2 ]              [ GREEN v2 ] ◀-traffic

   blue-green mass-kills at cutover; rolling kills pod-by-pod.
   Same root cause: lifetime of the run is coupled to lifetime of the process,
   then the process is replaced.
```

The cutover is instant and safe for stateless requests but catastrophic for an agent
fleet: every run still executing on BLUE is severed the moment BLUE is torn down, and
those runs have no v1 worker left to resume on.

### 2. The strategy: rainbow / versioned never-replace

```text
   ┌──────────────────────────────────────────────────────────────────────┐
   │  RAINBOW: every live version coexists                                  │
   │                                                                        │
   │  v1 workers ── runs r17, r88        (draining; run-age TTL running)    │
   │  v2 workers ── runs r402…r900       (draining)                         │
   │  v3 workers ── ALL new runs         (active)                           │
   │                                                                        │
   │  new run ──▶ pinned to NEWEST version (v3)                             │
   │  old run ──▶ stays on the version it STARTED on                        │
   │                                                                        │
   │  retire v1  ⟺  in-flight(v1) == 0   OR   run-age TTL fires             │
   │  retention :  at most N live versions; exceeding N forces oldest to    │
   │               migrate (patch) before the new deploy proceeds           │
   └──────────────────────────────────────────────────────────────────────┘
```

**Argument against the alternatives, for this workload.** Rolling and blue-green both
*remove* the version a run depends on while that run is still in flight; with multi-day
runs there is no drain window short enough to make removal safe. Rainbow never removes a
version with in-flight runs — that is precisely what rolling/blue-green cannot promise.
What rainbow provides that they do not: a run always has a **version-compatible worker**
to resume on, because its color stays alive until its runs drain. New runs go to the
newest color; old runs ride their original color to completion.

**Concrete mechanism.** Temporal **Worker Versioning / Build IDs**: each worker
advertises a build ID, the server pins each run to a build ID at start, and dispatches
tasks only to compatible workers. The server/router responsibilities: **pin new runs to
the newest build ID** and **keep old runs on theirs**, dispatching each task only to a
worker whose build ID matches the run's. Without an engine, the identical contract is
version-tagged deployments plus a router that records each started run's version and
routes its subsequent steps back to a matching deployment.

### 3. Versioning policy

- **Default: version pinning.** A run finishes on the code it started on — zero replay
  risk, at the cost of running multiple versions concurrently. This is the baseline.
- **Patching: only when forced.** A change is patched into in-flight runs (guarded by a
  recorded version marker so old runs replay the old path and new runs take the new one)
  only for: (a) a **security fix** that must reach runs that will live for days, or (b)
  a **broken downstream contract** (an API the in-flight runs call has changed
  shape) that would otherwise fail every in-flight run. Everything else pins.
- **Version-retention policy.** Support at most **N = 5** concurrent live versions. If a
  new deploy would create a 6th, the oldest still-live version's runs are **force-migrated
  via patching** (or aged out by TTL) *before* the deploy proceeds. This bounds the
  rainbow regardless of deploy rate.
- **Run-age TTL.** A run may keep its version alive for at most the **approval TTL from
  [exercise-02](../exercise-02-hitl-approval-architecture/README.md)** (72h for the
  compliance gate, 24h for the wire gate; a global ceiling of 7 days for any run). When
  the TTL fires, the run is escalated, cancelled, or force-migrated — so a sleeping
  approval cannot pin a color alive indefinitely. **One TTL, three payoffs**: it times
  out the approval (ex-02), bounds version sprawl (here), and bounds the fleet tail
  (exercise-04).

### 4. Deploy runbook

```text
  1. Build v_next; tag workers with build ID v_next.
  2. Deploy v_next workers ALONGSIDE all current versions — remove NOTHING.
  3. Server pins all NEW runs -> v_next; existing runs stay on their version.
  4. WATCH (per-version):
       - in-flight count per version
       - error rate per version
       - replay-failure count  ── MUST stay 0  (any non-zero = poisoned replay, halt)
       - rate of new runs landing on v_next (confirms pinning works)
  5. HEALTHY path:
       - let old versions drain naturally as their runs complete
       - retire a version when in-flight(version) == 0  OR its run-age TTL forces
         migration of the stragglers
       - if live versions would exceed N=5, force-migrate the oldest before next deploy
  6. UNHEALTHY path (instant rollback):
       - STOP pinning new runs to v_next
       - re-pin NEW runs -> previous version (its workers are STILL ALIVE)
       - investigate v_next; retire it once its (few) runs drain or are migrated
```

The monitoring signal that distinguishes "fine" from "stop now" is **replay-failure
count per version**: rolling/blue-green cannot even surface it (the runs are already
dead); under rainbow it must read zero, and any non-zero value means a deploy introduced
a replay-incompatible change that is poisoning in-flight runs — halt and roll back.

### 5. The rollback property

Rainbow rollback is **instant and lossless** because the previous version was never
removed: rolling back is just "re-pin new runs to the still-running previous color." No
redeploy, no rebuild, no scramble — the previous color is already serving its own runs,
so pointing new runs back at it is a routing change that takes effect immediately, and
no in-flight run is touched.

Contrast with blue-green: after cutover, BLUE has been *torn down*. Rolling back means
standing BLUE back up from scratch (a full redeploy) and hoping the runs it was serving
were not already lost at cutover — they were. Rollback under blue-green is a frantic
rebuild that cannot recover the severed runs; rollback under rainbow is a one-line route
change that loses nothing. That asymmetry is the direct payoff of never-replace.

### Stretch: canary, observability, and replay-safe changes

- **Canary on top of rainbow.** Pin only **5% of new runs** to v_next (the router
  weights the pin), keep 95% on the previous color, watch v_next's error and
  replay-failure rates, then ramp 5% -> 25% -> 100%. It composes cleanly with pinning
  because every run is still pinned to *a* version for its whole life; canary only
  changes the *split* of new runs, not whether a run can migrate. Rollback during canary
  is even cheaper — re-pin the 5% back.
- **On-call observability view.** A per-version panel: in-flight count, error rate,
  replay-failure count (alarmed at > 0), p50/p99 run age (to see which versions are
  TTL-bound), and new-run pin distribution. Signals come from the workflow engine's
  per-build-ID metrics (Temporal exposes task and replay metrics by build ID) plus the
  router's pin counters.
- **Replay-safe vs unsafe changes (reconcile with exercise-01).** Replay-*safe* to
  deploy without patching: adding a new activity *after* the current step frontier,
  changing activity-internal logic (the workflow only sees recorded results),
  performance fixes inside activities. *Unsafe* (needs pinning or a version-guarded
  patch): reordering or removing steps in the workflow, changing the branch structure,
  adding a step *before* the current frontier — anything that changes the recorded step
  *sequence*, because that is exactly what makes replay diverge (the exercise-01
  "add a fourth check" case).

## Meeting the acceptance criteria

- **Correctly diagnoses why rolling (and blue-green) severs multi-day runs, including
  why raising the drain timeout fails** — the diagnosis ties the 30s drain to killing a
  3-day run's worker, shows raising the timeout would stall the deploy queue forever,
  and shows blue-green mass-killing at cutover.
- **Selects rainbow/versioned never-replace and argues down the alternatives** — the
  argument is that rolling/blue-green remove a version while its runs are in flight,
  which rainbow never does; the concrete mechanism (Worker Versioning / Build IDs) is
  named with the server's pin-and-route duties.
- **Versioning policy: pin by default, when-to-patch, retention + run-age TTL** — pin is
  the baseline, patch is reserved for security fixes / broken contracts, N=5 retention
  with forced migration on overflow, and the run-age TTL inherited from exercise-02.
- **Ordered runbook naming the monitoring signals (replay-failure == 0) and both paths**
  — the six-step runbook lists the watch signals including replay-failure==0, and covers
  drain-and-retire and instant rollback.
- **Explains instant, lossless rollback under rainbow** — rollback is a route change back
  to the still-alive previous color, contrasted against blue-green's teardown-and-rebuild.

## Common pitfalls

- **Treating "raise the drain timeout" as the fix.** It cannot wait out a multi-day run
  without stalling every subsequent deploy. The drain knob is the wrong lever; the fix is
  to stop removing in-flight versions at all.
- **Deploying a step-sequence change without a version guard.** Reorder, remove, or
  insert-before-frontier and in-flight runs diverge on replay (replay-failure > 0). Such
  changes must pin (old runs finish on old code) or be patched behind a recorded version
  marker.
- **Unbounded version retention.** With 3–5 deploys/day and slow drains, colors accrete
  until they exhaust capacity. A retention cap (N) plus a run-age TTL is mandatory, not
  optional.
- **No replay-failure alarm.** Per-version error rate alone misses the specific failure
  rainbow is meant to prevent. Replay-failure count must be a first-class, alarmed
  signal that gates the healthy/unhealthy decision.
- **Rolling back by redeploying.** Under rainbow you never need to; re-pin new runs to
  the previous color. Spinning up the old version from scratch reintroduces the
  blue-green rollback gap and risks losing runs.

## Verification

A completed submission is correct when:

- The diagnosis explains *mechanically* why a 30s drain kills a multi-day run and why no
  larger timeout fixes it, and shows blue-green mass-killing at cutover.
- The chosen strategy is rainbow/never-replace, names Worker Versioning / Build IDs (or
  the engine-free equivalent), and specifies that the server pins new runs to the newest
  version and keeps old runs on theirs.
- The versioning policy states pin-by-default, lists the narrow conditions that justify
  patching, and bounds live versions with both a retention cap and a run-age TTL tied to
  the exercise-02 approval TTL.
- The runbook is ordered, deploys alongside (removes nothing), names in-flight/version,
  error-rate/version, and **replay-failure == 0** as watch signals, and includes the
  instant-rollback path that re-pins new runs to the previous color.
- The rollback explanation contrasts rainbow (route change to a live color) against
  blue-green (teardown + rebuild) and states why rainbow loses nothing.
- `NOTES.md` answers the three prompts: the 9-day sleeping-approval case (TTL fires ->
  escalate/migrate; the trade-off chosen), a change that *must* be patched (a downstream
  contract break, guarded so old runs replay the old path), and the version-count math
  (3–5 deploys/day x drain time, bounded by N and the TTL).
