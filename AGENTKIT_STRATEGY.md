# AgentKit strategy and learnings

A decision log, not an API guide. The call-by-call mechanics live in
[`WORLD_INTEGRATION.md`](./WORLD_INTEGRATION.md); this file records *why* we wired AgentKit the
way we did, what we refused to build, and what a weekend against the beta actually taught us.

## The one-line thesis

Selfie Check proves *a* human is present. AgentKit proves *which* human is accountable. Neither
is sufficient alone, and the product is what you build on the pair — not either credential by
itself.

- A liveness selfie with no stable identity is a fresh anonymous face every time. Great for
  "someone is here", useless for "the same someone who is answerable for this".
- A stable identity with no liveness is a rubber stamp. It says a human once registered a wallet,
  not that a human is behind *this* signature *right now*.

So we treat them as two halves of one claim: **a specific, accountable human was live for this
exact diff, and they spent something finite to say so.**

## Why continuity forced our hand toward humanId

The decision that shaped everything else: Selfie Check nullifiers are scoped to `rp + action`,
and we mint a unique action per commit (`git-sign:<label>:<signal>`). That is deliberate —
`max_verifications: 1` plus one-time-use v4 nullifiers mean a fixed action burns out after a
single use per human, so a per-commit action is the only thing that scales.

The side effect is that the nullifier is **different and unlinkable on every single commit**.
Excellent for privacy, fatal for any feature that needs "is this the same person as last time".
Our entire value proposition — accountability, per-human rate limits — needs exactly that
continuity.

AgentKit's `humanId` is the escape hatch. It is stable and anonymous: the same value on every
request from the same underlying human, revealing nothing about who they are. So we split the two
credentials by job and never conflate them:

| Signal | Source | Scope | What it is for |
|---|---|---|---|
| `nullifier` | Selfie Check | per commit (rp + action) | freshness — proves *this* diff got a live face |
| `humanId` | AgentKit / AgentBook | stable per human | continuity — accountability and rate limiting |

If we had only discovered this after designing the schema around nullifiers, it would have been a
rewrite. Worth stating loudly for anyone else building on Selfie Check.

## Budget over gate: the piece we care most about

A binary "human present? yes/no" gate is theatre if it can be spammed. One tired human can wave
through a thousand agent commits at 3am, and every one carries a real, fresh proof. The gate says
nothing about whether approving was a *decision*.

So the primitive is not a gate, it is a **budget**. Each human gets a finite daily allowance of
approvals. Because the counter is keyed on `humanId` and nothing else, the limit follows the
person across every machine, key, and agent they operate. Rubber-stamping stops being something
you shrug at and becomes something you *spend*.

That single design choice — count per human, not per key — is what turns a presence check into an
accountability primitive. It is the reason we needed AgentKit at all rather than just IDKit.

## What we deliberately refused to build

AgentKit ships access modes — `free`, `free-trial`, `discount` — that hand agents cheaper or
gated access once a human backs them. We use **none of them**, on purpose. Granting human-backed
benefits to AI agents (API calls, discounts, and the like) is the explicitly disqualified pattern
for this track. We use AgentKit for identity resolution only: attribute and rate-limit, never
reward.

Also off the table, for the same reason:

- **Agent reputation / trust scores.** Anything that renders as a score next to an agent name is
  the wrong shape. Identity is not merit; a resolved `humanId` says *who is answerable*, not *who
  is good*.
- **Any linkage that de-anonymises.** We never store the mapping between `humanId` and a real
  identity, because we never learn it. The whole system runs on a nullifier and a hash. There is
  nothing to leak because there is nothing to store.

Stating the negatives matters as much as the features: the restraint is the design.

## Enforcement modes, and why the default is `warn`

The AgentKit check runs in one of three modes (`WORLD_AGENTKIT`):

- `enforce` — reject any caller that does not resolve to a registered human. Correct for a real
  deployment and what we use for the demo.
- `warn` — run the check, log the outcome, but let the request through. **This is the default.**
- `off` — skip it entirely.

`warn` is the default deliberately. The `ssh-agent` does not yet mint the `agentkit` header (see
the transport problem below), so a hard `enforce` default would brick the very client that is
supposed to use this. Shipping `warn` means the plumbing works end to end today and flipping to
`enforce` is a one-env-var decision once the header transport lands — not a code change under
deadline pressure.

## Learnings from a weekend on the beta SDK

Ordered by how much time each cost us.

1. **The header has nowhere to ride.** git talks to the ssh-agent over a unix socket, and a
   socket carries no HTTP headers. Nothing naturally threads an `agentkit` header from git
   through the agent into `signing-service`. The workable answer: the ssh-agent itself holds
   `AGENT_PRIVATE_KEY` and mints the header when it POSTs. The AgentKit chain still genuinely
   runs and still resolves to an Orb-attested human — the socket is just plumbing in front of it.
   This is the single biggest open design question and it is architectural, not a bug.

2. **`resourceUri` must match byte-for-byte.** `validateAgentkitMessage(payload, resourceUri)`
   checks the audience against what the agent signed. Build it from `req.protocol`, `req.host` and
   `req.originalUrl` on the server and make sure the client signs the identical string, or every
   request 403s with a message that does not obviously point at a URL mismatch.

3. **`maxAge` defaults to 5 minutes.** Fine for us, but it means a replayed header stays valid for
   that window. If you need tighter freshness, set it explicitly — do not assume it is seconds.

4. **`lookupHuman` returning `null` is the whole signal.** No exception, no error — a wallet that
   was never registered by a verified human just resolves to `null`. That *is* "this is an
   unbacked bot". Treat null as a first-class reject, not an edge case.

5. **`InMemoryAgentKitStorage` loses everything on restart.** Same as our budget map. Fine for a
   demo, explicitly not fine for anything real — a restart refills every human's daily allowance.
   Noted as a known ceiling, not shipped as a feature.

6. **Registration needs an Orb, once, and is already done.** Selfie Check cannot register an
   agent — only an Orb-verified human scanning a QR can. Do it first because everything downstream
   depends on the wallet already being in AgentBook, and never re-run it casually: a new wallet
   means a new registration means another physical Orb scan.

## What we would build next

The honest limits, and where the interesting work is.

- **Persist the budget.** Move the per-human counter out of memory into anything durable so a
  restart cannot mint free approvals. Small, obvious, not done because it is a weekend.
- **Multi-approver on the sharpest paths.** For `release/**` and workflow files, require two
  distinct `humanId`s. The primitive already supports it; the policy layer does not yet enforce a
  quorum.
- **The compromised-machine gap.** The proof binds to the diff hash the *service* was shown. A
  compromised client that displays one diff and signs another defeats that, and no amount of
  liveness fixes it. This is the most interesting unsolved problem and it is where we would spend
  the next weekend.
