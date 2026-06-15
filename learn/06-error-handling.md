# Module 06 — Error Handling & Typed Errors

> **Core question:** How do failures travel through a system cleanly?
> **Reference doc:** [../06-error-handling-and-typed-errors.md](../06-error-handling-and-typed-errors.md)
> **Prerequisites:** [07 — Layered architecture](07-api-design.md) helps (where errors are thrown vs handled).

## Why this module
The code that *detects* a failure (a query, a service) is rarely the code that *decides what the
user sees* (the HTTP layer). If every layer invents its own error shape, translation logic
duplicates, status codes drift, and raw internals leak to clients. This module gives you one clean
way to classify, propagate, and translate failures.

### What you'll be able to do
- Design a typed error hierarchy where the *class* carries the meaning.
- Throw typed errors in services and map them centrally — once.
- Write fail-fast guard clauses and fail-*loud* code.
- Translate a specific low-level error (a DB code) into the correct domain outcome.

---

## Lesson 6.1 — Errors are values: classify, propagate, translate

**Goal:** frame error handling as three jobs.

### Concept
Good error handling is three things:
1. **Classify** failures into meaningful categories (not-found vs validation vs conflict).
2. **Propagate** them cleanly from where they occur to where they're handled.
3. **Translate** them into the right form for each audience: a *status code* for a client, a *full
   stack trace* for a log.

### ✅ Lesson 6.1 checklist
- [ ] 📖 Read & understood — I can name the three jobs of error handling.
- [ ] 💻 Applied — I categorized the failure modes of one endpoint.
- [ ] 🔍 Found in Mythos — the `{ success, error, details }` envelope is the single translation point.

---

## Lesson 6.2 — A typed error hierarchy (the class owns the status)

**Goal:** make a wrong status code *impossible* at the throw site.

### Concept
A base `AppError` with a subclass per category. **Each subclass owns its status code and error
code**, so a `NotFoundError` is *always* a 404 — the number can't be fat-fingered where you throw.

```ts
// src/errors/app-error.ts
export class AppError extends Error {
  constructor(message: string, public status: number, public code: string) { super(message); }
}
export class ValidationError    extends AppError { constructor(m: string, code = "VALIDATION")     { super(m, 400, code); } }
export class AuthError          extends AppError { constructor(m: string, code = "UNAUTHENTICATED"){ super(m, 401, code); } }
export class ForbiddenError     extends AppError { constructor(m: string, code = "FORBIDDEN")      { super(m, 403, code); } }
export class NotFoundError      extends AppError { constructor(m: string, code = "NOT_FOUND")      { super(m, 404, code); } }
export class ConflictError      extends AppError { constructor(m: string, code = "CONFLICT")       { super(m, 409, code); } }
export class PaymentRequiredError extends AppError { constructor(m: string, code = "PAYMENT")      { super(m, 402, code); } }
export class InternalError      extends AppError { constructor(m: string, code = "INTERNAL")       { super(m, 500, code); } }
```

> Note the **optional `code`**: the *status* stays owned by the class, but the domain code stays
> expressive (`SESSION_NOT_FOUND`, `SESSION_NOT_STARTED`) so SDKs can branch on it.

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — hardcoded status at the throw site invites a wrong number
throw new AppError("Session not found", 404, "SESSION_NOT_FOUND");
// ❌ even worse — loses classification entirely; central handler can only map to 500
throw new Error("Session not found");
```
```ts
// ✅ GOOD — the class guarantees the 404; the domain code stays expressive
throw new NotFoundError("Session not found", "SESSION_NOT_FOUND");
```

### ✅ Lesson 6.2 checklist
- [ ] 📖 Read & understood — I can explain why the class should own the status code.
- [ ] 💻 Applied — I wrote a small `AppError` hierarchy.
- [ ] 🔍 Found in Mythos — `BE#88` review required `NotFoundError`/`ConflictError` over `new AppError(…, 404/409, …)`; `BE#35` added `InternalError`.

---

## Lesson 6.3 — Throw typed, catch central

**Goal:** keep services out of the HTTP layer.

### Concept
**Services `throw` typed errors and never touch req/res.** A single global error-handling middleware
catches everything and maps it to the response envelope. This is the backbone of clean propagation —
the mapping lives in exactly one place.

```ts
// service layer — pure logic, no HTTP
async function getProfile(id: string) {
  const profile = await repo.find(id);
  if (!profile) throw new NotFoundError("profile not found");
  return profile;
}

// ONE global error middleware — the only place that knows about HTTP status
app.use((err, _req, res, _next) => {
  if (err instanceof AppError) {
    return res.status(err.status).json({ success: false, error: err.code, details: err.message });
  }
  console.error(err);                                   // log the FULL error server-side
  res.status(500).json({ success: false, error: "INTERNAL" }); // sanitized for the client
});
```

### ✅ Lesson 6.3 checklist
- [ ] 📖 Read & understood — I can explain "throw typed, catch central."
- [ ] 💻 Applied — I wrote a global error middleware that maps typed errors to an envelope.
- [ ] 🔍 Found in Mythos — constitution: "Services always throw a typed subclass. No bare `throw new Error()` in the service layer."

---

## Lesson 6.4 — Fail fast with guard clauses

