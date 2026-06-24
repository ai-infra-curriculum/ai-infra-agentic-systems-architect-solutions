# mod-309-governance-compliance-domain/exercise-04-k12-edtech-agentic-constraints — Solution

This is a reference safety-and-output-governance design for the **TutorMesh
tutoring agent** that serves minors: a bounded action space, a minor-safety
control stack, a tiered human-in-the-loop model, a hallucination-containment
pipeline, and a control-specification table. It is design, not code.

> **Not legal advice.** Crisis escalation, mandatory-reporting duties, and
> child-safety obligations are governed by law and district policy; the named
> human and escalation path here must be set with counsel and the district. This
> exercise is the architecture that makes those duties operable, not the legal
> determination of them.

## Approach

Privacy (exercise-03) is necessary and insufficient. This exercise governs what
the agent *says and does* to a child, and who is accountable for it. The design
moves through five layers, each producing an inspectable control:

1. **Bound the action space.** A child-facing agent gets a deliberately small tool
   set; dangerous capabilities are denied *by design*, not by prompt instruction.
2. **Stack minor-safety controls** on both the input and output sides, calibrated
   for a child audience, with crisis signals routed to a named human — not a
   canned refusal — and age-appropriate AI-interaction transparency.
3. **Tier HITL by consequence.** Ephemeral help can run with post-hoc monitoring;
   anything touching a grade, record, IEP/504, discipline, or parent
   communication requires a human decision, logged. The review surface is
   designed against rubber-stamping or the loop becomes theater.
4. **Contain hallucination.** Factual claims are grounded in a curriculum KB,
   cited with a confidence signal, constrained to domain, output-verified against
   source, and biased toward "I don't know"; an ungrounded high-stakes assertion
   does not reach the student.
5. **Specify each control** in a one-page table: control, risk, pipeline location,
   owner, evidence source.

## Reference solution

### 1. Bounded action space

| Tool | Why it exists | Scope |
| --- | --- | --- |
| `answer_question` | Core tutoring; grounded explanation | Curriculum domain only; cited |
| `generate_practice_problem` | Practice; low stakes | Curriculum-aligned; sampled review |
| `give_feedback_on_work` | Formative feedback | Non-graded; HITL before it informs any grade |
| `draft_progress_note` | Drafts a note a teacher may forward | Draft only; blocked from sending without HITL |
| `retrieve_curriculum` | Grounding source for factual claims | Read-only, vetted KB |

**Denied by design** (absent capabilities, not discouraged ones):

- Open web browsing — can surface harmful or ungrounded content to a child.
- Contacting a child outside the supervised channel — no email/DM/external reach.
- Mutating a permanent record (grade, IEP, discipline) without a human — writes
  to records are gated, never autonomous.
- External egress of student-derived text except through a governed plugin
  (exercise-02) under a DPA.

### 2. Minor-safety control stack

**Input side (before the model sees it):** age-appropriate content filter;
detectors for self-harm/crisis signals, grooming/exploitation indicators, and
bullying. **Output side (before the child sees it):** the same categories plus
age-inappropriate-content filtering on generated text.

**Crisis-escalation path** (a self-harm signal must reach a human fast, not a
canned refusal):

```text
self-harm / crisis signal detected (input or output side)
   │
   ▼
suppress the normal agent response; show a safe, supportive holding message
   │
   ▼
ROUTE to a NAMED human per district policy (e.g., the supervising teacher and
the district's designated counselor/safety contact), with the context, fast
   │
   ▼
log the escalation (signal, time, who was notified, response) in the audit record
```

The named human and channel are configured per district (this is where district
policy and any mandatory-reporting duty attach). The control is "route to a named
human, fast," not "refuse and move on."

**Interaction transparency.** The child sees an age-appropriate, persistent
indicator that they are talking to an AI ("I'm a tutoring helper, not a person");
the supervising adult sees a clearer disclosure. Transparency is rendered, not
buried in terms of service.

### 3. Tiered human-in-the-loop

| Output type | Consequence | HITL tier | Gate behavior |
| --- | --- | --- | --- |
| Ephemeral hint / rephrasing | Low | Post-hoc monitoring | Runs; sampled in monitoring |
| Generated practice problem | Low–Medium | Sampled review | Runs; periodic educator sampling |
| Feedback on submitted work | Medium | Pre-use review when it will inform a grade | Educator reviews before it feeds assessment |
| Draft progress note to a parent | High | Blocking gate before send | Educator approves/edits/rejects; logged |
| Affects a grade, record, IEP/504, discipline | Critical | Mandatory human decision | No autonomous path; human decides; logged |

**High-stakes gate surface (designed against rubber-stamping).** For a draft
progress note, the educator sees, in one view: the **claim** (the draft text), the
**sources** each factual statement traces to, and a **confidence signal** per
claim, with low-confidence claims highlighted. The educator can approve, edit
inline, or reject with one action. Their **identity and decision** (approve /
edit-with-diff / reject) and a timestamp land in the same audit record as the
send. Anti-rubber-stamp moves: surface only what needs judgment (highlight
low-confidence and high-stakes spans rather than presenting a wall of text), make
the source one click away, and track per-educator approve-without-edit rates as a
review-quality signal so a 100%-rubber-stamp pattern is visible.

### 4. Hallucination containment

