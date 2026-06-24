# mod-310-agentic-developer-platforms/exercise-04-platform-adoption-and-dx-plan — Solution

## Approach

This is a developer-experience and adoption-architecture exercise. The platform
shipped and stalled: many tried once, few returned. The work is to diagnose the
two exit-interview patterns against the **two decisive moments** — *activation*
(the first ten minutes) and *trust under failure* (the first failure) — name the
*root cause* of each (not the symptom), then design the activation path, the
trust-under-failure design, two skills that pay twice, the versioning policy, and
the eval-gated feedback loop.

The reference keeps the "users" deliberately role-agnostic — they "may be
developers, on-call engineers, support staff, or analysts" — and never assumes the
platform is a coding tool or that the first task is an SDLC task. The activation
path is written so it works whether the first valuable result is a drafted PR, a
triaged incident, a resolved ticket, or an answered analytics question. Adoption
is an *engineered outcome*, not a hope: each design choice ties to a moment and a
measurable signal.

The design choices that drive everything below:

- **Two moments decide adoption.** Activation (does the first ten minutes pay?)
  and trust under failure (does the first wrong answer cost trust once, or cause an
  incident?). Every fix maps to one of these.
- **Diagnose to root cause.** "30 minutes of setup" is the symptom; the root cause
  is *setup friction* — the platform made activation the user's job. "Irreversible
  bad action" is the symptom; the root cause is a *reversibility-ladder failure* —
  a wrong answer was allowed to be irreversible.
- **The versioned bundle is the activation and versioning lever.** One-step install
  ships the agents/skills/tools/MCP servers pre-wired; the same bundle versioning
  is what forbids silent behavior changes.
- **The feedback loop must make the platform improve under model change**, not
  regress: every reported bad run becomes a permanent regression eval case that
  gates every future change.

## Reference solution

### 1. Diagnose against the two decisive moments

| Exit-interview pattern | Decisive moment | Root cause (not symptom) |
| --- | --- | --- |
| New users spent 30+ minutes wiring tools and hunting credentials; most quit before the first result | **Activation** (first ten minutes) | **Setup-friction failure** — the platform offloaded integration and credential wiring onto the user, so the first valuable result sat behind 30 minutes of yak-shaving instead of arriving in the first few minutes |
| Several hit a confusing wrong answer early, lost trust, went back to manual work | **Trust under failure** (first failure) | **Illegibility + no recovery path** — the user could not see what the agent did or why, and had no frictionless way to report it, so one wrong answer read as "this thing is unreliable" |
| One bad action was irreversible and caused a minor incident | **Trust under failure** (reversibility) | **Reversibility-ladder failure** — an early, low-confidence action was allowed to land on the irreversible tier with no gate, turning a recoverable wrong answer into an incident |

The symptoms are "30 minutes" and "bad action"; the *root causes* are setup
friction, illegibility-without-recovery, and an ungated irreversible tier.

### 2. Activation path (the first ten minutes)

```text
   minute 0   one-step install: `platform install <bundle>@<version>`
              → pulls the versioned bundle (agents/skills/tools/mcp/hooks)
              → MCP servers + credentials wired via the bundle, NOT by the user
   minute 1   sane defaults active: scoped tools, read-only first, gates on
   minute 2   curated first task that obviously pays:
                dev    → "summarize and draft a fix plan for this open item"
                on-call→ "triage this alert: what fired, what changed, suggest next step"
                support→ "draft a reply to this ticket using the right KB policy"
                analyst→ "answer this scoped question from the warehouse"
   minute 5   the user has a real, legible, reversible result they trust
```

- **One-step install via the versioned bundle (exercise-01):** the bundle ships
  the MCP servers and credential wiring pre-configured, so the user does *not* hunt
  credentials — the single biggest activation root cause is removed by construction.
- **Curated first task that obviously pays:** not a blank prompt, but a pre-loaded
  task for the user's role that returns visible value on the first run.
- **Sane defaults that work out of the box:** scoped least-privilege tools,
  read-and-propose before any irreversible action, gates on by default — the user
  succeeds without configuring anything.

**Activation metric to track:** **time-to-first-valuable-result** — the median
minutes from install to a new user reaching a result they accept (drafted fix
accepted, triage acted on, reply sent, question answered). Target: minutes, not
half an hour.

