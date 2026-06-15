# Module 08 — Security Engineering

> **Core question:** How do you keep attackers and accidents out?
> **Reference doc:** [../08-security-engineering.md](../08-security-engineering.md)
> **Prerequisites:** [04 — Auth](04-auth.md); pairs with [09 — Payments](09-payments.md) and [12 — Frontend](12-frontend.md) (XSS context).

## Why this module
Anything reachable from the internet *will* be probed. Most breaches exploit a small set of
well-understood, preventable classes (injection, broken access control, secrets in code, missing
transport security). This module gives you the mental model and the controls.

### What you'll be able to do
- Reason in **trust boundaries** and apply "never trust input" + "defense in depth".
- Prevent XSS and injection by validating input and **encoding for context** on output.
- Manage secrets, encrypt in transit and at rest, and apply least privilege.
- Verify (not trust) third-party callbacks like webhooks.

---

## Lesson 8.1 — Trust boundaries, never trust input, defense in depth

**Goal:** adopt the core security mindset.

### Concept
A **trust boundary** is any point where data crosses from a place you don't control (a browser, a
third party, the network) into a place you do. **Every input that crosses a trust boundary is
hostile until proven otherwise.** Two recurring principles:
- **Never trust input** — validate everything from outside.
- **Defense in depth** — layer controls so one failure isn't a breach.

> **Trade-off:** every control adds friction (a stricter CSP can break a feature; rate limits can
> hit legitimate bursts). **Tune, don't disable.**

### ✅ Lesson 8.1 checklist
- [ ] 📖 Read & understood — I can identify the trust boundaries in a system.
- [ ] 💻 Applied — I listed the trust boundaries of an app I work on.
- [ ] 🔍 Found in Mythos — least-privilege DB functions + helmet + rate limits = layered defense.

---

## Lesson 8.2 — Input validation & output encoding (XSS, injection)

**Goal:** stop "data interpreted as code."

### Concept
Validate/sanitize on the way **in**; **encode for the context** on the way **out**. The same string
is safe in one context (plain text) and dangerous in another (HTML, SQL, a shell). **XSS and SQL
injection are both "data interpreted as code."**

### ❌ Bad → ✅ Good — stored XSS
```tsx
// ❌ BAD — user-supplied text injected as raw HTML → stored XSS
<div dangerouslySetInnerHTML={{ __html: activity.message }} />
// a payload like  <img src=x onerror="steal_cookies()">  runs in every viewer's browser
```
```tsx
// ✅ GOOD — render as text, or sanitize before injecting
<div>{activity.message}</div>                                  // React escapes by default
// or, if HTML is genuinely required:
<div dangerouslySetInnerHTML={{ __html: sanitize(activity.message) }} />
```

### ❌ Bad → ✅ Good — SQL injection
```ts
// ❌ BAD — string interpolation into SQL
db.query(`SELECT * FROM users WHERE email = '${email}'`);
// ✅ GOOD — parameterized query; the driver keeps data and code separate
db.query("SELECT * FROM users WHERE email = $1", [email]);
```

### ✅ Lesson 8.2 checklist
- [ ] 📖 Read & understood — I can explain "data interpreted as code" and context-specific encoding.
- [ ] 💻 Applied — I fixed (or wrote) a parameterized query and an escaped/sanitized render.
- [ ] 🔍 Found in Mythos — `FM#22`/`FM#51`: `dangerouslySetInnerHTML` of user text → sanitized before render.

---

## Lesson 8.3 — Secrets management

**Goal:** keep secrets out of everywhere they leak.

### Concept
Secrets live in **environment config or a secrets manager** — never in source, never in the client
bundle, never in logs. Example config files (`.env.example`) ship with **placeholders only**.

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — hardcoded secret in source / shipped to the browser
const GOOGLE_CLIENT_ID = "1234-realid.apps.googleusercontent.com";
// ❌ BAD — real credential committed in .env.example
DATABASE_URL=postgres://admin:hunter2@prod/db
```
```ts
// ✅ GOOD — read from environment; example file has placeholders only
const GOOGLE_CLIENT_ID = process.env.GOOGLE_CLIENT_ID;
// .env.example:  DATABASE_URL=postgres://user:password@host:5432/dbname
```

### ✅ Lesson 8.3 checklist
- [ ] 📖 Read & understood — I can list the places a secret must never appear.
- [ ] 💻 Applied — I moved a hardcoded value into env config with an `.env.example` placeholder.
- [ ] 🔍 Found in Mythos — `BE#16` removed credentials from `.env.example`; `FM#21` moved a hardcoded Google Client ID into `.env`.

---

## Lesson 8.4 — Encryption in transit & at rest

**Goal:** protect sensitive data on the wire and on disk.

### Concept
- **TLS everywhere** (in transit).
- **Sensitive stored data** (private keys, PII) **encrypted at rest** (e.g. AES-GCM).
- A **key-encryption-key (KEK)** protects the data-encryption-keys — you don't store the crown jewels
  in plaintext next to the data.

