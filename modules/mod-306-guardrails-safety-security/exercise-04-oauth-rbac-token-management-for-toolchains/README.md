# mod-306-guardrails-safety-security/exercise-04 (OAuth, RBAC, and Token Management for Toolchains) — Solution

## Approach

The workspace assistant reaches three real external systems — a CRM, a payments
provider, and a calendar — in two modes: **interactive** (employee present) and
**scheduled** (nightly digest, no human). The goal is to eliminate the
confused-deputy god credential so the agent's effective authority is the
**intersection** of three independent narrowings: RBAC ∩ OAuth scopes ∩ per-call
policy.

Steps:

1. Choose the OAuth flow per (integration × mode) and justify it; the nightly
   digest cannot use a user-present flow.
2. Design the token lifecycle: TTLs, storage, out-of-band refresh, the
   no-token-in-context rule, revocation, and rotation.
3. Add audience binding and per-tool downscoping so a CRM token cannot be replayed
   at the payments API.
4. Layer RBAC over the catalog (≥3 roles) and implement the per-call check that
   enforces the three-way intersection.
5. Trace a $200 refund end to end, showing where each control can deny.

Chapter 4's rule governs: the token an agent carries to a tool should encode
exactly the scopes that tool needs, for this user, for a short window — never a
broad, long-lived credential.

## Reference solution

### 1. Flow per integration × mode

```text
| integration | mode        | flow                          | why                                           |
|-------------|-------------|-------------------------------|-----------------------------------------------|
| CRM         | interactive | Auth Code + PKCE              | human present, consents; public client safe   |
| CRM         | scheduled   | Client Credentials (svc acct) | no human; service identity scoped crm.read    |
| Payments    | interactive | Auth Code + PKCE → Token Exch | consent, then downscope to {issue_refund,aud=pay} |
| Payments    | scheduled   | NOT GRANTED                   | nightly digest has no business issuing refunds |
| Calendar    | interactive | Auth Code + PKCE              | human present, consents to calendar scopes    |
| Calendar    | scheduled   | Client Credentials (svc acct) | no human; service identity scoped cal.read    |
```

The **nightly digest cannot use a user-present flow**, so it authenticates with
**Client Credentials** as a dedicated service identity scoped to the *minimum* its
job needs — `crm.read` and `calendar.read` only. It is deliberately *not* granted
payments access at all: an unattended agent has no legitimate reason to move money,
so the capability is removed rather than gated. This keeps the closest-thing-to-a-
god-credential (the service account) as tight as possible.

### 2. Token lifecycle

```text
| token type    | TTL       | stored where                  | who reads it              | refresh                 |
|---------------|-----------|-------------------------------|---------------------------|-------------------------|
| access        | 10 min    | tool middleware, per-call     | tool middleware only      | broker mints fresh      |
| refresh       | 30 days   | secret store (KMS-backed)     | credential broker only    | rotated on use          |
| svc-acct tok  | 10 min    | broker, per-call              | broker → tool middleware  | broker re-mints         |
```

Rules:

- **Short access TTL (10 min):** a leaked access token expires before most attacks
  complete.
- **Out-of-band refresh:** the refresh token lives in a KMS-backed secret store
  that the agent process cannot read into its context. A **credential broker**
  exchanges it for fresh access tokens. The model never sees the refresh token.
- **No token in context (the load-bearing rule):** access tokens are injected by
  the tool-calling middleware at call time, never placed in the model's prompt or
  reasoning context. A token in the context is one injection away from
  exfiltration.
- **Revocation:** every grant is revocable; the broker holds a revocation list and
  refuses to mint against a revoked grant, so a token can be killed mid-session.
- **Rotation:** refresh tokens rotate on every use (one-time-use refresh); rotation
  is also triggered on any suspicion signal (anomalous IP, failed audience check).

### 3. Audience binding and downscoping

```text
   employee ─consent(Auth Code + PKCE)─▶ Auth Server
                                            │  access(aud=crm, scopes=crm.read crm.write, TTL=10m)
                                            │  + refresh → secret store
                                            ▼
                                   credential broker  (model never sees refresh)
                                            │ token-exchange (RFC 8693): downscope
                                            ▼
                          access(aud=payments, scopes={issue_refund}, TTL=10m)
                                            │
              tool middleware ──injects per-call, audience-bound token──▶ payments API
```

