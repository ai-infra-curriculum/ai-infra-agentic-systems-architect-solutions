# mod-302-multi-agent-orchestration/exercise-04 — Solution

## Approach

This is the spec that governs the unhappy path, so the discipline is that every rule
has to be concrete enough to implement without a follow-up question — a number, a
deterministic tie-breaker, a named accountable owner. Two cross-cutting invariants
from Chapter 4 organize the whole thing: **bound every loop** (a tripped bound steps
*up* the ladder, it never crashes) and **never fabricate to avoid escalating** (the
ladder must always have a sanctioned "admit the gap" rung). The travel system makes
both invariants bite: workers can pick incompatible dates (a real conflict that must
resolve deterministically), and some bookings are irreversible and over a cost
threshold (a hard human gate), and the assistant must never invent a booking that
doesn't exist (the no-fabrication invariant, made literal).

Design choices that drive the spec:

- **Orchestrator-mediated coordination, not a blackboard.** The minimum mechanism
  that keeps three workers coherent is to route all cross-worker state through the
  orchestrator (Chapter 4 default: simplest to reason about). A shared blackboard
  would let flights and lodging see each other's date picks mid-flight, but it adds a
  concurrency surface (locking, read-your-writes) to govern, and the conflict here is
  rare and resolvable *after* the fact. Blackboard is the stretch goal, deliberately
  declined for the baseline, with the concurrency cost named.
- **A deterministic, ordered tie-breaker for date conflicts.** When the flight worker
  and lodging worker disagree on dates, the orchestrator does not re-ask and hope — it
  applies a fixed precedence rule (flights win the date anchor, because flight
  availability/price is the least flexible input) and re-tasks lodging against the
  fixed dates. If even that can't produce a compatible plan within the re-decompose
  cap, it steps to the human gate. "Can't decide" has a defined path; it is not a hang.
- **Every loop has a numeric cap and a ladder step on trip.** Turns per worker, total
  worker count, re-decompose iterations, wall-clock, and spend each get a number, and
  each trip maps to a specific rung — never to a crash.
- **Two human gates, both with timeout defaults.** The irreversible over-threshold
  booking is gated (risk gate); the unresolvable date conflict is gated (judgment
  gate). Each specifies trigger, what the human sees, a timeout *default* (a gate with
  no timeout default is itself a deadlock, Chapter 4), and an accountable owner.
- **Partial failure flags the gap.** If activities fails but flights and lodging
  succeed, the assistant returns the two-of-three plan and says so plainly — it does
  not invent activities to fill the hole.

## Reference solution

### Coordination mechanism

**Orchestrator-mediated** state. All cross-worker information (chosen dates,
destination, budget envelope) flows through the orchestrator; workers never read each
other directly. This is the minimum mechanism that keeps the system coherent for a
fan-out where the only real cross-worker dependency is the date anchor. The
concurrency surface taken on is **none** — there is no shared mutable store, so there
is nothing to lock. (If a blackboard were adopted later to pre-empt the date conflict,
the added surface is a lock or last-writer-wins rule on the shared `dates` cell plus a
read-your-writes guarantee for workers — named here so the tradeoff is explicit.)

### Conflict-resolution rule

```text
DATE CONFLICT (flight worker vs lodging worker pick incompatible dates)

  deterministic tie-breaker:
    1. FLIGHTS anchor the dates. Flight availability + price is the least flexible
       input, so the flight worker's chosen date window wins.
    2. Orchestrator re-tasks the LODGING worker against the fixed flight dates
       (this consumes one re-decompose iteration).
    3. If lodging cannot find compatible lodging within the re-decompose cap,
       the tie-breaker CANNOT decide ──▶ step to the judgment human gate
       (present both partial plans; human picks or adjusts dates).

  this is deterministic: same conflict, same flight-anchor rule, same outcome.
  "can't decide" is NOT a hang — it has an explicit rung (the judgment gate).
```

### Loop bounds

