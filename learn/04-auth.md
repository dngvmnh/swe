# Module 04 — Authentication & Authorization

> **Core question:** How do you know *who* is calling and *what* they may do?
> **Reference doc:** [../04-authentication-and-authorization.md](../04-authentication-and-authorization.md)
> **Prerequisites:** basic HTTP. Pairs with [08 — Security](08-security.md) and [06 — Errors](06-error-handling.md) (401/403 mapping).

## Why this module
The moment a system has more than one user and anything worth protecting, it must distinguish
callers and gate actions. Get authn wrong → impostors get in. Get authz wrong → legitimate users
reach things they shouldn't (the #1 serious web vuln class: *broken access control*).

### What you'll be able to do
- Keep authentication and authorization distinct.
- Choose sessions vs tokens, and symmetric vs asymmetric signing — and explain JWKS.
- Implement "required" and "optional" auth *correctly* (the subtle bug most people ship).
- Harden cookies and clean up auth state on logout.

---

## Lesson 4.1 — Authn vs authz (don't conflate them)

**Goal:** separate the two questions.

### Concept
- **Authentication (authn): *who are you?*** Proving identity — a password, a Google login, a signed token.
- **Authorization (authz): *what are you allowed to do?*** Deciding whether a principal may perform an action — roles, ownership, scopes.

You **authenticate first, then authorize.** A system can authenticate you perfectly and still
(correctly) deny you. **Logged in ≠ allowed.**

```ts
// authn: who is this? (verify the token)         → 401 if it fails
const user = verifyToken(req.headers.authorization);
// authz: may THIS user do THIS? (ownership/role) → 403 if it fails
if (resource.ownerId !== user.id) throw new ForbiddenError();
```

### ✅ Lesson 4.1 checklist
- [ ] 📖 Read & understood — I can state the difference and the order (authn → authz).
- [ ] 💻 Applied — I labeled the authn vs authz step in an endpoint I know.
- [ ] 🔍 Found in Mythos — `FM#39` renders different dashboards by role (authz-driven composition).

---

## Lesson 4.2 — Sessions vs tokens

**Goal:** choose a session model and know its trade-offs.

### Concept
| | Server-side sessions | Stateless tokens (JWT) |
|---|---|---|
| Where state lives | server stores it; client holds an opaque cookie | the token itself carries signed claims |
| Verify a request | look up the session | verify the signature — no lookup |
| Revoke | easy (delete the session) | **hard before expiry** → keep lifetimes short |
| Scaling | needs shared session storage | scales horizontally |

> **Rule of thumb:** if you need instant revocation, lean sessions (or short-lived tokens + a
> refresh/blocklist). If you need stateless horizontal scale, lean JWT with short TTLs.

### ✅ Lesson 4.2 checklist
- [ ] 📖 Read & understood — I can explain the revocation/scaling trade-off.
- [ ] 💻 Applied — I picked a model for a hypothetical app and justified it.
- [ ] 🔍 Found in Mythos — `FM#47`/`FM#48` migrated frontend from cookie tokens to **NextAuth.js v5 sessions**.

---

## Lesson 4.3 — Symmetric vs asymmetric signing & JWKS

**Goal:** understand the one idea behind third-party verification.

### Concept
- **HMAC (HS256):** one **shared secret** signs *and* verifies. Simple — but every verifier must
  hold the secret.
- **RSA/ECDSA (RS256):** a **private** key signs, a **public** key verifies. Third parties can
  verify **without ever seeing your secret** — the basis of **JWKS** (a published set of public keys
  at `/.well-known/jwks.json`).

```
Mythos                              Creator's web app (third party)
  ├─ private key (secret, AES-GCM encrypted at rest)
  ├─ signs launch JWT (RS256) ─────────────────► receives JWT
  └─ publishes PUBLIC key via JWKS ────────────► fetches JWKS, verifies signature
                                                  (never needs Mythos's secret) ✅
```

> **This is precisely why RS256 + JWKS exists:** verification by parties you don't share secrets with.

### Check yourself
1. Why can't you hand a creator your HMAC secret so they can verify launch tokens? (Then they could
   *forge* tokens too — signing and verifying use the same key.)

### ✅ Lesson 4.3 checklist
- [ ] 📖 Read & understood — I can explain why asymmetric signing enables third-party verification.
- [ ] 💻 Applied — I described where I'd use HS256 vs RS256.
- [ ] 🔍 Found in Mythos — `BE#75`/`BE#78`/`BE#84`: per-listing RSA-2048 keys, RS256 launch JWTs, JWKS endpoint.

---

## Lesson 4.4 — OAuth/OIDC & password storage

**Goal:** delegate identity safely and store passwords correctly.

### Concept
- **OAuth / OpenID Connect.** Delegate authn to a provider (Google) so **you never handle the user's
  password**. You receive an identity assertion and map it to a local account.
- **Password storage.** Never store plaintext. Use a **slow, salted hash** (bcrypt/argon2) with a
  sensible cost factor. The slowness is the point — it makes brute-force expensive.

```ts
// ✅ store a slow salted hash, never the password
const hash = await bcrypt.hash(password, 12);   // cost factor 12
// verify later
const ok = await bcrypt.compare(attempt, hash);
```

