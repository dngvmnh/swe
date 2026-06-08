# 04 — Authentication & Authorization

## The idea

Two different questions that are easy to conflate:

- **Authentication (authn): *who are you?*** Proving identity — a password, a Google login, a
  signed token.
- **Authorization (authz): *what are you allowed to do?*** Deciding whether an authenticated (or
  anonymous) principal may perform an action — roles, ownership, scopes.

You authenticate first, then authorize. A system can authenticate you perfectly and still
(correctly) deny you.

## Why it exists

The moment a system has more than one user and anything worth protecting, it must distinguish
callers and gate actions. Get authn wrong and impostors get in; get authz wrong and legitimate
users reach things they shouldn't (the most common serious web vulnerability class —
"broken access control").

## Patterns & trade-offs

- **Sessions vs tokens.**
  - *Server-side sessions*: the server stores session state, the client holds an opaque cookie.
    Easy to revoke; requires server/shared storage.
  - *Stateless tokens (JWT)*: the token itself carries claims and is signed; the server verifies
    the signature without a lookup. Scales horizontally; **hard to revoke before expiry**, so keep
    lifetimes short.
- **Symmetric vs asymmetric signing.**
  - *HMAC (HS256)*: one shared secret signs and verifies. Simple; every verifier must hold the
    secret.
  - *RSA/ECDSA (RS256)*: a private key signs, a **public** key verifies. Lets third parties verify
    without ever seeing the secret — the basis of **JWKS** (a published set of public keys).
- **OAuth / OpenID Connect.** Delegate authn to a provider (Google) so you never handle the
  user's password. You receive an identity assertion and map it to a local account.
- **Password storage.** Never store plaintext. Use a slow, salted hash (bcrypt/argon2) with a
  sensible cost factor.
- **Required vs optional auth.** Some endpoints demand a valid token (401 if missing); others
  work anonymously but personalize if a token is present — but a *present-but-invalid* token must
  still be rejected, never silently ignored.
- **Cookie security.** `HttpOnly` (no JS access → mitigates XSS token theft), `Secure`
  (HTTPS only), `SameSite` (CSRF mitigation). Clear auth state fully on logout.

## Pitfalls

- **Silently accepting invalid tokens** in "optional auth" — turns optional into a bypass.
- **Trusting client-supplied identity** (a `userId` in the body) instead of the verified token.
- **Long-lived JWTs with no revocation story** — a leaked token is valid until it expires.
- **Mismatched claim shapes** between issuer and verifier (`id` vs `userId`) — a silent 0-results
  or undefined-user bug.
- **Authn without authz** — logged in is not the same as allowed.
- **Secrets in source / client bundles.**

## Seen in Mythos

- **The full authn evolution.** `BE#1` added register + login (password). `BE#4` added Google
  sign-in (OAuth). `BE#21` fixed Google login error handling. The frontend mirrors this across
  `FM#3` (signup), `FM#7` (Google sign-in), `FM#27`/`FM#52` (Google signup redirection fixes).

- **The "optional auth silently ignores invalid JWT" bug — `BE#17`.** `optionalAuthenticateToken`
  was swallowing invalid tokens instead of rejecting present-but-invalid ones — exactly the
  pitfall above. The fix preserves the rule: *no token → anonymous; bad token → 401.* `BE#35`
  later re-encoded these status-code rules into the typed middleware (`authenticateToken` → 401
  no-token / 403 invalid; optional → 401 if present-but-invalid).

- **Asymmetric signing for third-party launch tokens — `BE#75`, `BE#78`, `BE#84`.** Creators'
  web apps need to verify that a launch request really came from Mythos *without* holding a
  Mythos secret. So Mythos generates **per-listing RSA-2048 keys**, stores the private key
  AES-GCM-encrypted, and exposes the **public** keys via JWKS (`GET /.well-known/jwks.json`,
  `BE#84`; per-listing JWKS in `BE#75`). `BE#78` mints an **RS256** launch JWT signed with the
  per-listing private key. This is precisely why RS256 + JWKS exists: verification by parties you
  don't share secrets with.

- **The classic claim-shape mismatch — `BE#48`.** `req.user` exposed `id` but a service read
  `userId`, silently breaking profile lookups. Fixed by aligning the property. A textbook
  issuer/verifier contract drift.

- **Sessions vs tokens, migrated live — `FM#47`, `FM#48`.** The frontend moved from cookie-based
  tokens to **NextAuth.js v5 sessions** (`FM#47`), then migrated the API client from cookie tokens
  to NextAuth sessions (`FM#48`). `FM#50` hardened it: secure cookies in production and clearing
  `userId` on logout — the cookie-security and clean-logout pitfalls, addressed.

- **Auth guards at the edge — `FM#43`.** The Next.js migration added `src/proxy.ts` route guards
  so unauthenticated users can't reach protected routes — authorization enforced before render.

- **Secrets out of source — `BE#16`, `FM#21`.** Removing hardcoded credentials from
  `.env.example` (`BE#16`) and moving a hardcoded Google Client ID into `.env` (`FM#21`).

## See also

- [08 — Security engineering](08-security-engineering.md)
- [06 — Error handling & typed errors](06-error-handling-and-typed-errors.md) (401/403 mapping)
- [05 — Incremental migration](05-incremental-migration-strangler-fig.md) (the NextAuth migration)