**Goal:** keep the happy path flat and readable.

### Concept
Validate preconditions at the top of a function and **throw/return immediately**, rather than
nesting the happy path inside `if`s.

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — deep nesting; the happy path is buried
function consume(session) {
  if (session) {
    if (session.startedAt) {
      if (!session.consumedAt) {
        return doConsume(session);
      } else { throw new ConflictError("already consumed"); }
    } else { throw new ConflictError("not started"); }
  } else { throw new NotFoundError("session not found"); }
}
```
```ts
// ✅ GOOD — guard clauses; happy path reads top-to-bottom, ≤2 levels deep
function consume(session) {
  if (!session)            throw new NotFoundError("session not found", "SESSION_NOT_FOUND");
  if (!session.startedAt)  throw new ConflictError("not started", "SESSION_NOT_STARTED");
  if (session.consumedAt)  throw new ConflictError("already consumed", "ALREADY_CONSUMED");
  return doConsume(session);
}
```

### ✅ Lesson 6.4 checklist
- [ ] 📖 Read & understood — I can convert nested conditionals into guard clauses.
- [ ] 💻 Applied — I refactored a nested function to guard-clause style.
- [ ] 🔍 Found in Mythos — `BE#88` `consumeSession`/`meterSession` are guard-clause shaped, max 2 levels.

---

## Lesson 6.5 — Two audiences; fail loud, not silent

**Goal:** never hide a bug behind a fake success.

### Concept
- **Two audiences, two messages.** Log the *full* error server-side (stack, DB detail); return a
  *sanitized* message to the client. **Never leak stack traces or raw DB errors.**
- **Fail loud, not silent.** A swallowed error that returns a fake success is **worse than a crash**
  — it hides the bug and corrupts trust in green checks.

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — unchecked read silently falls back to a stale value
const { data } = await db.from("wallets").select("credits").single();
return data?.credits ?? staleValue;   // on error, you silently use stale data
```
```ts
// ✅ GOOD — verify the read; fail loud
const { data, error } = await db.from("wallets").select("credits").single();
if (error) throw new InternalError("failed to read wallet");
return data.credits;
```

### ✅ Lesson 6.5 checklist
- [ ] 📖 Read & understood — I can explain log-full / return-sanitized and "fail loud".
- [ ] 💻 Applied — I added an error check to a read I previously left unchecked.
- [ ] 🔍 Found in Mythos — `BE#88` tests that did `console.warn; return` (false green) were changed to `throw`; `BE#18` removed Stripe webhook silent failures.

---

## Lesson 6.6 — Translate the *specific* error to the right outcome

**Goal:** turn a low-level code into the correct domain result.

### Concept
Don't catch a *specific* failure as a *generic* one. A unique-violation (`23505`) doesn't mean
"server error" — it means "already done, return the original." Special-case it. (This is the bridge
to [Module 01](01-idempotency.md).)

```ts
// ✅ translate 23505 (unique_violation) into the correct domain outcome
try {
  return await db.rpc("metered_charge", { charge_id, credits });
} catch (e: any) {
  if (e.code === "23505") return await readLedgerRow(charge_id);  // already done → replay 200
  throw e;                                                         // anything else → genuine error
}
```

> **Errors-as-values vs exceptions:** some styles return `Result`/`Either` instead of throwing.
> Trade-off: explicitness (every caller must handle) vs ergonomics (exceptions auto-propagate). Pick
> one convention **and hold the line** — don't mix `return {success:false}` with `throw` in one layer.

### ✅ Lesson 6.6 checklist
- [ ] 📖 Read & understood — I can explain why catching a specific error generically breaks contracts.
- [ ] 💻 Applied — I special-cased one known error code into a domain outcome.
- [ ] 🔍 Found in Mythos — `BE#88` special-cases `rpcError.code === '23505'` and replays as 200.

---

## 🎯 Module 06 mastery checklist
- [ ] I can design a typed error hierarchy where the class owns the status code.
- [ ] I can implement "throw typed in services, map centrally in one middleware."
- [ ] I can convert nested conditionals into guard clauses.
- [ ] I can log full / return sanitized, and make code fail loud instead of silent.
- [ ] I can translate a specific low-level error (e.g. `23505`) into the correct domain outcome.

## 🛠️ Mini-project
1. Build an `AppError` hierarchy (`ValidationError`, `NotFoundError`, `ConflictError`, `InternalError`).
2. Wire a single global error middleware that maps them to `{ success, error, details }` and logs the
   full error server-side while returning a sanitized message.
3. Write a service with guard clauses that throws the right typed error for each precondition.
4. Add a path that catches a specific DB error code and translates it into a domain outcome; test it.

## 🔗 Mythos PR map
| PR | What it demonstrates |
|----|----------------------|
| `BE#35` | Added `InternalError`; re-exported the error family as the TS foundation. |
| `BE#88` | Typed-error review (NotFound/Conflict), `23505`→200 replay, fail-loud tests, unchecked-read fix. |
| `BE#18` | Removing Stripe webhook silent failures. |

## See also
- [01 — Idempotency](01-idempotency.md) — the `23505` replay.
- [07 — API design & layered architecture](07-api-design.md) — where to throw vs handle.
- [10 — Testing strategy](10-testing.md) — fail-loud tests.