### ✅ Lesson 4.4 checklist
- [ ] 📖 Read & understood — I can explain OAuth delegation and why password hashes must be slow+salted.
- [ ] 💻 Applied — I hashed and verified a password with bcrypt/argon2.
- [ ] 🔍 Found in Mythos — `BE#1` password register/login; `BE#4` Google OAuth; `BE#21` Google login error handling.

---

## Lesson 4.5 — Required vs optional auth (the bug everyone ships)

**Goal:** implement optional auth without creating a bypass.

### Concept
- **Required auth:** missing/invalid token → reject (`401`).
- **Optional auth:** works anonymously but personalizes if a token is present — **but a
  *present-but-invalid* token must still be rejected, never silently ignored.** Silently accepting
  invalid tokens turns "optional" into a **bypass**.

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — swallows an invalid token and proceeds as anonymous → bypass
function optionalAuth(req, _res, next) {
  try { req.user = verify(req.headers.authorization); } catch { /* ignore */ }
  next();
}
```
```ts
// ✅ GOOD — no token → anonymous; bad token → 401
function optionalAuth(req, _res, next) {
  const token = req.headers.authorization;
  if (!token) return next();                         // truly optional
  try { req.user = verify(token); next(); }
  catch { throw new AuthError("invalid token"); }    // present-but-invalid → reject
}
```

### Check yourself
1. What are the three cases optional auth must handle? (no token → anonymous; valid token →
   personalized; invalid token → 401.)

### ✅ Lesson 4.5 checklist
- [ ] 📖 Read & understood — I can state the three-case rule for optional auth.
- [ ] 💻 Applied — I implemented optional auth that rejects present-but-invalid tokens.
- [ ] 🔍 Found in Mythos — `BE#17` fixed "optional auth silently ignores invalid JWT"; `BE#35` re-encoded the 401/403 rules in typed middleware.

---

## Lesson 4.6 — Cookie security, claim shapes & clean logout

**Goal:** close the operational gaps.

### Concept
- **Cookie security:** `HttpOnly` (no JS access → mitigates XSS token theft), `Secure` (HTTPS only),
  `SameSite` (CSRF mitigation). **Clear auth state fully on logout.**
- **Claim-shape drift.** If the issuer puts `id` in the token but a service reads `userId`, you get a
  silent undefined-user / 0-results bug. Keep issuer and verifier in lockstep.
- **Secrets out of source.** No credentials in the repo or the client bundle; `.env.example` ships
  placeholders only.

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — mismatched claim shape, silently breaks lookups
const profile = await getProfile(req.user.userId);  // but token sets req.user.id
// ❌ BAD — logout that leaves userId lingering
res.clearCookie("token"); // ...but userId still set elsewhere
```
```ts
// ✅ GOOD — aligned shape + full teardown
const profile = await getProfile(req.user.id);
res.clearCookie("token", { httpOnly: true, secure: true, sameSite: "lax" });
clearUserId();   // clear ALL auth state
```

### ✅ Lesson 4.6 checklist
- [ ] 📖 Read & understood — I can list the cookie flags and why claim shapes must match.
- [ ] 💻 Applied — I set `HttpOnly`/`Secure`/`SameSite` and wrote a clean-logout path.
- [ ] 🔍 Found in Mythos — `BE#48` claim-shape `id` vs `userId` fix; `FM#50` secure cookies + clearing `userId` on logout.

---

## 🎯 Module 04 mastery checklist
- [ ] I can distinguish authn from authz and apply 401 vs 403 correctly.
- [ ] I can choose sessions vs tokens and justify it by revocation/scale needs.
- [ ] I can explain symmetric vs asymmetric signing and why JWKS enables third-party verification.
- [ ] I can implement optional auth that rejects present-but-invalid tokens.
- [ ] I can harden cookies, align claim shapes, and fully clear auth state on logout.

## 🛠️ Mini-project
1. Build `POST /register`, `POST /login` (bcrypt), and a `requireAuth` middleware (JWT, 401 on bad token).
2. Add an `optionalAuth` middleware and an endpoint that personalizes when a token is present but
   works anonymously otherwise — and **rejects a present-but-invalid token**. Write a test for all 3 cases.
3. Add an ownership check on a resource so a logged-in user gets **403** on someone else's record.
4. Bonus: sign a token with RS256 and verify it with only the public key (mini-JWKS).

## 🔗 Mythos PR map
| PR | What it demonstrates |
|----|----------------------|
| `BE#1`, `BE#4`, `BE#21` | Password auth, Google OAuth, OAuth error handling. |
| `BE#17`, `BE#35` | Optional-auth invalid-token bug + typed 401/403 middleware. |
| `BE#48` | Claim-shape `id` vs `userId` drift. |
| `BE#75`/`BE#78`/`BE#84` | RSA per-listing keys, RS256 launch JWTs, JWKS endpoint. |
| `FM#47`/`FM#48`/`FM#50` | Cookie-token → NextAuth sessions; secure cookies + clean logout. |

## See also
- [08 — Security engineering](08-security.md)
- [06 — Error handling & typed errors](06-error-handling.md) — 401/403 mapping.
- [05 — Incremental migration](05-incremental-migration.md) — the NextAuth migration.
