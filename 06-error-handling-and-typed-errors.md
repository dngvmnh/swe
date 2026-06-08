# 06 — Error Handling & Typed Errors

## The idea

Errors are values that travel through a system. Good error handling is about three things:
**classifying** failures into meaningful categories, **propagating** them cleanly from where
they occur to where they're handled, and **translating** them into the right response for each
audience (a status code for a client, a full stack trace for a log).

A **typed error hierarchy** gives each category of failure its own class
(`NotFoundError`, `ValidationError`, `ConflictError`…), so the *type* carries the meaning instead
of a loose string or a magic number passed around by hand.

## Why it exists

In a layered system, the code that *detects* a failure (a DB query, a service) is rarely the code
that *decides what the user sees* (the HTTP layer). If every layer invents its own ad-hoc error
shape, the translation logic gets duplicated, status codes drift, and raw internals leak to
clients. A single typed hierarchy plus one central handler keeps the mapping consistent and in
one place.

## Patterns & trade-offs

- **Error hierarchy.** A base `AppError` with subclasses per category. Each subclass *owns* its
  status code and error code, so an `NotFoundError` is *always* a 404 — the number can't be
  fat-fingered at the throw site.
- **Throw typed, catch central.** Services `throw` typed errors and never touch the
  request/response. A single global error-handling middleware catches everything and maps it to
  the response envelope. This is the backbone of clean propagation.
- **Fail fast with guard clauses.** Validate preconditions at the top of a function and
  throw/return immediately, rather than nesting the happy path inside `if`s.
- **Two audiences, two messages.** Log the *full* error server-side (stack, DB detail); return a
  *sanitized* message to the client. Never leak stack traces or raw DB errors.
- **Fail loud, not silent.** A swallowed error that returns a fake success is worse than a crash —
  it hides the bug and corrupts trust in green checks.
- **Errors as values vs exceptions.** Some languages/styles return `Result`/`Either` types
  instead of throwing. The trade-off is explicitness (every caller must handle it) vs ergonomics
  (exceptions auto-propagate). Pick one convention and hold the line — don't mix
  `return {success:false}` with `throw` in the same layer.

## Pitfalls

- **Bare `throw new Error('...')` in services.** Loses classification — the central handler can't
  map it to anything but 500.
- **Hardcoded status codes at throw sites.** `throw new AppError(msg, 404, code)` scattered around
  invites a wrong number; a `NotFoundError` can't be wrong.
- **Catching a specific failure as a generic one.** E.g. catching a unique-violation as
  `InternalError` (500) when it actually means "already done, return 200" — breaks contracts.
- **Silent fallback on an unchecked read.** Ignoring a query's `error` and using a stale/`null`
  value as if it succeeded.
- **Leaking internals.** Returning the raw DB error message to the client.

## Seen in Mythos

- **The hierarchy itself.** `src/errors/app-error.ts` defines
  `AppError → ValidationError | AuthError | ForbiddenError | NotFoundError | ConflictError |
  PaymentRequiredError | InternalError`. The constitution mandates: *"Services always throw a
  typed subclass. No bare `throw new Error()` in the service layer,"* and a single global error
  middleware maps everything to the `{ success, error, details }` envelope. `BE#35` added
  `InternalError` and re-exported the family through a barrel as part of the TS foundation.

- **The "typed errors" review on `BE#88`.** The first cut threw `new AppError('Session not
  found', 404, 'SESSION_NOT_FOUND')` and `new AppError(..., 409, ...)`. The reviewer flagged each
  as a constitution violation and required `NotFoundError`/`ConflictError`. To keep the specific
  code strings the SDK relies on (`SESSION_NOT_FOUND`, `SESSION_NOT_STARTED`), the subclasses were
  *extended* to accept an optional `code` — the status code stays owned by the class, the domain
  code stays expressive.

- **Catching the *specific* DB error — `BE#88`.** The standout error-handling lesson: the
  `metered_charge` RPC's `23505` unique violation was being caught as a generic `InternalError`
  → 500. The fix special-cases `rpcError.code === '23505'` and replays the original result as a
  200 — translating a low-level DB error into the correct domain outcome. (See
  [doc 01](01-idempotency-and-exactly-once-semantics.md).)

- **Fail loud, not silent — `BE#88` tests.** The integration tests originally did
  `console.warn(...); return;` when the `wallets` table was absent, so they went **green on
  un-run code**. The reviewer required `throw new Error('wallets table missing — apply migration…')`
  so a missing schema fails the build loudly. A green check on code that never ran is a lie.

- **Unchecked read → silent stale value — `BE#88`.** A `.single()` fetch in the idempotency path
  had no error handler; on failure it silently fell back to a stale `metered_credits_charged`. The
  fix adds an error check that throws `InternalError`. Don't trust a read you didn't verify.

- **Webhook silent failures — `BE#18`.** The Stripe webhook handler had failure paths that
  returned without surfacing the problem — "silent failures" that mask money bugs. Part of the
  fix was making those paths fail visibly.

## See also

- [01 — Idempotency](01-idempotency-and-exactly-once-semantics.md)
- [07 — API design & layered architecture](07-api-design-and-layered-architecture.md)
- [10 — Testing strategy](10-testing-strategy-and-quality-gates.md) (fail-loud tests)
