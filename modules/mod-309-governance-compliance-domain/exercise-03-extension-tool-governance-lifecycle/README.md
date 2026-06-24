# mod-309-governance-compliance-domain/exercise-03-extension-tool-governance-lifecycle — Solution

## Approach

The deliverable is a **governance lifecycle** — evaluate → version → approve → monitor,
with a fast revoke path — for an extension surface that today has none: tools are added
by merging a PR, with no evaluation, no version pinning, no monitoring, and no way to
pull a bad tool short of a redeploy. The design has to fix that for *both* third-party
tools (opaque, you do not own the code) and internal tools (you own the code), and it
has to treat **every tool as a data-flow/egress boundary** — a tool is where the agent's
context can leave the trust boundary.

The choices that drive the lifecycle below:

- **Every tool is a boundary, so least privilege is the spine.** The rubric leads with
  permissions/scope and data flow because a tool's danger is what it can *reach* and
  where it can *send*, not its nominal purpose. The "send-message" tool that can message
  arbitrary recipients is the canonical failure: unbounded egress.
- **The registry is the single control point.** Version pinning, the immutable audit
  record, and — critically — *revocation* all live in the registry. Because the agent
  resolves `tool@x.y.z` through the registry at call time, pulling a tool from the
  registry removes it from the reachable set *immediately*, with no redeploy.
- **A version is a new tool.** `summarize-record` silently regressed after an upstream
  model change precisely because a version bump was treated as transparent. Under this
  policy a new version **re-enters Evaluate** and cannot be called until re-approved, so
  the eval harness catches the drift *before* it ships.
- **Approval is a named-owner decision, risk-tiered.** A read-only utility and a
  money-moving third-party tool do not get the same gate. Approval is scoped, expiring,
  conditional, and recorded against a named owner — never an automatic merge.

## Reference solution

### 1. Evaluation rubric (five dimensions, evidence differentiated)

| Dimension | What it checks | Evidence — third-party (opaque) | Evidence — internal (code you own) |
| --- | --- | --- | --- |
| Permissions / scope (least privilege) | The narrowest set of capabilities, data classes, and fleets the tool needs. | Declared scope in the manifest; tested empirically against the declaration (does it try more than it claims?). | Same declaration, **verified by reading the code**: which APIs/credentials it actually touches. |
| Data flow (privacy/residency boundary) | What context it receives and where it sends it; whether regulated data crosses a residency boundary. | Vendor declaration of inputs/egress + network observation; **fails** if a regulated class would cross a disallowed region. | Same rule, verified by reading the code + tests; egress endpoints enumerated from source. |
| Security / provenance | Where the tool came from, that it is what it claims, supply-chain integrity. | Signature / publisher verification, provenance hash, dependency manifest (you cannot read the code, so provenance carries more weight). | Commit provenance, internal build attestation, dependency scan on your own tree. |
| Quality / correctness | Does it do its job acceptably against a baseline? | Black-box eval suite (mod-304): inputs → expected outputs, scored; this becomes the drift baseline. | Same eval suite **plus** unit/integration tests you can read and run. |
| Reversibility / consequence | How bad is a wrong call; can it be undone? | Declared side effects + tested: read-only vs. writes vs. irreversible external effect. | Side effects read directly from the code; irreversible effects flagged. |

The asymmetry is the point: for third-party tools you cannot read the code, so
**provenance and empirical/black-box evidence carry the weight**; for internal tools you
substitute code review and tests for the parts you would otherwise have to infer. The
*rule* is identical across both; only the *evidence source* differs.

### 2. Versioning and registry policy

**Exact-version pinning.** The agent calls `tool@x.y.z` — **never `@latest`**. A floating
tag would let an upstream change alter behavior under a stale approval (exactly the
`summarize-record` failure). Resolution goes through the registry, so the agent can only
reach versions the registry admits.

**A new version re-enters Evaluate.** `tool@1.2.0 → tool@1.3.0` is treated as a *new
artifact*: it must pass the rubric and be re-approved before any agent can pin it. The
old version remains callable under its existing approval until retired; the new one is
not callable until it earns its own.

**Immutable registry record** (per admitted version; append-only):

```text
RegistryRecord {
  tool_id, version (x.y.z),
  scope:        { fleets[], purposes[], data_classes[], regime_profiles[] },
  approver:     named role/identity,
  approved_at:  timestamp,
  expiry:       re-review deadline,
  provenance:   { source, signature, provenance_hash, dependency_manifest },
  eval_baseline_ref:  pointer to the quality baseline from Evaluate,
  status:       active | quarantined | revoked
}
```

It doubles as an **auditability artifact**: because the record is append-only and
timestamped, the registry answers *"which tools (and which versions, at which scope)
could this agent call on date D?"* by reading the records active at D. This record is
also what feeds the lineage spec of exercise-04 (the tool versions in a run's provenance
chain come from here).