```text
  student input
      │
      ▼
  content-safety + age filter (in) ──▶ crisis signal? ──▶ escalate to named human
      │ ok
      ▼
  grounded retrieval (curriculum KB)      ← factual claims must trace to a source
      │
      ▼
  in-domain check  (math agent + history question? -> defer/escalate)
      │
      ▼
  output check: grounded? confident? in-domain?
      │                              │
   yes & low-stakes           no  OR  high-stakes
      │                              │
      ▼                              ▼
  to student (cited,            HITL gate: educator approves/edits/rejects;
  confidence shown)             identity + decision -> audit log
                                     │  OR
                                     ▼
                                "I don't know — let's ask your teacher"
                                (bias toward deferral on high-stakes factual)
```

- **Grounded retrieval** for factual claims: the agent draws facts from the vetted
  curriculum KB rather than free-generating; the architecture makes it hard to
  assert a fact not in an approved source.
- **Cite + confidence** on student-facing factual claims, so the educator can
  verify and the child learns to check sources.
- **In-domain constraint:** an out-of-domain question (math agent asked a history
  question) defers or escalates rather than improvising.
- **Output-side verification:** a claim-vs-source check flags ungrounded factual
  assertions before they reach the student.
- **The gate:** an ungrounded *high-stakes* factual assertion does **not** reach
  the student — it defers ("I don't know") or routes to the HITL gate. The safe
  default for a child-facing agent is admitting uncertainty.

### 5. The control specification

| Control | Risk addressed | Pipeline location | Owner | Evidence |
| --- | --- | --- | --- | --- |
| Bounded tool set + denied capabilities | Excessive agency; harmful reach to a child | Agent definition / registry | Tutor product owner | Registry tool grants; denied-capability list |
| Input content-safety + age filter | Harmful/age-inappropriate input | Pipeline entry (in) | Safety lead | Filter config; flagged-input logs |
| Crisis-signal detection + named-human escalation | Self-harm / crisis mishandled | In and out sides | Safety lead | Escalation runbook; crisis-routing logs + latency |
| AI-interaction transparency | Child unaware they talk to AI | Rendered UI surface | Tutor product owner | Disclosure copy + render config |
| HITL gate (high/critical tiers) | Automated consequential decision about a child | Before send / record write | Tutor product owner | Approval logs with human identity + decision |
| Anti-rubber-stamp review surface | Rubber-stamped HITL (theater) | HITL gate UI | Tutor product owner | Approve-without-edit rate metric per educator |
| Grounded retrieval (curriculum KB) | Confabulation in instruction | Mid-pipeline retrieval | Curriculum/eval lead | Retrieval logs; KB version |
| Cite + confidence on claims | Ungrounded claim believed by child | Output assembly | Curriculum/eval lead | Per-claim source + confidence in output record |
| In-domain constraint | Out-of-domain improvisation | Pre-output check | Curriculum/eval lead | Domain-classifier config; defer logs |
| Output claim-vs-source verification + high-stakes gate | Ungrounded high-stakes assertion reaches student | Output check | Quality/eval lead | Verification logs; gated-assertion count |
| Bias-toward-"I don't know" | Confidently wrong instruction | Output decision | Tutor product owner | Deferral-rate metric; defer logs |

## Meeting the acceptance criteria

- **Deliberately small action space with denied capabilities justified.** Five
  scoped tools; open browsing, out-of-channel contact, and unsupervised record
  mutation are denied by design with reasons.
- **Minor-safety stack with crisis escalation to a named human (not a refusal) and
  age-appropriate transparency.** Input/output filters plus a crisis path that
  suppresses the normal response and routes to a named human fast, and a rendered
  AI-interaction disclosure.
- **HITL tiered by consequence; identity + decision logged; surface designed
  against rubber-stamping.** Five-tier table puts grades/records/IEP/discipline/
  parent comms behind a human decision; the gate surface shows claim+source+
  confidence and tracks approve-without-edit rates.
- **Hallucination containment grounds, cites + scores, constrains to domain, gates
  ungrounded high-stakes assertions, biases toward deferral.** The pipeline
  diagram and the spec table cover all five.
- **Control-spec table ties each control to a risk, a pipeline location, an owner,
  and an evidence source.** Eleven rows, each with all five columns.

## Common pitfalls

- **Treating a canned refusal as crisis handling.** A self-harm signal answered
  with "I can't help with that" abandons the child. The control is routing to a
  named human, fast, with the escalation logged.
- **HITL theater.** A gate a teacher cannot exercise at classroom scale gets
  rubber-stamped. Without claim+source+confidence on the surface and an
  approve-without-edit metric, you have automation with extra steps.
- **Denying capabilities by prompt instead of by design.** "The agent is told not
  to browse the web" is not a control; the capability must be *absent* from the
  tool set so it cannot be jailbroken into existence.
- **Grounding the easy claims but not gating the hard ones.** Retrieval + citation
  without an output-side gate still lets an ungrounded high-stakes assertion
  through. The gate (defer or escalate) is the load-bearing part.
- **Punishing deferral.** If "I don't know" is treated as a failure metric, the
  system learns to guess. For a child-facing agent, a deferral that costs a
  correct answer is the right trade against implanting a confident falsehood.

## Verification

- **Action-space audit.** Confirm the tool set is minimal and the denied
  capabilities are absent from the registry, not merely discouraged in the prompt.
- **Crisis path.** Confirm a self-harm signal routes to a *named* human with the
  escalation logged and a latency metric, not to a refusal.
- **HITL tiering.** Confirm every grade/record/IEP/discipline/parent-comms output
  sits behind a human decision with identity + decision logged, and that an
  anti-rubber-stamp metric exists.
- **Containment gate.** Trace an ungrounded high-stakes claim through the pipeline
  and confirm it defers or escalates rather than reaching the student.
- **Spec completeness.** Confirm every control-spec row has a risk, a pipeline
  location, an owner, and an evidence source.