> Mythos stores per-listing **RSA-2048 private keys AES-GCM-encrypted**, protected by a
> `KEY_ENCRYPTION_KEY`; only the *public* half is exposed via JWKS. Defense in depth for the most
> sensitive stored material.

### ✅ Lesson 8.4 checklist
- [ ] 📖 Read & understood — I can explain encryption at rest and the KEK→DEK idea.
- [ ] 💻 Applied — I described what stored data in my project should be encrypted at rest.
- [ ] 🔍 Found in Mythos — `BE#75` AES-GCM-encrypted private keys + `KEY_ENCRYPTION_KEY`.

---

## Lesson 8.5 — Least privilege, security headers, rate limiting

**Goal:** shrink blast radius and harden the edge.

### Concept
- **Least privilege.** Each component gets the narrowest permissions that work — DB roles, function
  `GRANT`s, IAM scopes. (Mythos RPCs run `SECURITY DEFINER` with a pinned `search_path`, `GRANT`ed
  only to `service_role`.)
- **Security headers.** CSP, HSTS, `X-Content-Type-Options`, frame options instruct the browser to
  refuse classes of attack. `helmet`-style middleware sets sane defaults. Restrict which remote
  origins can load resources/images. **No wildcard CORS in prod.**
- **Rate limiting.** Throttle auth/reset endpoints **hard** (e.g. 10 req / 15 min / IP) to blunt
  brute-force; apply a broad limit everywhere else. Cap body size (e.g. 1 MB JSON).

### ✅ Lesson 8.5 checklist
- [ ] 📖 Read & understood — I can explain least privilege, security headers, and rate limiting.
- [ ] 💻 Applied — I added `helmet`-style headers and a rate limit to an endpoint.
- [ ] 🔍 Found in Mythos — constitution: helmet first, strict limits on `/api/auth/*` & `/api/reset/*`, origin-whitelist CORS; `BE#88`/`BE#80` least-privilege RPCs.

---

## Lesson 8.6 — Verify, don't trust, callbacks (webhooks)

**Goal:** treat third-party callbacks as hostile until verified.

### Concept
Webhooks and redirects from third parties must be **cryptographically verified** *and* their
contents **re-checked against your own records**. A security check wrapped in a **skippable
condition is not a security check.**

### ❌ Bad → ✅ Good
```ts
// ❌ BAD (severity: Critical) — the amount check is skipped if amount_total isn't an integer
if (Number.isInteger(session.amount_total) && session.amount_total !== expected) {
  reject();   // attacker sends a non-integer amount_total → whole check is bypassed
}
```
```ts
// ✅ GOOD — verify signature first, then check amount UNCONDITIONALLY
const event = stripe.webhooks.constructEvent(rawBody, sig, endpointSecret); // throws if forged
const session = event.data.object;
if (session.amount_total !== expected) {                                    // always runs
  throw new ValidationError("amount mismatch");
}
```

### ✅ Lesson 8.6 checklist
- [ ] 📖 Read & understood — I can explain "verify signature + re-check contents unconditionally."
- [ ] 💻 Applied — I verified a webhook signature and added an unconditional amount check.
- [ ] 🔍 Found in Mythos — `BE#18` (Critical): amount verification was inside a skippable guard; made unconditional.

---

## 🎯 Module 08 mastery checklist
- [ ] I can identify trust boundaries and treat all crossing input as hostile.
- [ ] I can prevent XSS (escape/sanitize) and SQL injection (parameterize).
- [ ] I can keep secrets out of source, bundles, and logs.
- [ ] I can encrypt sensitive data at rest with a KEK and explain TLS in transit.
- [ ] I can apply least privilege, security headers, and rate limits.
- [ ] I can verify a webhook's signature and re-check its contents unconditionally.

## 🛠️ Mini-project
1. Take a small app with a comment/notification feature; introduce and then **fix** a stored-XSS
   hole (`dangerouslySetInnerHTML` of user text → escaped/sanitized). Confirm a payload no longer executes.
2. Add `helmet` headers, an origin-whitelist CORS, and a strict rate limit on a login route.
3. Add a webhook receiver that verifies a signature and re-checks the payload amount unconditionally;
   test that a forged signature and a tampered amount are both rejected.

## 🔗 Mythos PR map
| PR | What it demonstrates |
|----|----------------------|
| `FM#22`/`FM#51` | Stored XSS via `dangerouslySetInnerHTML` → sanitize. |
| `BE#18` | Webhook amount-verification bypass (Critical). |
| `BE#75` | AES-GCM encryption at rest for signing keys. |
| `FM#53`/`FM#54` | Security headers + restricted image remote patterns. |
| `BE#16`, `FM#21` | Secrets out of source. |
| `FM#50` | Cookie hardening + clean logout. |

## See also
- [04 — Authentication & authorization](04-auth.md)
- [09 — Payments & billing](09-payments.md)
- [12 — Frontend architecture & rendering](12-frontend.md) — XSS in the render path.