### 3. Approval contract (named-owner, scoped, expiring, risk-tiered)

Approval is **not** an automatic merge. It is a decision recorded against a named owner,
scoped and time-bounded, with rigor keyed to the risk tier.

| Field | Definition |
| --- | --- |
| allowed scope | Which agents/fleets, purposes, data classes, and regime profiles the tool may be called within. |
| approver | The named role/identity who made the decision, recorded in the registry. |
| conditions | Required guardrails / sandbox tier / regime profile that must hold for the approval to be valid. |
| expiry | The re-review deadline; past it, the tool is not callable until re-approved. |
| risk tier | The rigor level of the gate (see below). |

**Risk-tiered rigor** — a read-only utility ≠ a money-moving third-party tool:

```text
   TIER 1  read-only / no egress     -> lightweight review, single approver
   TIER 2  writes internal state     -> standard review + eval baseline + 1 owner
   TIER 3  external egress / irreversible / money-moving / third-party
           -> full rubric, 2 distinct approvers, mandatory sandbox + monitoring,
              shorter expiry
```

The same gate template applies to all tools; the *tier* sets how many approvers, how
much evidence, how short the expiry, and whether a sandbox is mandatory.

### 4. Monitoring and revocation

**Runtime scope enforcement.** The approved scope is enforced at *call* time, not just
at admission: when the agent invokes `tool@x.y.z`, the runtime checks the call's fleet/
purpose/data-class/region against the registry's scope record and **blocks** an
out-of-scope call (e.g., a tool approved for fleet A invoked from fleet B, or a regulated
data class it was not approved for). Approval without runtime enforcement is a
suggestion, not a control.

**Quality-drift detection.** Each call's behavior is compared against the
`eval_baseline_ref` captured at Evaluate, via the eval harness (mod-304) and
observability (mod-305). A statistically significant regression against the baseline
raises a drift alert — this is precisely what would have caught `summarize-record`
degrading after the upstream model change.

