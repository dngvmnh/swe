# 08 — Security Engineering

## The idea

Security engineering is designing systems that keep their guarantees even when someone is
*actively trying to break them* — and when honest mistakes happen. The mental model is the
**trust boundary**: any point where data crosses from a place you don't control (a browser, a
third party, the network) into a place you do. Every input that crosses a trust boundary is
hostile until proven otherwise.

Two recurring principles: **never trust input**, and **defense in depth** (layer controls so one
failure isn't a breach).

## Why it exists

Anything reachable from the internet *will* be probed. The cost of a breach — stolen credentials,
drained wallets, leaked PII, XSS hijacking every visitor's session — is far higher than the cost
of the controls. Most breaches exploit a small set of well-understood, preventable classes
(injection, broken access control, secrets in code, missing transport security).

## Patterns & trade-offs

- **Input validation & output encoding.** Validate/sanitize on the way in; *encode for the
  context* on the way out. The same string is safe in one context (plain text) and dangerous in
  another (HTML, SQL, a shell). XSS and SQL injection are both "data interpreted as code."
- **Secrets management.** Secrets live in environment config or a secrets manager, never in
  source, never in the client bundle, never in logs. Example config files (`.env.example`) ship
  with placeholders only.
- **Encryption in transit and at rest.** TLS everywhere; sensitive stored data (private keys,
  PII) encrypted at rest (e.g. AES-GCM). Key-encryption-keys protect the data-encryption-keys.
- **Least privilege.** Each component gets the narrowest permissions that work — DB roles,
  function `GRANT`s, IAM scopes.
- **Security headers & transport hardening.** HTTP response headers (CSP, HSTS, `X-Content-Type-
  Options`, frame options) instruct the browser to refuse classes of attack. `helmet`-style
  middleware sets sane defaults. Restrict which remote origins can load resources/images.
- **Rate limiting.** Throttle auth and reset endpoints hard to blunt brute-force and abuse;
  apply a broad limit everywhere else.
- **Signed, verifiable tokens.** Use asymmetric signatures so third parties verify without your
  secret (see [doc 04](04-authentication-and-authorization.md)).
- **Verify, don't trust, callbacks.** Webhooks and redirects from third parties must be
  cryptographically verified *and* their contents re-checked against your own records.

Trade-off: every control adds friction (stricter CSP can break a feature; rate limits can hit
legitimate bursts). Tune, don't disable.

## Pitfalls

- **`dangerouslySetInnerHTML` / raw HTML injection of user data** → stored XSS.
- **Trusting webhook metadata** because "we set it ourselves" — without verifying amounts/signature
  an attacker may replay or tamper.
- **Conditional security checks** that can be skipped when a field is absent or oddly typed.
- **Secrets committed to the repo** or shipped to the browser.
- **Insecure cookies** (missing `Secure`/`HttpOnly`/`SameSite`) and auth state lingering after
  logout.
- **Wildcard CORS / permissive remote patterns** in production.

## Seen in Mythos

- **Stored XSS via `dangerouslySetInnerHTML` — `FM#22`/`FM#51`.** The most instructive security
  PR. Notification and profile text — *originating from user input* — was injected as raw HTML in
  several spots (e.g. `latest-activities.tsx`), so a payload like
  `<img src=x onerror="steal_cookies()">` would execute in every viewer's browser. The fix
  sanitizes the HTML before render. Classic "data interpreted as code" across a trust boundary.

- **Webhook amount-verification bypass — `BE#18` (severity: Critical).** The Stripe webhook only
  compared the charged amount to the expected amount *inside* a guard:
  `if (Number.isInteger(session.amount_total) && session.amount_total !== expected) reject`. If
  `amount_total` wasn't an integer the check was skipped entirely — a bypass on a money path.
  Plus "silent failures" hid problems. The lesson: a security check wrapped in a skippable
  condition is not a security check. (See also [doc 09](09-payments-and-billing-architecture.md).)

- **Encryption at rest for signing keys — `BE#75`.** Per-listing RSA-2048 private keys are stored
  **AES-GCM encrypted**, protected by a `KEY_ENCRYPTION_KEY`; only the *public* half is exposed via
  JWKS. Defense in depth for the most sensitive stored material.

- **Security headers & remote patterns — `FM#53`/`FM#54`.** Adds HTTP response headers and
  restricts image remote patterns on the Next.js app — browser-enforced defense in depth.

- **Secrets out of source — `BE#16`, `FM#21`.** Removed hardcoded credentials from `.env.example`
  (`BE#16`) and moved a hardcoded Google Client ID into `.env` (`FM#21`).

- **Cookie hardening & clean logout — `FM#50`.** Secure cookies in production and clearing `userId`
  on logout — directly addressing the insecure-cookie / lingering-state pitfalls.

- **Rate limiting & helmet as policy.** The constitution requires `helmet()` as the first
  middleware, strict rate limits on `/api/auth/*` and `/api/reset/*` (10 req/15 min/IP), a broad
  limit on all `/api/*`, a 1 MB JSON body cap, and an origin-whitelist CORS (no wildcard in prod).

- **Least privilege on DB functions — `BE#88`, `BE#80`.** RPCs run `SECURITY DEFINER` with a pinned
  `search_path` and are `GRANT`ed only to `service_role`.

## See also

- [04 — Authentication & authorization](04-authentication-and-authorization.md)
- [09 — Payments & billing architecture](09-payments-and-billing-architecture.md)
- [12 — Frontend architecture & rendering](12-frontend-architecture-and-rendering.md) (XSS context)
