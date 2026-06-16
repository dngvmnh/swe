# Module 19 — Integration & Protocol Design

> **Core question:** When two systems you partly control must talk, how do you design the protocol — who trusts whom, what crosses the wire, and how do you stop the two ends from drifting apart?
> **Source:** the Mythos crawl (`backend/docs/adr/*`, the SaaS-pivot proposal, `mythos-sdk-demo` contract findings).
> **Prerequisites:** [04 — Auth](04-auth.md), [08 — Security](08-security.md), [10 — Testing](10-testing.md), [17 — SDK design](17-sdk-library-design.md).

## Why this module
Mythos's whole business depends on one integration: a third-party SaaS, launched inside Mythos, must trust "this user is allowed and has credits" — *without* Mythos handing over passwords and *without* the SaaS being able to cheat the meter. That's a protocol-design problem, and the way Mythos solved it (invert the direction, sign a short-lived token, hide the security-critical call inside the SDK, test the contract with an independent backend) is a masterclass you can reuse for any service-to-service integration.

### What you'll be able to do
- Dissolve a hard integration by inverting its direction.
- Design a launch token modeled on the OAuth authorization code.
- Reason about token-delivery trade-offs and document an upgrade path.
- Place a security invariant on the right side of the trust boundary.
- Make state transitions system-driven (API as the primitive, UI as a wrapper).
- Catch producer/consumer contract drift with an independent reference implementation.

---

## Lesson 19.1 — Invert the integration direction to dissolve the hard problem

**Goal:** when an integration is intractable in one direction, flip it.