The token minted for the CRM carries `aud=crm`; the payments API validates `aud`
and **rejects** any token whose audience is not `payments`. So a CRM token
(stolen, logged, or injected) cannot be replayed against the payments API. Each
tool gets a token downscoped (via Token Exchange) to exactly its scope and
audience — `issue_refund` gets `{scopes: issue_refund, aud: payments}`, nothing
broader.

### 4. RBAC over the catalog + effective-authority intersection

```python
"""RBAC over the tool catalog, composed with OAuth scopes and per-call policy.

Effective authority = RBAC ∩ OAuth scopes ∩ per-call policy. Three independent
narrowings; an attacker must defeat all three.
"""
from __future__ import annotations

from dataclasses import dataclass, field

ROLE_TOOLS = {
    "sales_rep": {"crm_read", "crm_write", "calendar_read", "calendar_write"},
    "finance": {"crm_read", "issue_refund"},
    "read_only": {"crm_read", "calendar_read"},
}

REFUND_CAP = 1000.0


@dataclass(frozen=True)
class Token:
    scopes: frozenset[str]
    audience: str


@dataclass(frozen=True)
class Call:
    tool: str
    arguments: dict


@dataclass(frozen=True)
class Ctx:
    roles: frozenset[str]
    token: Token
    call: Call
    known_contacts: frozenset[str] = field(default_factory=frozenset)


def _refund_policy(args: dict, ctx: Ctx) -> bool:
    amount = args.get("amount", 0)
    return isinstance(amount, (int, float)) and 0 < amount <= REFUND_CAP


TOOL_POLICY = {
    "crm_read": lambda a, c: True,
    "crm_write": lambda a, c: True,
    "calendar_read": lambda a, c: True,
    "calendar_write": lambda a, c: True,
    "issue_refund": _refund_policy,
}

# A token whose audience matches the tool's resource (downscoping check).
TOOL_AUDIENCE = {
    "crm_read": "crm", "crm_write": "crm",
    "calendar_read": "calendar", "calendar_write": "calendar",
    "issue_refund": "payments",
}


def rbac_allows(roles: frozenset[str], tool: str) -> bool:
    return any(tool in ROLE_TOOLS.get(r, set()) for r in roles)


def effective_allowed(ctx: Ctx) -> tuple[bool, str]:
    tool = ctx.call.tool
    if not rbac_allows(ctx.roles, tool):
        return False, "RBAC: role does not grant tool"
    if tool not in ctx.token.scopes:
        return False, "OAuth: token scope missing"
    if ctx.token.audience != TOOL_AUDIENCE.get(tool):
        return False, "OAuth: token audience mismatch"
    if not TOOL_POLICY[tool](ctx.call.arguments, ctx):
        return False, "policy: arguments outside envelope"
    return True, "RBAC ∩ scopes ∩ policy all pass"
```

The agent's effective authority is the intersection: the role must grant the tool,
the token must carry the scope *and* the matching audience, and the arguments must
pass the per-call policy. Any one failing denies the call.

### 5. End-to-end trace: "employee asks for a $200 refund"

```text
1. identity        → employee authenticated; roles = {finance}
                     CAN DENY: not authenticated / no session.
2. RBAC check      → issue_refund ∈ ROLE_TOOLS["finance"]?  yes
                     CAN DENY: a sales_rep here → RBAC denies (no issue_refund).
3. token acquire   → broker exchanges user's grant → access(aud=payments,
                     scopes={issue_refund}, TTL=10m)
                     CAN DENY: grant revoked / refresh expired → broker refuses.
4. per-call policy → amount=200 ≤ REFUND_CAP(1000)?  yes
                     CAN DENY: amount > cap or ≤ 0 → policy fails.
5. approval gate   → issue_refund is HIGH-tier → REQUIRE_APPROVAL,
                     "refund $200 to order X" shown to a human (ex-03 gate)
                     CAN DENY: human rejects.
6. API call        → middleware injects the aud=payments token; payments API
                     validates aud → executes
                     CAN DENY: audience mismatch at the API.
```

