# World integration notes

Everything below was verified by running it on 25 July 2026, not read off a docs page.
Doubles as our submission for the **Selfie Check Beta** testing-documentation requirement.

SDK versions in use: `@worldcoin/idkit-core@4.2.2`, `@worldcoin/agentkit@0.2.0`,
`@worldcoin/agentkit-cli@0.2.0`.

---

## What works, with evidence

### Selfie Check, production, real face

```
identifier               face
protocol_version         3.0
user_presence_completed  true
verify                   HTTP 200, "Proof verified successfully"
```

All four of our assertions passed on the first real run:

1. `verify.success === true`
2. `result.nonce` equalled the nonce we minted (60s window)
3. `user_presence_completed === true`
4. `signal_hash === hashSignal(commitSha)`

No Orb needed. World App from the App Store was enough.

### AgentKit

```
agent      0x6Cd5F846AD3e9436D24Bb2e71Fe7a6Eb2FF431aA
tx         0x1aba81a4ef1ab53f150f41c5eb7c3ce5d60df97e1839e4f630d3778636613e6d
lookup     resolves to a stable anonymous humanId
control    0x…dEaD returns null, as expected
```

Registration went through the hosted relay at `x402-worldchain.vercel.app`. No gas, nothing funded.

---

## Developer feedback

Ordered by how much time each one cost us.

**1. `idkit-core` cannot load in Node without a shim.** It fetches its WASM over a `file://`
URL and Node refuses that. There is no exported `initSync`, so the public API gives you no way
in. About ten lines of `globalThis.fetch` patching fixes it. Nothing in the docs mentions this,
presumably because nobody expected a CLI. This is the single biggest blocker for anyone building
a non-browser integration.

**2. The verify endpoint does not check your nonce.** `POST /api/v4/verify/{rp_id}` validates the
proof and returns success, but it does not confirm that the nonce inside the proof is the one you
minted. You have to compare it in your own code. Miss it and the entire "a human was present just
now" guarantee silently evaporates while everything still looks green. This deserves a warning box
in the docs.

**3. `enable_face_check` is invisible until you go looking.** A valid app, rp and action does not
imply access to the credential. The failure mode is the flow hanging as an unexplained spinner, so
you assume your own code is broken. `POST /api/v1/precheck/{app_id}` with `{"action":"..."}` is the
check, and it should be step one of the Selfie Check quickstart rather than a note in a SKILL file.

**4. The "Sybil score" does not exist in the response.** The credential page says Selfie Check
"returns a Sybil score, a similarity signal that flags whether the user has created an abnormal
number of accounts on your platform". There is no such field in the actual payload. We got
`identifier`, `merkle_root`, `nullifier`, `proof`, `signal_hash`, and a top-level
`user_presence_completed`. Nothing else. Either the docs are ahead of the API or the score is
internal. Right now a developer reading that sentence will design a risk flow around a field that
is not there.

**5. `max_verifications: 1` plus one-time-use nullifiers is a sharp edge.** A fixed action can be
used exactly once per human, ever. For anything repeated you need a unique action per event.
Dynamic actions do work for verify without portal registration, which is the escape hatch, but it
is documented in passing rather than prominently.

**6. Per-event actions destroy linkability, and that is not obvious upfront.** Nullifiers are
scoped to rp plus action, so a unique action per event gives you a fresh unlinkable nullifier every
time. If your product needs "is this the same person as last time", you have to know this before
you design your schema. We only avoided a rewrite because AgentKit's `humanId` gave us continuity
from the other direction.

**7. Ordering is undiscoverable from the error messages.** `create_world_id_action` fails with
"World ID is not configured for this app" and the precheck fails with "No action found for this
app". Neither tells you the actual sequence, which is:

```
create_app -> configure_world_id -> poll until registered -> create_world_id_action -> precheck
```

**8. Cloudflare blocks default Python and curl user agents** on `developer.world.org` with error
1010 `browser_signature_banned`. Any server-side integration test script hits this immediately and
the error looks nothing like an auth problem.

**9. `agentkit-cli@0.2.0` ships a deprecated dependency.** Installing it prints
`npm warn deprecated @worldcoin/idkit-core@2.1.0: Old-versions moved to new ones`. Minor, but it
undermines confidence at the exact moment a developer is deciding whether to trust the SDK.

**10. Selfie Check is 3.0-only and Identity Check is 4.0-only**, so they cannot share one request.
That is a real product constraint and worth stating on both credential pages rather than being
inferred from `allow_legacy_proofs`.

### What was genuinely good

- The **Developer Portal MCP** is excellent. App creation, RP configuration, key generation,
  action creation and registration polling without touching a dashboard. `configure_world_id`
  went from `pending` to `registered` on both production and staging in about four seconds.
- `require_user_presence: true` works on any preset and is the cleanest step-up primitive in the
  stack. It gave us a fresh liveness signal without adopting a whole new credential.
- The `signal` mechanism is exactly the right shape for binding a proof to application context.
  `hashSignal()` being a pure JS subpath export with no WASM init made server-side comparison
  trivial.
- The staging simulator plus production split, once you know the environments must match, is a
  good testing story.
- `agentkit-cli register` was the smoothest part of the whole day. QR, scan, done, no gas.

---

## User feedback

The UI was great. Intuitive and fast, with nothing to explain and no confusion at any step. From
tapping the link to holding a verified credential was a matter of seconds, and nobody hesitated or
declined. The camera and selfie step just worked. No drop-off, no re-tries, no "wait, what is this
asking me to do".

The one thing we'd genuinely want: a way to do the whole flow **directly in the browser**. Capture
is phone-only today. We grepped the shipped `idkit-core` bundle and there is no `getUserMedia`, no
`mediaDevices`, and no camera permission request anywhere in the SDK. A desktop browser hitting
`world.org/verify` is redirected to a download page, so a laptop-only user cannot complete the flow
at all and everyone is forced through a QR handoff to their phone. The handoff itself is smooth, but
a native in-browser selfie capture would remove the only friction we hit.

---

## Unverified, do not assert

- **The 90-day validity period on credential 11.** It is rendered from a JSX prop in the docs
  component and appears in neither the page text nor the API response. If real, it should mean the
  credential expires and the user re-enrols, not that a proof is cached. Needs confirmation from
  the World team.
- **Whether the staging simulator can complete a Selfie Check.** It generates proofs, but a
  liveness selfie needs a real camera, so we assume not and test the credential on production.
