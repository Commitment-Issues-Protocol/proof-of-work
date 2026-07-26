# World integration: AgentKit + Selfie Check

Everything World-side is provisioned and verified working. Two things to build.

```bash
npm i @worldcoin/idkit-core@4.2.2 @worldcoin/agentkit@0.2.0 qrcode-terminal
```

Pin `4.x`. Every IDKit sample older than that online is v2/v3 and will not work.

## Already live, do not recreate

```
app_id          app_bc54fc90967dc037abf5374c4e6323e9
rp_id           rp_d048b1d7148f48d8
agent address   0x6Cd5F846AD3e9436D24Bb2e71Fe7a6Eb2FF431aA   (registered in AgentBook)
AgentBook       0xA23aB2712eA7BBa896930544C7d6636a96b944dA   (World Chain)
enable_face_check = true
```

Requires **Node 20+** (the shim below uses global `fetch`, `Request` and `Response`).

## You need two secrets, ask for them privately

Neither is in this file and neither can be recovered from what is here.

```bash
WORLD_SIGNING_KEY=     # RP private key, signs every proof request
AGENT_PRIVATE_KEY=     # the wallet registered in AgentBook
```

**Do not generate your own.** The signing key can technically be rotated through the portal API,
but rotating invalidates the existing one and breaks everyone else's setup. The agent key is
worse: generating a new wallet means a new AgentBook registration, which needs another
Orb-verified human physically scanning a QR. Ask for both.

---

# 1. Selfie Check — the QR flow