**Vanity metric explicitly refused:** **total installs** (or "users who tried it
once"). Installs were already high — that is exactly the trap. We refuse to
optimize a number that the failure mode (try once, never return) already inflates.

### 3. Trust under failure

The three trust-preserving properties:

- **Legibility.** Every run shows *what the agent did and why*: the tools it
  called (with scoped targets), the context it used, and the proposed action
  before it lands. A wrong answer is then a *visible, inspectable* wrong answer —
  the user sees the flawed step, not an opaque verdict, and corrects it instead of
  abandoning the platform.
- **Reversibility by default.** Failures land on the *safe* side of the
  reversibility ladder: the default is read-and-propose (draft, plan, suggested
  transition), and anything on the irreversible tier (merge, deploy, refund,
  prod-restart, public post) is **gated** — the agent proposes, a human or policy
  disposes. A wrong answer costs a wasted draft, not an incident.
- **Blameless, frictionless report path that visibly leads somewhere.** One action
  ("report this run") captures the run and routes it into the feedback loop (task
  5). It is blameless (the user is reporting a *platform* bug, not confessing a
  mistake) and visibly effective (the user can see the report became a tracked
  eval case) — so reporting feels worth doing.

**Addressing the incident specifically.** The minor incident happened because an
early, low-confidence action was allowed to be irreversible. The fix is structural:
the **irreversible tier is gated by default** (exercise-02's ladder), so the *same*
wrong answer in the new design produces a *proposed* merge/deploy/refund that a
human declines — it costs trust once (a visibly wrong proposal) instead of causing
an incident. The platform makes "wrong answer" and "irreversible action" two
events that cannot coincide without a human in between.

### 4. Documentation that pays twice

Two conventions authored as **skills**, each serving a human *and* the agent:

| Skill | Serves the human as… | Serves the agent as… |
| --- | --- | --- |
| **`how-we-handle-an-irreversible-action`** | onboarding doc: "here is our gate policy — propose, never auto-execute merge/deploy/refund/prod-write" | loaded behavior: the agent reads it and *defaults to proposing*, gating the irreversible tier |
| **`how-we-write-an-external-response`** (a customer reply, a PR description, an incident note) | convention doc: tone, required sections, what to cite | loaded behavior: the agent drafts in the house style and structure every time |

Because a skill is *both* the human-readable convention and the model-loaded
behavior, the onboarding doc and the agent's behavior are **the same artifact** —
update the skill and both move together.

**Where they live and why.** Both skills live in the **versioned bundle**
(exercise-01). That is what keeps doc and behavior from drifting: there is no
separate wiki that rots out of sync with what the agent actually does. The doc *is*
the behavior, versioned as one unit, shipped by the one-step install.

### 5. Versioning policy and feedback loop

**Versioning policy:**

- **Versioned bundle.** The platform ships as `bundle@version`; users pin a
  version and adopt new ones deliberately.
- **No silent behavior changes — especially on the privileged/irreversible tier.**
  Any change that widens tool scope, loosens a gate, or alters irreversible-tier
  behavior is a **major** version bump with a changelog entry and a review, never a
  silent push. The privileged tier never changes under a user without notice.
- **Reversible rollout.** New versions roll out behind a flag/canary and can be
  rolled back instantly; a bad release is recoverable without re-installing.

**Feedback loop (report → regression eval case → eval gate):**

```text
   bad run reported (1 frictionless action)
     ─▶ captured {inputs, context, tools called, bad output}
        ─▶ distilled to a permanent regression eval case
           ─▶ added to the eval suite
              ─▶ EVERY platform change (model / API / skill / tool / bundle) GATED ◀┐
                 on the full eval suite passing ─────────────────────────────────────┘
```

**Why this makes the platform improve under model/API change instead of
regressing.** Without the loop, a model upgrade or API change can silently
reintroduce a bug a user already hit — the platform regresses and re-loses the
same trust. With the loop, that bug is a *permanent* eval case: the next model
update is **gated** on it, so the regression is caught before release. The eval
suite grows monotonically with every reported failure, so the platform's worst
behaviors only ever get *fixed and pinned*, never silently un-fixed. Adoption
compounds because trust, once earned on a failure mode, is not given back.

### Stretch: dashboards, non-developer users, and widened scope as one event

**Activation + trust dashboards (real signals only).** Instrument
**time-to-first-valuable-result** (activation) and **return-after-failure rate**
(the share of users who hit a wrong answer and *still came back*). Keep
**total installs**, **total runs**, and **tokens consumed** *off* these dashboards
— they are vanity metrics that move without adoption improving.

**Non-developer population (on-call, support, analysts).** The two moments are
unchanged, but the **first ten minutes often happens under pressure** — an on-call
user's first use is *during* an incident, not in a calm tutorial. So the curated
first task must be the *real* task (triage *this* live alert), the result must be
**legible and reversible by default** because a stressed user has no patience for
an opaque or risky tool, and the report path must be one action because no one
files a careful bug report mid-incident. Activation under pressure raises, not
lowers, the bar on reversibility and legibility.

**Widened tool scope as one reviewed event.** When someone proposes widening a
tool's scope, it flows through *both* paths as a single reviewed change: the
**security-review path** (does the wider scope cross a trust boundary, does it add
an irreversible action that needs a gate?) and the **versioning path** (it is a
privileged-tier change → major bump, changelog, no silent push). One pull request,
two required reviewers (security + platform), one version bump — scope never widens
quietly.

## Meeting the acceptance criteria

- **Each pattern mapped to activation or trust-under-failure with a root cause,
  not a symptom.** Task 1's table maps the 30-minute setup to activation
  (setup-friction root cause), the wrong-answer-abandonment to trust
  (illegibility + no recovery), and the incident to the reversibility-ladder
  failure.
- **Activation path is one-step-install + curated-first-task + sane-defaults, with
  a real metric and a refused vanity metric.** Task 2 specifies bundle install,
  role-appropriate curated first tasks, and safe defaults; names
  time-to-first-valuable-result; and explicitly refuses total installs.
- **Trust-under-failure covers legibility, reversibility-by-default, and a
  frictionless report path, and prevents the incident.** Task 3 covers all three
  and shows the gated irreversible tier converting the incident-causing action into
  a declined proposal.
- **At least two conventions are skills that pay twice, in the versioned bundle.**
  Task 4 authors the irreversible-action gate convention and the external-response
  convention as skills serving human + agent, kept in the bundle so doc and
  behavior cannot drift.
- **Versioning forbids silent privileged-tier changes; feedback loop turns every
  bad run into an eval-gated regression case.** Task 5 makes privileged-tier
  changes a major bump and routes every reported bad run into a permanent eval case
  that gates every future change.

## Common pitfalls

- **Fixing the symptom, not the root cause.** "Add a setup wizard" treats the
  30-minute symptom; the root cause is that setup is the user's job at all — the
  bundle should pre-wire it.
- **Optimizing installs / total runs.** These are exactly the numbers a
  try-once-never-return failure inflates. The activation metric is
  time-to-first-*valuable*-result; the trust metric is return-after-failure.
- **Treating legibility as a log dump.** Legibility is the user seeing *what and
  why* before an action lands, not a wall of tokens after the fact.
- **Leaving the irreversible tier ungated "for power users."** That is the exact
  cause of the incident. Gate by default; widening is a reviewed, versioned event.
- **Authoring conventions as a separate wiki.** A wiki drifts from behavior.
  Author the convention as a skill in the bundle so the doc *is* the behavior.
- **A feedback loop with no gate.** Capturing bad runs into a doc no one tests
  against does nothing. The loop only pays when every reported run becomes an eval
  case that **gates** every future change.

## Verification

A completed submission is correct when:

- Each exit-interview pattern is mapped to **activation** or **trust under
  failure** with a stated **root cause** distinct from the symptom.
- The activation path is **one-step-install + curated-first-task + sane-defaults**,
  names a real **activation metric**, and **explicitly refuses** a named vanity
  metric.
- The trust-under-failure design covers **legibility, reversibility-by-default,
  and a frictionless report path**, and shows specifically how the gated
  irreversible tier prevents the incident from recurring.
- **At least two** conventions are authored as **skills** that serve human and
  agent, kept in the **versioned bundle**.
- The versioning policy **forbids silent behavior changes on the privileged tier**,
  and the **feedback loop turns every reported bad run into an eval-gated
  regression case**.
- `NOTES.md` answers the three prompts: which fix moves adoption most in month one
  (activation gets users *to* a first result; trust keeps them after the first
  failure — defend the pick), one bad run walked from report to permanent eval case
  with what the *next* model update would have done without the gate (silently
  regressed), and which roadmap feature to cut to fund the DX work.
