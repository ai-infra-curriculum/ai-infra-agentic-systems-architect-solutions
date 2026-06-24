# mod-310-agentic-developer-platforms/exercise-05-developer-tooling-adoption-and-dx — Solution

## Approach

The premise is blunt: **a developer platform nobody adopts is a failed platform,
regardless of technical merit.** The platform from exercises 1–4 can be
flawless and still die if engineers try it once, hit friction, and go back to
doing things by hand. This exercise designs the **DX architecture** that wins
adoption and keeps it — for a 50-engineer org that is *skeptical*, because a
previous internal tool flopped.

Adoption is won or lost in two moments: the **first ten minutes** (activation)
and the **first failure** (trust). I design for both, then build the loops that
keep the platform from decaying after launch. The work maps to the tasks:

1. Diagnose each bypass risk *in this skeptical org* and give it an
   architectural answer.
2. Design the first ten minutes: one-command install, sane defaults, one golden
   first task, with a time-to-first-success target.
3. Design the documentation strategy, separating agent-consumed skills from
   human task docs, and being honest about limits.
4. Design bidirectional feedback: telemetry out, one-keystroke bad-run report
   in — every bad run becoming a regression eval case.
5. Gate every platform change on evals (the mod-304 discipline).
6. Define *measurable* success signals per rollout phase.

The throughline reuses the plugin (exercise-01) as the activation lever and the
eval suite (mod-304) as the anti-decay gate.

## Reference solution

### 1. Bypass risks in a skeptical org, with architectural answers

| Failure mode | How it manifests here (skeptical, prior tool flopped) | Architectural answer |
| ------------ | ----------------------------------------------------- | -------------------- |
| High activation cost | "The last tool needed a wiki and six config steps; I'm not doing that again." Engineers never start. | **One-command plugin install** (exercise-01) with **sane defaults** that work at zero config. |
| Untrustworthy output | The agent does one dumb thing on a watching skeptic's first run; they tell the team channel; nobody else tries it. | **Guardrails + visible reasoning + honest limits** — trust comes from transparency, not just accuracy. |
| Invisible value | "I don't know what it's actually good at, so I won't risk a real task on it." | **Discoverability**: a short list of **3–5 golden tasks** and in-context "what can you do here?" |
| No feedback channel | Annoyance has nowhere to go but Slack venting, which the platform team never sees turned into a fix. | **One-keystroke in-loop bad-run report** that routes the trajectory to the platform team. |
| Silent breakage | A model or API change quietly degrades it; the skeptics' "I knew it" is confirmed and adoption craters. | **Eval-gated releases** + usage/quality telemetry the platform team watches. |

For this org specifically, the binding constraint is *reputation debt* from the
prior flop. Every answer above is also a credibility move: low activation cost
signals "we respect your time," honesty about limits signals "we won't
overpromise like last time," and eval-gating signals "we won't let it silently
rot."

### 2. The first ten minutes

- **One-command install.** `claude plugin install acme-sdlc-platform` (from the
  org marketplace, exercise-01). One command yields every subagent, skill, hook,
  and internal-service MCP connection the platform team curated. No per-person
  setup wiki, no copy-pasting config.
- **Sane, opinionated defaults.** It does something useful with **zero**
  configuration: the protected-path guard, the PR-conventions skill, the
  security-review subagent, and Jira/GitHub MCP access are all on by default.
  Every required choice before first value is a drop-off point, so there are
  none.
- **One golden first task.** Onboarding hands the engineer a single concrete win,
  not a feature tour: *"Ask it to fix this flaky test in your service."* First
  value beats first completeness.
- **Target: time-to-first-success < 15 minutes** from install to a green,
  merged-or-mergeable result on the golden task. This is the headline activation
  metric.

### 3. Documentation strategy

Two audiences, two kinds of docs:

- **Agent-consumed docs (skills as executable conventions).** The
  `pr-conventions` and `commit-style` skills (exercise-01) *are* the
  documentation of how Acme writes PRs and commits — and the agent reads and
  applies them. This pays twice: humans read the same skill the agent enforces,
  collapsing the usual "the wiki says X but the tooling does Y" gap. The cost of
  sloppy skills is also doubled — a vague skill mis-documents *and* mis-behaves.
- **Human docs (task-oriented, honest).** Engineers want "how do I get the agent
  to fix a flaky test?" not "reference for all 14 MCP tools." Lead with the
  golden tasks.

**The 3–5 golden tasks (what it's genuinely good at):**

1. Fix a failing/flaky CI test on an existing service.
2. Implement a small, well-specified ticket and open a conventions-compliant PR.
3. Run a focused security review on a diff before merge.
4. Triage a production incident from logs into a 5-line root cause.
5. Update a ticket's status and comment back when CI goes green.

**Honest "not good at yet" list (managing expectations *is* an adoption tool):**

- Large cross-cutting refactors spanning many modules.
- Greenfield architecture decisions with no existing pattern to follow.
- Anything touching `infra/`, `*.tf`, or migrations (gated by design,
  exercise-02) — it proposes, a human disposes.
- Ambiguous tickets with no acceptance criteria — it will ask, not guess.

Overpromising and underdelivering is how you lose a skeptic permanently; the
honest list is a feature, not a hedge.

### 4. Bidirectional feedback loops

```text
   ENGINEERS                          PLATFORM TEAM
   ─────────                          ─────────────
  use the agent  ───telemetry──────▶  usage + outcome metrics
       │                               (which tasks, success rate,
       │                                where it gives up / gets corrected)
       │◀──improvements───────────     │
       │   (new skills, fixed tools,    ▼
       │    better defaults)      eval suite gates every plugin
       │                          change (mod-304)
       │                               ▲
  one-key  ──"this run was bad"──▶  triaged bad runs →
  in-loop      captures trajectory   become regression eval cases
```

- **Telemetry out.** Track which tasks engineers actually use the agent for, the
  success rate per task type, and *where it gives up or gets corrected*. This is
  the product signal that tells the platform team what to build next, and it
  reuses the observability instrumentation from the evaluation track.
- **One-keystroke bad-run report in.** A single keystroke captures the run's
  trajectory (the task, the tool calls, the output) and routes it to the
  platform team — far better than hoping people file tickets. **Every reported
  bad run becomes a regression eval case** (mod-304), so the same failure cannot
  silently return after the next model or API change.

### 5. Eval-gated releases

The platform is software; ship it like software. **A change to a skill, a tool,
or the model version passes the eval suite before rollout.** The eval suite
includes the golden tasks *and* the accumulated bad-run-to-eval-case regressions
from Section 4. A change that regresses any of them does not ship — the
regression is caught before an engineer hits it. This closes the loop:
field failures become permanent guards, and silent breakage — the fastest way to
lose the trust this org is already short on — is designed out.

### 6. Success signals per rollout phase

| Phase | Move | Success signal (measurable) |
| ----- | ---- | --------------------------- |
| Activate | one-command install; 3–5 golden tasks documented | **time-to-first-success < 15 min** |
| Trust | guardrails + visible reasoning; honest "not good at yet" list | **return-after-failure rate** — % who run again after their first imperfect run |
| Discover | in-context "what can you do here?"; task-oriented docs | **usage spread** — # distinct engineers/teams beyond the pilot |
| Feedback | one-key bad-run report → eval case; usage telemetry | **bad-run-to-eval-case conversion** — every reported bad run becomes a regression case (count > 0, ratio → 1) |
| Sustain | eval-gated plugin releases; metrics reviewed weekly | **regression count at release = 0**; adoption curve holds (weekly active flat-or-up) |

Adoption is the deliverable, and every phase signal is a number, not a vibe.

### Stretch: champions, changelog, weekly review

**Champions / pilot rollout.** Seed with **3–5 respected engineers** across
different teams (not just the keenest early adopters — credibility with skeptics
comes from peers they respect). They get it first, run the golden tasks on real
work, and their bad-run reports shape the skills and defaults *before* the wider
launch. A skeptic is far more likely to try a tool their trusted teammate
already vouches for.

**Platform changelog (for engineers, not just the team).** Belongs in it:
"the agent now follows the new PR template," "fixed the flaky-test debugger
giving up on async failures," "security review now catches SSRF." Builds trust
by showing the platform *responds* to feedback. **Noise to keep out:** internal
refactors, dependency bumps with no behavior change, and metric dashboards — an
engineer-facing changelog is about *what changed for you*, not platform-team
housekeeping.

**Weekly metric review — the 3–4 numbers and the action each triggers:**

- **Time-to-first-success** trending up → friction crept into activation; audit
  the install/golden-task path.
- **Return-after-failure rate** dropping → the first-failure experience is
  losing people; tighten guardrails/transparency or the honest-limits list.
- **Usage spread** flat → discoverability problem; the platform isn't being
  found beyond the pilot.
- **Regression count at release** > 0 → a change slipped the eval gate;
  tighten the suite before the next rollout.

## Meeting the acceptance criteria

- **Each bypass risk has a concrete manifestation + architectural answer.**
  Section 1's table grounds all five failure modes in *this* skeptical,
  prior-flop org and matches each to an architectural control.
- **One-command activation with sane defaults and a single golden first task,
  with a TTFS target.** Section 2 specifies the one-command install, zero-config
  defaults, the "fix this flaky test" golden task, and < 15 min.
- **Doc strategy separates agent skills from human task docs, honest about
  limits.** Section 3 distinguishes the two, lists 3–5 golden tasks, and gives an
  explicit "not good at yet" list.
- **Bidirectional feedback routing every bad run into a regression eval case.**
  Section 4 specifies telemetry out and the one-keystroke report in, with each
  bad run becoming an eval case.
- **Eval-gated releases; every phase has a measurable signal.** Section 5 gates
  every change on the eval suite (including accumulated regressions); Section 6's
  table gives each phase a numeric signal.

## Common pitfalls

- **Optimizing average accuracy, ignoring the first failure.** A skeptic's
  *first* imperfect run, not the mean, decides whether they return. Design what
  the agent does when unsure (ask, show reasoning, decline) — that moment is
  worth more than a percentage point of average quality.
- **Feature-tour onboarding.** Walking a new user through all 14 tools instead
  of handing them one win. First value beats first completeness; lead with the
  golden task.
- **Skills treated as throwaway prose.** Sloppy skills mis-document *and*
  mis-behave because the agent executes them. They pay twice when good and cost
  twice when bad — invest in them like code.
- **Feedback that goes nowhere.** A thumbs-down that doesn't capture the
  trajectory or reach the platform team is theater. Capture the run and route it,
  and turn every report into an eval case.
- **Shipping platform changes without eval gates.** A model or API bump that
  silently regresses a golden task confirms the skeptics' fears and craters
  adoption. Gate every change on the eval suite.

## Verification

- **Activation timing.** Run the install-to-first-success path on a fresh
  machine with a stopwatch and confirm < 15 minutes on the golden task; if it
  exceeds, find and remove the friction step.
- **First-failure dry-run.** Force an ambiguous/underspecified task and confirm
  the agent asks or declines with visible reasoning rather than confidently
  doing the wrong thing — the behavior that retains a skeptic.
- **Feedback round-trip.** Trigger the one-keystroke bad-run report, confirm the
  trajectory is captured and reaches the platform team, and confirm it is
  converted into a regression eval case in the suite.
- **Eval-gate test.** Introduce a deliberate regression (break a golden task in a
  skill), run the release gate, and confirm the change is blocked before rollout.
- **Signal instrumentation check.** Confirm each Section 6 signal is actually
  emitted by telemetry (TTFS, return-after-failure, usage spread, regression
  count) so the weekly review has real numbers to act on.
- **Markdownlint** clean: one H1, every fence tagged (`text`), blank lines around
  blocks, dash bullets, trailing newline.
