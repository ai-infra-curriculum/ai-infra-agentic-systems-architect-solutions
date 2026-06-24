# mod-301-agentic-systems-foundations/exercise-03-tool-vs-subagent-decomposition — Solution

## Approach

The deliverable is a *capability map plus ADRs plus a reference architecture*, not
an implementation. The whole exercise turns on one Chapter 3 rule applied capability
by capability: **hardcode by default, escalate to a tool only where the model must
decide whether/how, and to a sub-agent only where a separate context window is
genuinely justified (isolation, specialization, or parallel decomposition).**

Method for the reference answer:

- **Enumerate granularly first.** "Look up order" and "search the knowledge base"
  are different capabilities with different boundaries; collapsing them hides
  decisions. The reference map has twelve capabilities.
- **For every capability, record *who decides* and *what context it costs*** — the
  two columns from Chapter 3's comparison table. That discipline is what forces the
  boundary out of vibes and onto a defensible axis.
- **Reserve exactly one sub-agent.** Only the cross-system investigation earns a
  separate window (large context to read, distilled result to return). Everything
  else is a hardcoded path or a tool. This is the start-simple bias made visible:
  one sub-agent, not five.
- **Write ADRs only for the genuinely contestable calls** — the three boundaries a
  reviewer would actually argue: investigation (tool vs. sub-agent), classification
  (hardcoded router vs. tool), and email send (hardcoded trigger vs. model tool).
- **Plant and correct one anti-pattern** — knowledge-base search modeled as a
  sub-agent, corrected down to a tool.

## Reference solution

### Capability map

```text
CAPABILITY MAP — support-automation platform

  Capability                       Boundary       Who decides         Context cost       Justification
  -------------------------------  -------------  ------------------  -----------------  ----------------------------
  1  Authenticate requester        hardcoded      engineer            none               security-critical, fixed path
  2  Rate-limit / abuse check      hardcoded      engineer            none               deterministic guard, auditable
  3  Validate / parse inbound msg  hardcoded      engineer            none               schema check before any model
  4  Classify issue category       hardcoded      engineer (router)   small (label)      enumerable categories -> route
                                    router                                               (see ADR-001)
  5  Look up order                 tool           model: whether      small result       bounded lookup, compact return
  6  Look up account/customer      tool           model: whether      small result       bounded lookup, compact return
  7  Search knowledge base         tool           model: whether/qry  medium (snippets)  bounded retrieval (see ADR-003
                                                                                          anti-pattern correction)
  8  Cross-system investigation    sub-agent      parent: spawn       separate window;   reads logs+docs+tickets; only
                                                                       distilled return   distilled result to parent
                                                                                          (see ADR-002)
  9  Draft resolution text         tool           model: when ready   inline             single generation in context
  10 Check refund threshold        hardcoded      engineer            none               deterministic numeric gate
  11 Human approval (refund)       hardcoded      human (gated)       none               approval is a gate, not a call
     gate
  12 Send resolution email         hardcoded      engineer (post-     none               the ACT is code; the DECISION
                                    trigger        gate)                                 to send is gated (see ADR-002)
```

Note the shape: the model's judgment is fenced into capabilities 5–9 (decide
whether to look up, what to search, when a draft is ready), and everything around
them — auth, validation, threshold checks, the approval gate, the send trigger —
is deterministic code. That is Chapter 3's "fence the judgment, surround it with
code you can test and replay" applied directly.

### Reference architecture

```text
REFERENCE ARCHITECTURE — control + data flow

  inbound message
        |
        v
  [ 1 authenticate ] -> [ 2 rate-limit ] -> [ 3 validate/parse ]   <-- HARDCODED
        |                                                              (deterministic
        v                                                               fence)
  [ 4 classify -> route ]   <-- HARDCODED ROUTER (enumerable categories)
        |
        +--- simple case ---> ===== MODEL JUDGMENT FENCED IN =====
        |                      main agent context:
        |                        - tool 5 look-up-order
        |                        - tool 6 look-up-account
        |                        - tool 7 search-knowledge-base
        |                        - tool 9 draft-resolution
        |                      ==========================================
        |                                |
        +--- hard case -------> [ 8 SUB-AGENT: cross-system investigation ]
                                   own context window: reads logs, docs,
                                   prior tickets; returns DISTILLED findings
                                |
                                v   (distilled result re-enters main context)
                        [ 9 draft resolution (tool) ]
                                |
                                v
                        [ 10 refund threshold check ]   <-- HARDCODED gate
                                |
                  under threshold |  over threshold
                                |  v
                                |  [ 11 human approval gate ]   <-- HARDCODED
                                |  |
                                +--+
                                v
                        [ 12 send email ]   <-- HARDCODED trigger

  Legend: HARDCODED = deterministic code (no model decision).
          tool      = model decides whether/how to call; result returns inline.
          SUB-AGENT = separate loop + own context window; distilled return.
```