Docs: [Selfie Check](https://docs.world.org/world-id/credentials/11) ·
[Integrate IDKit](https://docs.world.org/world-id/idkit/integrate) ·
[RP signatures](https://docs.world.org/world-id/idkit/signatures) ·
[Verify endpoint](https://docs.world.org/api-reference/developer-portal/verify) ·
[Error codes](https://docs.world.org/world-id/idkit/error-codes)

### Shim first, or nothing loads in Node

`idkit-core` fetches its WASM over a `file://` URL and Node refuses. No `initSync` export, so
this is the only way in. Must run **before** importing the package.

```js
import { readFile } from "node:fs/promises";
const realFetch = globalThis.fetch;
globalThis.fetch = async (input, init) => {
  const raw = input instanceof Request ? input.url : String(input?.url ?? input);
  const url = input instanceof URL ? input : new URL(raw);
  if (url.protocol === "file:")
    return new Response(await readFile(url), { headers: { "Content-Type": "application/wasm" } });
  return realFetch(input, init);
};
const { IDKit, selfieCheckLegacy } = await import("@worldcoin/idkit-core");
```

### Mint → QR → poll → verify

```js
import { signRequest } from "@worldcoin/idkit-core/signing";
import { hashSignal } from "@worldcoin/idkit-core/hashing";  // pure JS, no WASM

const APP_ID = "app_bc54fc90967dc037abf5374c4e6323e9";
const RP_ID = "rp_d048b1d7148f48d8";
const signingKeyHex = process.env.WORLD_SIGNING_KEY;

// Unique action per event. Actions carry max_verifications: 1 and v4 nullifiers are
// one-time-use, so a fixed action can only ever be used once per human.
//
// Dynamic actions do NOT need registering in the portal. The docs' troubleshooting table
// says "action not found" means you forgot to create it in that environment — that applies
// to static actions only. We ran `git-sign:<sha>` unregistered against production and it
// verified fine. Ignore that warning for dynamic actions.
const action = `git-sign:${sha}`;
const signed = signRequest({ signingKeyHex, action, ttl: 60 });
// -> { sig, nonce, createdAt, expiresAt }

const request = await IDKit.request({
  app_id: APP_ID,
  action,
  rp_context: {
    rp_id: RP_ID,
    nonce: signed.nonce,
    created_at: signed.createdAt,
    expires_at: signed.expiresAt,
    signature: signed.sig,        // note the rename
  },
  allow_legacy_proofs: true,      // required, Selfie Check is World ID 3.0 only
  require_user_presence: true,    // forces a FRESH liveness check
  environment: "production",
}).preset(selfieCheckLegacy({ signal: sha }));

request.connectorURI    // QR on desktop, deep link on mobile. Same string.

const completion = await request.pollUntilCompletion({ pollInterval: 2000, timeout: 120_000 });

// Forward the result VERBATIM. Do not reshape, do not compute signal_hash yourself.
const res = await fetch(`https://developer.world.org/api/v4/verify/${RP_ID}`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/140.0.0.0 Safari/537.36",
  },
  body: JSON.stringify(completion.result),
});
```

### All four checks, or the proof is worthless

```js
verify.success === true
result.nonce === signed.nonce                     // <-- everyone misses this one
result.user_presence_completed === true
result.responses[0].signal_hash === hashSignal(sha)
```

**The verify endpoint does not check your nonce.** It only confirms the proof is
cryptographically valid. Skip that comparison and an attacker replays an old proof while
everything still returns green.

### What comes back

```js
{
  action: 'git-sign:4f2c1ab…',
  nonce: '0x009ff976…',
  protocol_version: '3.0',
  responses: [{
    identifier: 'face',            // not "selfie"
    nullifier: '0x00040413…',      // scoped to rp + action
    signal_hash: '0x00ca5bf1…',
    merkle_root: '0x3a12bc…',
    proof: '0x1975cf…',
  }],
  user_presence_completed: true,
}
```

No Sybil score field, despite the docs describing one in the present tense. Not enabled yet.

---

# 2. AgentKit — which human is behind the agent

Docs: [Integrate](https://docs.world.org/agents/agent-kit/integrate) ·
[SDK reference](https://docs.world.org/agents/agent-kit/sdk-reference) ·
[AgentBook](https://agentbook.world/)

Agent is already registered, so this works right now:

```js
import { createAgentBookVerifier } from "@worldcoin/agentkit";
const agentBook = createAgentBookVerifier();
await agentBook.lookupHuman("0x6Cd5F846AD3e9436D24Bb2e71Fe7a6Eb2FF431aA");
// -> "0xc9a3e0cf…"   stable anonymous humanId, same on every commit
// -> null            unregistered, i.e. a bot
```

### Verifying an incoming agent request

Agent sends an `agentkit` header (base64 SIWE). Server side, four calls:

```js
import {
  AGENTKIT, parseAgentkitHeader, validateAgentkitMessage,
  verifyAgentkitSignature, createAgentBookVerifier,
} from "@worldcoin/agentkit";

const payload = parseAgentkitHeader(req.headers[AGENTKIT]);

// resourceUri = the absolute URL of the endpoint being protected, e.g.
// "http://localhost:3000/sign/abc123". It must match what the agent signed.
await validateAgentkitMessage(payload, resourceUri);   // maxAge default 5 min

const verification = await verifyAgentkitSignature(payload);
const humanId = await agentBook.lookupHuman(verification.address);
if (!humanId) return 403;
```

Agent side, `createAgentkitClient()` needs `AGENT_PRIVATE_KEY` (the registered wallet), then call
`agentkit.fetch` instead of `fetch` and it attaches the header for you.

**Open design question:** git talks to the ssh-agent over a unix socket, so nothing naturally
carries an HTTP header into `signing-service`. Simplest workable answer is that the ssh-agent
holds `AGENT_PRIVATE_KEY` and mints the header when it POSTs. The chain still genuinely runs and
still resolves to an Orb-attested human. Worth agreeing on before you build it.

### Use cases: what to build with it

The point is that a service can tell a bot from an agent acting for a real unique human. Prize
brief names: access, authorization, **rate limits**, economic terms, payments, commerce,
moderation, **accountability**.

Ours:

- **Accountability** — `humanId` names who is answerable for a commit, without revealing who
  they are.
- **Rate limits** — budget counters keyed on `humanId`, so one limit follows the person across
  every machine and agent they run. One tired human cannot bless a thousand agent commits.
- **Continuity** — Selfie Check nullifiers change every commit (scoped to rp + action), so they
  cannot be linked. `humanId` is the stable identity. This is why we need both.

### Never use these

```js
{ type: "free" }   { type: "free-trial" }   { type: "discount" }
```

AgentKit's built-in access modes are the **explicitly disqualified pattern** for this prize
("human-backed benefits for AI agents, i.e. API calls, discounts"). Also banned: agent
reputation, i.e. anything that looks like a trust score next to an agent name. We use identity
resolution only.

`InMemoryAgentKitStorage` is fine for the demo, loses state on restart.

---

# 3. Where this plugs into `signing-service`

Three seams, all already stubbed in `src/app.ts`.

| Now | Change to |
|---|---|
| `getVerificationUrl()` returns `https://example.com/verify/...` | return `request.connectorURI` |
| `GET /verify/:requestId` calls `approvePendingRequest()` unconditionally | verify the proof + run all four checks first, then approve |
| `POST /sign/:requestId` signs with no identity check | run the AgentKit chain, 403 if `humanId` is null, then check budget |

Fits the existing pending-request pattern without restructuring it:

```
POST /sign/:requestId
  ├─ agentkit header -> humanId          (403 if null)
  ├─ budget check on humanId
  ├─ signal = hash of the data being signed
  ├─ mint IDKit request, keep { nonce, action, expectedSignalHash }
  ├─ return connectorURI so the human can scan
  └─ pollUntilCompletion() in the background
        ├─ 4 checks pass -> approvePendingRequest(requestId)
        └─ else           -> rejectPendingRequest(requestId, reason)
```

Three existing bugs worth fixing while you are in there:

- `ssh-agent/src/agent/signer.ts` POSTs to `${apiURL}/sign`, but `app.ts` exposes
  `/sign/:requestId`. One of them is wrong.
- `PENDING_REQUEST_TIMEOUT_MS` is 60s, RP signature TTL defaults to 300s. Make the pending
  timeout longer than the proof window or a slow scan dies on the wrong side.
- The signature is computed *before* approval and held in memory. Works, but consider computing
  it only after the human approves.

---

## Environments

All three must agree or you get a dead QR and no error: the IDKit `environment` prop, the
action's environment, and which app scans it.

```
"staging"      simulator.worldcoin.org in a browser, no phone needed
"production"   real World App from the App Store
```

Both registered for us. Unverified whether the simulator can do Selfie Check specifically
(liveness needs a real camera), so test plumbing on staging, test the credential on production.

## Gotchas

1. WASM shim before importing `idkit-core`.
2. Compare the nonce yourself.
3. Unique action per event, else `nullifier_replayed`.
4. `allow_legacy_proofs: true` for Selfie Check. Identity Check is 4.0-only, so the two cannot
   share one request.
5. Browser User-Agent on every `developer.world.org` call, or Cloudflare 1010.
6. `identifier` is `'face'`.
7. AgentKit registration needs an Orb and is already done. Do not re-run it.

## Reference

The code in section 1 is complete and verified. Assembled into one file it runs standalone: shim,
`signRequest`, `IDKit.request`, print `connectorURI` as a QR with `qrcode-terminal`,
`pollUntilCompletion`, POST to verify, then the four assertions. That exact script passed against
a real face on production on 25 Jul. Build that first as a scratch script before touching
`signing-service`, so you know your credentials and environment are right before you add
plumbing on top.

Whole World documentation site as one 506 KB file: https://docs.world.org/llms-full.txt
Download it and point your AI agent at that instead of letting it fetch pages one at a time.

Official World ID agent skill, written for exactly this task (portal setup, RP signing,
verification, nullifier replay protection, Face Check gate, plus a symptom-to-cause
troubleshooting table): https://docs.world.org/world-id/SKILL