```text
| Loop bound                  | Cap                    | On trip (steps UP the ladder)                |
| --------------------------- | ---------------------- | -------------------------------------------- |
| Turns per worker            | 10 turns               | return partial result, mark axis incomplete  |
|                             |                        |   (rung 2 degrade), do not loop              |
| Fan-out depth / agent count | depth 1, ≤ 3 workers   | reject any spawn beyond the 3 named workers; |
|                             |   (flights/lodging/    |   flat only — no sub-orchestrators           |
|                             |    activities)         |                                              |
| Re-decompose iterations     | 2                      | after 2 re-tasks without convergence,        |
|                             |                        |   step to judgment human gate (rung 3)       |
| Total wall-clock            | 90 s                   | synthesize from whatever returned (rung 2),  |
|                             |                        |   flag the timed-out axis                    |
| Total spend                 | budget cap (token $)   | halt new work, degrade to partial (rung 2),  |
|                             |                        |   never silently exceed                      |

  invariant: a tripped bound steps up the escalation ladder. It NEVER crashes the run.
```

### Escalation ladder

```text
ESCALATION LADDER

  rung 0  retry once
    trigger: transient error (search API timeout, 5xx)
    this system: a flight search call times out ──▶ retry the flight worker once

  rung 1  reassign / re-decompose
    trigger: bad assignment or resolvable conflict
    this system: lodging incompatible with flight dates ──▶ re-task lodging against
                 the flight-anchored dates (≤ 2 iterations)

  rung 2  degrade (partial), flag the gap
    trigger: a worker fails or a bound (turns/wall-clock/spend) trips
    this system: activities worker fails ──▶ return flights + lodging, flag
                 "activities unavailable" (see partial failure below)

  rung 3  human-in-the-loop
    trigger: judgment/risk gate — unresolvable conflict OR irreversible booking
    this system: re-decompose cap hit with no compatible dates ──▶ judgment gate;
                 irreversible booking over threshold ──▶ risk gate

  rung 4  safe failure
    trigger: even the human path is unavailable or declines, and no safe partial exists
    this system: no flights at all for the destination/dates ──▶ return
                 "couldn't build a plan: no available flights", never invent one

  invariant: the model may NEVER fabricate a booking to avoid a higher rung.
```

### Human gates

```text
| Human gate                       | Trigger                          | Human sees                          | Timeout default        | Accountable        |
| -------------------------------- | -------------------------------- | ----------------------------------- | ---------------------- | ------------------ |
| Irreversible booking > threshold | about to commit a non-refundable | the exact booking, price, cancel    | 15 min ──▶ DO NOT book | the user (it's     |
|   (RISK gate)                    | booking over the $ threshold     | terms, and the full itinerary       |   (fail safe: no spend)| their money) +     |
|                                  |                                  |                                     |                        | the team's gate    |
| Unresolvable worker conflict     | re-decompose cap hit, no         | both candidate plans + the date     | 10 min ──▶ present     | the team           |
|   (JUDGMENT gate)                | compatible dates found           | conflict, asks which to take/adjust |   plans, book nothing  |   (design owner)   |
```

Both gates fail **safe**: on timeout the irreversible booking is *not* made and the
unresolved conflict books *nothing*. A gate that defaulted to "book anyway" on timeout
would convert a deadlock into an irreversible mistake — exactly the outcome the gate
exists to prevent.

### Partial failure behavior

If the **activities worker fails** but flights and lodging succeed, the assistant
returns the two-of-three plan and says so explicitly:

```text
  "Planned your flights and lodging. I could not complete the local-activities
   research (the activities worker failed), so this itinerary has no activities
   yet — want me to retry just that part?"
```

It returns the real flights and the real lodging, names the missing piece, and offers
a bounded retry. It does **not** invent plausible-sounding activities to fill the hole
— a complete-looking itinerary with fabricated activities is the bug this rule
prevents (Chapter 4: "a confident answer that silently dropped a source is a bug").