### ADR-001 — Classification: hardcoded router vs. tool

```text
ADR-001: Issue classification is a hardcoded router, not a model-decided tool

  Status:        Accepted
  Context:       Every inbound message must be categorized (refund / technical /
                 billing / other) before handling. The category set is fixed and
                 known. Two homes compete: a hardcoded routing call over a fixed
                 label set, or a tool the main agent chooses to call.
  Options:       A) hardcoded router (cheap model or classifier -> known handler)
                 B) tool the agent may call when it "decides" classification helps
                 C) sub-agent for classification
  Decision:      A — hardcoded router.
  Rationale:     Who-decides: the route set is enumerable, so no runtime model
                 judgment over control flow is needed — Chapter 3 says hardcode it.
                 Context-cost: a label is tiny; a tool adds a non-deterministic
                 decision to a deterministic dispatch (the "tool-where-hardcoded-
                 would-do" anti-pattern). C is absurd overhead for a label.
  Consequences:  Gain: predictable, replayable, auditable routing; cheap. Accepted
                 failure mode: a brand-new category needs a code/route change
                 rather than emergent handling. Revisit if categories stop being
                 enumerable (then it becomes model-driven decomposition).
```

### ADR-002 — Cross-system investigation: tool vs. sub-agent

```text
ADR-002: Cross-system investigation is a sub-agent, not a fat tool

  Status:        Accepted
  Context:       Hard tickets require reading across logs, internal docs, and prior
                 tickets — potentially dozens of large documents — to find the
                 root cause. This raw material must not flood the main context.
  Options:       A) one big "investigate" tool that returns everything inline
                 B) a sub-agent with its own context window and a distilled return
                 C) hardcoded multi-source query pipeline
  Decision:      B — sub-agent.
  Rationale:     Context-cost: the defining justification for a sub-agent is
                 context isolation (Chapter 3). The investigation reads a large,
                 variable volume; a tool returning it inline would blow the main
                 context budget and degrade every subsequent step. The path is
                 also genuinely model-driven (it course-corrects on what each
                 lookup reveals) so C (hardcoded) cannot enumerate it.
  Consequences:  Gain: the parent sees only distilled findings; its context stays
                 clean. Accepted failure mode: distillation can drop a detail the
                 parent needed, and the extra loop adds latency + tokens. Revisit
                 (collapse to a tool) if investigations turn out to read a small,
                 bounded result set in practice.
```

### ADR-003 — Email send: hardcoded trigger vs. model-callable tool

```text
ADR-003: Sending the resolution email is a hardcoded trigger, not a model tool

  Status:        Accepted
  Context:       After a resolution is drafted (and, for refunds over threshold,
                 approved), the email must go out. The DECISION to send is gated;
                 the ACT of sending is mechanical. Should the model hold the send
                 capability as a tool it may call, or should send be code?
  Options:       A) "send_email" tool the agent calls when it judges it is done
                 B) hardcoded trigger fired by code after the approval/threshold
                    gates pass
                 C) sub-agent that owns the send
  Decision:      B — hardcoded trigger after the gates.
  Rationale:     Blast radius: sending is an irreversible external action. Letting
                 the model own the trigger means a bad trajectory can email a
                 customer the wrong thing or skip the approval gate. Chapter 3's
                 worked example is explicit: the model drafts the body; code sends.
                 Who-decides: the *decision to send* belongs to the threshold/human
                 gates (deterministic), not to the model's discretion.
  Consequences:  Gain: no model trajectory can send unapproved or unreviewed mail;
                 fully auditable. Accepted failure mode: the system cannot
                 "decide" to send out-of-band; every send goes through the fixed
                 gate. Revisit only if a low-stakes, fully-reversible send channel
                 appears where model discretion is cheap to get wrong.
```