**Abuse / anomaly detection.** Call-rate spikes, unusual recipients/egress targets, and
out-of-distribution arguments raise anomaly alerts (the "send-message to thousands of
arbitrary recipients" pattern).

**Revocation / quarantine — immediate, no redeploy.** The control point is the
**registry**:

```text
   revoke(tool_id, version):
     registry.set_status(tool_id, version, REVOKED)   # append-only event
   # because the agent resolves tool@x.y.z THROUGH the registry at call time,
   # the very next resolution fails -> the tool is out of the reachable set
   # NOW, with no deploy, no pod roll, no restart.

   quarantine(tool_id, version):
     registry.set_status(... QUARANTINED)  # callable only in sandbox / blocked
```

Flipping the registry status is the whole revocation: no code ships, no fleet restarts.
This is why the registry — not the deploy pipeline — is the revocation control point.

### 5. Two extensions walked through the pipeline

**Third-party "send-message" tool** (can message arbitrary recipients — Tier 3):

```text
   EVALUATE
     scope:    requests "message anyone" -> REDUCE to an allow-listed recipient
               domain + a per-run rate cap (least privilege)
     data flow: declares it sends message body externally -> body must be
               minimized; regulated classes blocked from the body (boundary!)
     provenance: verify signature + provenance_hash; scan dependency manifest
     quality:  black-box eval suite -> becomes the drift baseline
     consequence: external egress, hard to recall -> Tier 3
   VERSION
     pin send-message@2.1.0 ; record provenance_hash in registry
   APPROVE
     Tier 3 -> 2 distinct approvers; scope = {fleet: support, purpose:
     notify-known-contact, data_classes: [internal-only], regions: [allowed]};
     conditions = mandatory sandbox + egress allow-list; expiry = 90 days
   MONITOR
     runtime: block any call to a non-allow-listed recipient or with a
              regulated class in the body
     anomaly: alert on recipient-count spike or rate-cap breach
     revoke:  on abuse, registry.set_status(REVOKED) -> tool gone immediately
```

The bounded approval converts a "message anyone" tool into "message a known contact in
the support fleet, internal data only, rate-capped, sandboxed, 90-day expiry, instantly
revocable."

**Internal "summarize-record" tool, version bump after the model change** (Tier 2):

```text
   trigger: upstream model change -> bump summarize-record@1.2.0 -> 1.3.0
   VERSION-BUMP RE-ENTERS EVALUATE (a new version is a new tool):
     quality dimension runs the SAME eval suite that produced the 1.2.0 baseline
     -> 1.3.0 scores BELOW baseline (the silent regression) -> Evaluate FAILS
   APPROVE
     blocked: the gate never opens because Evaluate did not pass; 1.3.0 cannot
     be pinned; agents keep calling the still-approved 1.2.0
   OUTCOME
     the drift is caught BEFORE approval, not in production -> the regression
     never reaches the fleet. (Contrast today: a silent merge shipped 1.3.0 and
     the regression ran live until someone noticed.)
```

This is the entire reason a version bump re-enters Evaluate: the eval harness sees the
drift against the baseline *before* the new version is callable.

### Stretch: sandbox tiers, supply chain, deploy-version coupling

- **Sandboxing tiers keyed to risk tier.** Tier 1: in-process, no network. Tier 2:
  scoped credentials, no outbound egress beyond declared internal endpoints. Tier 3:
  network-egress allow-list enforced at the sandbox boundary, short-lived scoped
  credentials, CPU/memory/time resource limits, and per-call audit. The sandbox is a
  *condition* on the approval, so an out-of-sandbox call is an unapproved call.
- **Supply-chain check + CVE handling.** Evaluate verifies third-party signatures and
  records a dependency provenance manifest in the registry. When a dependency later gets
  a CVE, the registry's dependency manifests are queried to find *every* admitted version
  that pulls the vulnerable dependency, and those records are flipped to `quarantined`
  pending a patched re-evaluation — the same fast registry control point used for
  revocation.
- **Coupling to mod-308 deploy versioning.** When the agent *code* version changes
  (rainbow deploy), in-flight runs must keep the tool-version pins they started with: the
  run's lineage (exercise-04) captured `tool@x.y.z` at start, and the registry resolves
  by exact version, so a code roll does not silently swap a tool out from under an
  in-flight run. A tool revoked mid-run is the deliberate exception — revocation is
  allowed to interrupt even in-flight runs, because that is its purpose.

## Meeting the acceptance criteria

- **Rubric covers all five dimensions, evidence differentiated** — permissions, data
  flow, security/provenance, quality, reversibility, each with a third-party vs. internal
  evidence column.
- **Versioning pins exact versions, forces re-eval on change, immutable registry doubles
  as audit** — `tool@x.y.z` only, new version re-enters Evaluate, append-only registry
  record answers "which tools on date D?".
- **Approval is a named-owner, scoped, expiring, risk-tiered decision** — the contract
  table plus the three-tier rigor ladder; not an automatic merge.
- **Monitoring enforces scope at runtime, detects drift vs. baseline; revocation
  immediate and deploy-free** — call-time scope checks, baseline drift detection,
  anomaly detection, and registry-status revocation that removes the tool on the next
  resolution.
- **Both worked extensions traverse the full pipeline; the internal one catches drift
  before re-approval** — send-message gets scope-reduced/pinned/bounded/revocable;
  summarize-record@1.3.0 fails Evaluate against the 1.2.0 baseline before approval.

## Common pitfalls

- **`@latest` pinning.** A floating version lets an upstream change alter behavior under
  a stale approval — the exact `summarize-record` regression. Pin `tool@x.y.z` and make a
  new version re-enter Evaluate.
- **Revocation through the deploy pipeline.** If pulling a bad tool requires a redeploy,
  the bad tool keeps running for the length of a release. Make the registry the control
  point so a status flip removes it on the next call.
- **Approving scope as declared.** A "send-message" tool that asks for "message anyone"
  must be *reduced*, not rubber-stamped. Least privilege is enforced at approval and
  again at call time.
- **Exempting internal tools from governance.** "We own the code" is not "we evaluated
  this version." Internal tools get the same rubric — the evidence just comes from code
  review instead of black-box probing.
- **Treating the tool as not-a-boundary.** Every tool is where context can leave the
  trust boundary. Skipping the data-flow dimension is how regulated data silently egresses
  through a tool call.

## Verification

A completed submission is correct when:

- The rubric scores permissions, data flow, security/provenance, quality, and
  reversibility, with a distinct evidence bar for third-party (opaque) vs. internal
  (owned) extensions.
- The versioning policy mandates exact-version pinning, makes a new version re-enter
  Evaluate, and defines an append-only registry record that answers "which tools could
  this agent call on date D?".
- The approval contract fills allowed-scope, approver, conditions, expiry, and risk
  tier, and ties rigor (approver count, evidence, sandbox, expiry) to the tier.
- Monitoring enforces scope at call time, detects quality drift against the Evaluate
  baseline, and the revocation/quarantine path flips registry status to remove the tool
  immediately with no redeploy.
- The send-message walk-through shows scope reduction + version pin + bounded approval +
  monitoring/revoke, and the summarize-record walk-through shows the version bump
  re-entering Evaluate and failing the baseline *before* approval.
- `NOTES.md` answers the three prompts: governing internal tools without killing velocity
  (and why full exemption is wrong), why approval does not transfer from v1.2 to a
  "minor" v1.3, and what the lifecycle requires of tools that an approved tool itself
  calls (transitivity).