## Meeting the acceptance criteria

- **Minimum coordination chosen and justified; concurrency surface named** —
  orchestrator-mediated (the default, simplest to reason about), with the blackboard
  alternative declined and its added concurrency surface (locking + read-your-writes
  on the shared `dates` cell) named explicitly.
- **Deterministic conflict rule with a defined can't-decide path** — flights anchor
  dates, lodging is re-tasked, and an unresolved conflict after the re-decompose cap
  steps to the judgment gate. Same conflict → same outcome.
- **Every loop bound has a numeric cap and a ladder-step on trip** — five bounds, each
  with a number and an explicit "steps up the ladder, doesn't crash" entry.
- **Five-rung ladder, each mapped to a concrete failure, no-fabrication invariant** —
  retry (API timeout), reassign (date conflict), degrade (activities fails), human
  (unresolvable conflict / irreversible booking), safe-failure (no flights at all),
  with the never-fabricate-a-booking invariant stated.
- **Human gates fully specified; irreversible booking gated** — both gates list
  trigger, what the human sees, a timeout *default*, and an accountable owner; the
  irreversible over-threshold booking is the risk gate and fails safe (no booking on
  timeout).
- **Partial failure flags the gap honestly** — flights + lodging returned, activities
  named as missing, bounded retry offered, nothing fabricated.

## Common pitfalls

- **A non-deterministic conflict rule.** "The orchestrator decides which dates win"
  is not implementable — decides *how*? The rule must be a fixed precedence (flights
  anchor) with a defined fallback (re-task, then judgment gate). A tie-breaker that
  can't articulate its precedence will hang on the conflict the system is guaranteed
  to hit.
- **A human gate with no timeout default.** Chapter 4 is explicit: a gate without a
  timeout default *is* a deadlock. Every gate needs a default action on timeout, and
  for irreversible spend that default must fail safe (do not book).
- **Bounds that crash instead of escalating.** A turn cap that throws is a reliability
  incident. Every tripped bound must map to a ladder rung — usually degrade-and-flag —
  so the system returns a partial answer, not an error.
- **Fabricating to look complete.** The strongest pull in a travel planner is to fill
  a failed worker's slot with plausible content (invent activities, invent a flight).
  The no-fabrication invariant and the "flag the gap" partial-failure rule exist
  precisely to make the honest path the implemented one. A made-up booking is the
  worst possible failure here.
- **Adding a blackboard reflexively.** A shared state store feels like the way to stop
  date conflicts, but it imports locking and consistency obligations for a conflict
  that the orchestrator-mediated tie-breaker resolves cheaply. Add the blackboard only
  if mid-flight visibility is genuinely required, and govern the concurrency surface
  when you do.

## Verification

A completed submission is correct when:

- The coordination mechanism is the *minimum* that keeps three workers coherent
  (orchestrator-mediated), and if a blackboard is chosen instead, its concurrency
  surface is named and governed.
- The conflict rule is deterministic (a stated precedence) and has an explicit
  can't-decide path (the judgment gate), not a re-ask loop.
- Every loop bound is a concrete number with a "steps up the ladder" entry; none crash.
- The ladder has all five rungs, each mapped to a concrete failure from *this* system,
  with the never-fabricate-a-booking invariant written down.
- Both human gates specify trigger, what the human sees, a timeout default, and an
  accountable owner; the irreversible over-threshold booking is gated and fails safe.
- The partial-failure output returns the real two-of-three plan and names the missing
  axis — no fabricated activities.
- `NOTES.md` answers the three prompts: the hardest bound to set a number for (the
  wall-clock / spend caps, tuned from production p95 latency and per-plan cost data);
  the deadlock the spec prevents (date ping-pong without the flight-anchor rule, or a
  gate with no timeout default) and the exact rule that breaks it; and where the system
  is most tempted to fabricate (a failed worker's slot) and what makes the honest path
  easy (the flag-the-gap rung plus the no-fabrication invariant).