### Anti-pattern found and corrected

```text
ANTI-PATTERN: sub-agent where a tool would do

  Tempting design: model "search the knowledge base" as a sub-agent, reasoning
  that retrieval + ranking + synthesis "feels like its own job."

  Why it is wrong (Chapter 3): KB search is a single bounded capability that
  returns a compact set of snippets. It needs no separate context window, no
  specialized prompt, and no parallel decomposition. A sub-agent here adds an
  entire extra loop — latency, tokens, and another trajectory that can fail — for
  work one tool call returns inline. This is the single most common
  over-engineering ("sub-agent-where-a-tool-would-do").

  Correction: capability 7 is a TOOL. The model decides whether to search and with
  what query; the snippets return inline to the same context. If the KB result set
  were genuinely huge (hundreds of long documents needing distillation), the
  calculus would flip toward a sub-agent — but that is the investigation
  capability (8), not routine KB lookup.
```

## Meeting the acceptance criteria

- **Every capability (10+) assigned a boundary with who-decides and context-cost**
  — the capability map has twelve rows, each with a boundary, a who-decides value,
  a context-cost value, and a justification.
- **Reference-architecture diagram shows boundaries and fenced-in judgment** — the
  diagram marks each capability's home and the `MODEL JUDGMENT FENCED IN` region
  surrounded by hardcoded auth, gates, and the send trigger.
- **3–4 ADRs document the contestable calls with alternatives and consequences** —
  ADR-001 (classification), ADR-002 (investigation), ADR-003 (email send), each
  with options A/B/C, a decision, and an accepted failure mode.
- **At least one anti-pattern identified and corrected** — KB search as a sub-agent,
  corrected to a tool, with the reasoning shown.
- **The decomposition reflects the start-simple bias** — exactly one sub-agent
  appears, justified by context isolation; everything else is hardcoded or a tool.

## Common pitfalls

- **Reaching for a sub-agent because work "feels like its own job."** A separate
  context window costs a whole extra loop; it only pays off for isolation,
  specialization, or parallel decomposition. KB search has none of those — it is a
  tool. The default is *down*, not up.
- **Making the email send a model tool.** Irreversible external actions belong
  behind a hardcoded trigger gated by deterministic checks. The model drafts; code
  sends. Giving the model the send tool widens the blast radius for nothing.
- **Promoting classification to a tool.** If the categories are enumerable, routing
  is a hardcoded dispatch. Wrapping it as a tool adds a non-deterministic decision
  to a deterministic process — the "tool-where-hardcoded-would-do" anti-pattern.
- **Coarse capability enumeration.** Lumping "look up order," "look up account,"
  and "search KB" into one "retrieve data" capability hides three separate boundary
  decisions. Granularity is what makes the map defensible.
- **Skipping who-decides / context-cost.** An ADR or map row that states the
  boundary but not who decides and what context it costs has no axis to defend on —
  those two columns *are* the Chapter 3 argument.

## Verification

A completed submission is correct when:

- The capability map has at least ten rows, and every row fills boundary,
  who-decides, context-cost, and justification (no blanks).
- At most one or two capabilities are sub-agents, and each one names its
  justification as isolation, specialization, or parallel decomposition — not "it
  felt like its own job."
- The architecture diagram shows deterministic code (auth, validation, threshold
  check, approval gate, send trigger) surrounding a clearly marked model-judgment
  region.
- Three to four ADRs exist for the contestable calls, each listing real
  alternatives and the failure mode being accepted (not just the chosen option).
- At least one anti-pattern is shown *and corrected*, with the corrected boundary
  appearing in the capability map (KB search as a tool, row 7).
- `NOTES.md` answers the three prompts: the hardest capability to place (here
  classification — router vs. tool, settled by enumerability), where you resisted a
  sub-agent (KB search) and what would change the call (a huge result set), and the
  input change that would reverse one ADR (e.g. categories ceasing to be
  enumerable flips ADR-001).
