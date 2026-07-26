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