Every step is a place a control can deny. A leaked $200-refund token from step 6
is bounded: it expires in 10 minutes and is `aud=payments`-bound, so it cannot be
replayed at the CRM or calendar, and it carries only `issue_refund` scope.

### Stretch: step-up + integration with exercise-03

- **Step-up authorization:** `issue_refund` requires a *fresh, higher-assurance*
  consent (re-auth / MFA) even within an active session, so a long-running session
  cannot silently accumulate refund authority.
- **Token binding (DPoP / mTLS):** bind the access token to the agent's key so a
  stolen bearer token cannot be replayed from another client.
- **Approval composition:** `issue_refund` requires *both* RBAC `finance` *and* the
  exercise-03 human-approval gate — RBAC narrows who can propose, approval narrows
  what fires.

## Meeting the acceptance criteria

- **Each integration × mode has a justified flow; scheduled agent uses Client
  Credentials with a tight service identity, not a god key** — section 1; the
  nightly digest gets `crm.read` + `calendar.read` only and no payments access.
- **Token lifecycle specifies TTLs, out-of-band refresh, storage, revocation,
  rotation, no-token-in-context** — section 2.
- **Tokens audience-bound and downscoped per tool** — section 3, `aud` validation
  with Token Exchange.
- **RBAC for ≥3 roles; per-call check enforces RBAC ∩ scopes ∩ policy** — section 4,
  three roles and `effective_allowed`.
- **End-to-end refund trace shows every deny point** — section 5, six steps each
  with a "CAN DENY."

## Common pitfalls

- **The single admin API key.** Baking one powerful service credential into the
  deployment is the confused deputy — a low-privilege user can drive it via
  injection. Act with the user's downscoped authority.
- **Putting tokens in the model's context.** A token in the prompt is one injection
  away from exfiltration. Inject at call time in the middleware, never in context.
- **Long-lived access tokens.** A multi-day token means a leak is a multi-day
  breach. Keep access TTLs in minutes; refresh out-of-band.
- **Skipping audience binding.** Without `aud`, a CRM token replays at payments.
  Bind every token to one resource and validate it at the API.
- **Granting the scheduled agent payments scope "to be safe."** An unattended agent
  with refund authority and no human is a standing risk. Remove the capability, do
  not just gate it.

## Verification

- **Flow table:** every (integration × mode) cell has a flow and justification; the
  scheduled row uses Client Credentials with named minimal scopes; scheduled
  payments is explicitly not granted.
- **Lifecycle:** TTLs, storage, broker-mediated out-of-band refresh, revocation,
  rotation, and the no-token-in-context rule are all stated.
- **Downscoping:** confirm the diagram shows Token Exchange and `aud` validation at
  the payments API.
- **RBAC code:** exercise the intersection:

  ```python
  good = Ctx(roles=frozenset({"finance"}),
             token=Token(scopes=frozenset({"issue_refund"}), audience="payments"),
             call=Call("issue_refund", {"amount": 200}))
  assert effective_allowed(good)[0] is True
  # sales_rep cannot issue refunds — RBAC denies.
  bad_role = Ctx(roles=frozenset({"sales_rep"}),
                 token=Token(frozenset({"issue_refund"}), "payments"),
                 call=Call("issue_refund", {"amount": 200}))
  assert effective_allowed(bad_role)[0] is False
  # CRM token replayed at payments — audience mismatch denies.
  wrong_aud = Ctx(roles=frozenset({"finance"}),
                  token=Token(frozenset({"issue_refund"}), "crm"),
                  call=Call("issue_refund", {"amount": 200}))
  assert effective_allowed(wrong_aud)[0] is False
  # Over the cap — policy denies.
  over = Ctx(roles=frozenset({"finance"}),
             token=Token(frozenset({"issue_refund"}), "payments"),
             call=Call("issue_refund", {"amount": 5000}))
  assert effective_allowed(over)[0] is False
  print("all OAuth/RBAC checks passed")
  ```

- **Trace:** confirm all six steps and their deny points are present.
- **`NOTES.md`** answers: where the god-credential temptation appeared and how you
  removed it; the blast radius of a logged access token given your TTL + audience
  binding; how the nightly agent's lack of a human changes scope and approval.