### Concept
Mythos faced two "hard problems": **auth bypass** (don't make the user log in again) and **payment bypass** (don't let the SaaS dodge metering). The naive direction — *Mythos forwards the user's Google credentials into each SaaS* — has a huge blast radius (Mythos holds scoped refresh tokens, per-SaaS consent, only works for Google-IdP apps). **Inverting the direction** dissolves it: *the SaaS integrates a small Mythos SDK* that accepts a signed launch token (identity + entitlement) and reports usage back. Now the creator integrates **once**, it works for **any** SaaS, and Mythos never holds the user's third-party credentials. (Same pattern as Steam's auth ticket, Slack OIDC, Roblox Open Cloud.)

### Worked example — the decision table (pivot proposal §2.1)
```
A — Forward Google creds   → ❌ huge blast radius, Google-only, per-SaaS consent
B — Mythos SDK launch token → ✅ short-lived signed JWT (sub, aud, exp, entitlement_id);
                                 SaaS verifies via JWKS, creates its OWN session. Integrate once.
C — Hybrid (B default, A opt-in for apps that want Google scopes) → Phase 2
```

### Check yourself
1. Why is "SDK into the SaaS" lower-risk than "creds into the SaaS"? (Mythos never holds/forwards third-party credentials; the trust flows *out* as a verifiable token instead of *in* as a secret.)
2. What did inverting buy the creator? (Integrate **once** against the SDK, works regardless of their existing identity provider.)

### ✅ Lesson 19.1 checklist
- [ ] 📖 Read & understood — I can explain integration-direction inversion and why it shrinks blast radius.
- [ ] 💻 Applied — I re-framed a "push secrets in" integration as a "verify a token out" one.
- [ ] 🔍 Found in Mythos — `MYTHOS_SAAS_PIVOT_PROPOSAL.md` §0/§2.1 (the inversion + options table).

---

## Lesson 19.2 — The launch token as an OAuth authorization-code analog

**Goal:** design a hand-off token with the right four properties.

### Concept
The launch token is a deliberate analog of an **OAuth authorization code**: **short-lived, single-use, signed, and audience-scoped.** Each property defends a specific attack:
- **Short-lived** (5-min `exp`) → a leaked token is useless minutes later.
- **Single-use** (consumed once) → replay is blocked (Lesson 19.4).
- **Signed** (RS256, verified via JWKS) → can't be forged ([Module 16](16-cryptography-key-management.md)).
- **Audience-scoped** (`aud = listing_id`) → a token for app A can't launch app B ([Module 17](17-sdk-library-design.md), audience validation).

```
launch JWT claims:  sub = user_id,  aud = listing_id,  exp = +5min,
                    entitlement_id,  jti  (the single-use handle)
SaaS: verify signature via JWKS → check aud → consume jti once → create its OWN session
```

### ✅ Lesson 19.2 checklist
- [ ] 📖 Read & understood — I can list the four token properties and the attack each defends.
- [ ] 💻 Applied — I designed a hand-off token with exp + single-use + signature + audience.
- [ ] 🔍 Found in Mythos — pivot §2.1 option B; `aud`/`exp`/`jti` in `verify.py`/`apps.service.ts`.

---

## Lesson 19.3 — Token-delivery trade-offs and a documented upgrade path

**Goal:** pick the simplest delivery that works for MVP, name its risk, and write down the upgrade.

### Concept
*How* the token reaches the SaaS is its own decision. For an iframe MVP, Mythos passes it as a **URL query param** (`?token=...`) — simplest, no JS coordination, unblocks a one-line `GET /launch` handler. The honest part: the ADR **states the trade-off** (tokens land in server logs / browser history), **mitigates** it (redact `/launch` logs, 5-min expiry), and **names the upgrade path** (`postMessage`, mandated later). That's how you take on debt responsibly — documented, bounded, with an exit.

### Worked example — ADR-0001
```
Decision: deliver the launch token via ?token= query param for MVP.
Trade-off (accepted): tokens appear in logs + history.
Mitigation: instruct Producers to redact /launch logs; tokens expire in 5 min.
Upgrade path: postMessage (documented), mandated when the integration surface matures.
```

### ❌ Bad → ✅ Good
- ❌ Ship the query-param approach silently; nobody knows it's interim or why it's risky.
- ✅ ADR records the decision, the accepted risk, the mitigation, and the upgrade trigger.

### ✅ Lesson 19.3 checklist
- [ ] 📖 Read & understood — I can take on a delivery trade-off responsibly (state risk, mitigate, name the upgrade).
- [ ] 💻 Applied — I wrote a short ADR for an interim mechanism with an upgrade path.
- [ ] 🔍 Found in Mythos — `backend/docs/adr/0001-launch-token-delivery-query-param.md`.

---

## Lesson 19.4 — Place the invariant on the right side of the trust boundary

**Goal:** enforce a platform-critical rule where it *can't* be bypassed.

### Concept
The single-use guarantee is a **Mythos platform invariant**, not something a third-party SaaS should be trusted to honour. So the `consume` call lives **inside the SDK middleware** (`requireLaunchToken`) — automatic, not skippable. A misconfigured or malicious SaaS can't "forget" to consume or delay it; the security property's accountability sits with Mythos. (If a Producer bypasses the SDK with raw HTTP, they lose single-use protection and violate the agreement — enforced at publish-time review.) **Trust-boundary placement:** put the enforcement where you control it, not where you hope a partner will.

### Worked example — ADR-0003
```
consume (single-use enforcement) is executed automatically INSIDE mythos.express.requireLaunchToken().
Producers cannot skip it, delay it, or call it themselves.
→ a malicious/misconfigured SaaS cannot enable token replay; accountability stays with Mythos.
```
This is why the SDK's `require_launch_token` *always* calls `consume` and **fails closed** if it can't ([Module 18](18-resilience-failure-modes.md), Lesson 18.1).

### Check yourself
1. Why not document "please call /consume" and trust the SaaS? (Security can't depend on a partner remembering or choosing to comply; put it inside the code they can't bypass.)
2. What enforces the rule for a SaaS that skips the SDK entirely? (Out-of-band: publish-time handshake + listing review + the creator agreement — the technical control is backed by a process control.)

### ✅ Lesson 19.4 checklist
- [ ] 📖 Read & understood — I can explain trust-boundary placement and why invariants go SDK-internal.
- [ ] 💻 Applied — I moved a security-critical step out of "integrator responsibility" into enforced code.
- [ ] 🔍 Found in Mythos — `backend/docs/adr/0003-consume-endpoint-is-sdk-internal.md`; SDK `require_launch_token`.

---

## Lesson 19.5 — System-driven state transitions (API is the primitive)

**Goal:** make state change by deterministic rules, with the API as the primitive and the UI a thin wrapper.

### Concept
Entitlements expire only through **deterministic system events** — balance exhaustion or time-box expiry — *not* arbitrary human action, which preserves consumer trust ("access follows rules, not someone's mood"). For emergencies there's an API kill switch (`DELETE /listings/{id}/issuance`), and crucially the decision rejects a **dashboard-only** control: a UI button depends on a human with credentials and the frontend being up during an incident. **The API is the primitive; the dashboard merely exposes it.** Design the capability as an API first; the UI is one client of it.

### Worked example — ADR-0002 (options considered)
```
✅ System-driven expiry (balance/time) + API kill switch (DELETE /listings/{id}/issuance)
❌ Producer per-entitlement revocation — breaks consumer trust, human counterparty risk
❌ Dashboard-only kill switch — needs a human + a live frontend during an incident; API is the primitive
On any revocation: entitlement_revoked webhook fires → SaaS terminates the session.
```

### ✅ Lesson 19.5 checklist
- [ ] 📖 Read & understood — I can explain system-driven transitions and "API as primitive, UI as wrapper."
- [ ] 💻 Applied — I designed a capability as an API first, with the UI as one client.
- [ ] 🔍 Found in Mythos — `backend/docs/adr/0002-entitlement-revocation-model.md`.

---

## Lesson 19.6 — Catch contract drift with an independent reference

**Goal:** find producer/consumer mismatches before they reach production.

### Concept
The SDK (consumer) and backend (producer) are built by different efforts — so they **drift**. The Mythos team caught three live mismatches only by building an **independent mock backend** that implements the contract the SDK *expects*, then running the real SDK against it end-to-end:
1. **Path mismatch** — SDK fetches `/.well-known/jwks.json` (global); backend serves `/api/listings/:id/jwks` (per-listing). They never resolve to the same URL.
2. **Envelope mismatch** — backend wraps `{ success, data: { keys } }`; the SDK passes the raw JSON into `jwt.decode`, which wants a bare `{ keys }`.
3. **Missing endpoints** — the SDK calls `/consume` and `/meter`; the backend branch implemented only `/launch`.

The lesson: **a contract documented in prose still drifts; verify it with an executable reference and an end-to-end test.** (Well-known endpoints are conventionally *unwrapped* — a reminder that envelope conventions are part of the contract.)

### Worked example — `mythos-sdk-demo/fixes/ISSUE-3-sdk-backend-contract.md`
```
| | Path |
| SDK fetches   | GET {API}/.well-known/jwks.json   (global)      |
| Backend serves| GET /api/listings/:listingId/jwks (per-listing) |   ← never the same URL

Acceptance (the contract test):
- [ ] SDK fetches JWKS from the backend (path + envelope agreed)
- [ ] /consume enforces single-use (200 → 409)
- [ ] /meter debits the wallet and returns 402/404 correctly
- [ ] one integration test: launch → verify → consume → meter against the REAL backend
```

### ❌ Bad → ✅ Good
- ❌ "The API doc says `/jwks.json` returns `{keys}`" — both teams code to the doc, never run them together, ship, 100% of launches fail.
- ✅ Stand up a mock of the *expected* contract, run the real consumer against it, and write a launch→verify→consume→meter integration test that exercises both ends.

### ✅ Lesson 19.6 checklist
- [ ] 📖 Read & understood — I can explain contract drift and catching it with an independent reference + E2E test.
- [ ] 💻 Applied — I wrote a contract/integration test that runs a real client against a producer.
- [ ] 🔍 Found in Mythos — `mythos-sdk-demo/fixes/ISSUE-3-sdk-backend-contract.md`; `FINDINGS.md` (F3/F4/F5).

---

## 🎯 Module 19 mastery checklist
- [ ] I can dissolve a hard integration by inverting its direction.
- [ ] I can design a hand-off token that is short-lived, single-use, signed, and audience-scoped.
- [ ] I can take on a delivery trade-off with a stated risk, mitigation, and upgrade path.
- [ ] I can place a security invariant inside code the integrator can't bypass.
- [ ] I can design state transitions as system-driven, API-first capabilities.
- [ ] I can catch producer/consumer contract drift with an independent reference and an E2E test.

## 🛠️ Mini-project
Design a mini "launch" protocol between two services you control:
1. Service A mints a short-lived, signed, audience-scoped, single-use token; Service B verifies it via A's JWKS and creates its own session.
2. Put the single-use `consume` call inside a tiny SDK middleware B imports — make it fail closed.
3. Write an ADR for how the token is delivered (query param vs header) with a stated trade-off + upgrade path.
4. Build an **independent mock** of A's contract and run B's real SDK against it; add a launch→verify→consume integration test. Then deliberately change A's response envelope and watch the test catch the drift.

## 🔗 Mythos source map
| File / PR | What it demonstrates |
|-----------|----------------------|
| `docs/adr/MYTHOS_SAAS_PIVOT_PROPOSAL.md` | Integration-direction inversion; auth-bypass options table. |
| `docs/adr/0001-launch-token-delivery-query-param.md` | Delivery trade-off + documented upgrade path. |
| `docs/adr/0003-consume-endpoint-is-sdk-internal.md` | Trust-boundary placement (invariant inside the SDK). |
| `docs/adr/0002-entitlement-revocation-model.md` | System-driven transitions; API as primitive. |
| `mythos-sdk-demo/fixes/ISSUE-3-sdk-backend-contract.md` + `FINDINGS.md` | Contract drift caught by an independent mock backend. |

## See also
- [17 — SDK & library design](17-sdk-library-design.md) — the SDK that carries this protocol.
- [16 — Cryptography & key management](16-cryptography-key-management.md) — the signing/JWKS underneath.
- [04 — Auth](04-auth.md) · [08 — Security](08-security.md) — identity & trust boundaries.
- [10 — Testing](10-testing.md) — contract & integration testing.
