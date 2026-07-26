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
