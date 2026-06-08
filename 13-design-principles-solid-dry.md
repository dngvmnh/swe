# 13 — Design Principles: SOLID, DRY & Clean Code

## The idea

Design principles are heuristics for writing code that is **easy to change** — because most of a
program's lifetime cost is in changing it, not writing it the first time. They don't make code
"work" (tests do that); they make code *survivable* six months later, for the next person, who is
often you.

The famous bundles:

- **SOLID** — five object/module design principles.
- **DRY** — Don't Repeat Yourself.
- **Clean code heuristics** — small functions, guard clauses, good names, immutability, purity.

## Why it exists

Code is read far more than it's written, and changed far more than it's read once. Tangled,
duplicated, deeply nested code makes every change risky: you can't predict what a edit will break.
These principles are accumulated wisdom about how to keep the cost of the *next* change low.

## The principles

### SOLID
- **S — Single Responsibility.** A module/function has one reason to change. If a function's name
  needs an "and," split it.
- **O — Open/Closed.** Open for extension, closed for modification. Add new behavior as a new
  function/module rather than bolting an `if (type === 'new')` branch into existing code.
- **L — Liskov Substitution.** A subtype must be usable anywhere its base type is. Wrap external
  providers (email, storage) behind an interface so swapping one changes only the adapter.
- **I — Interface Segregation.** Depend on narrow interfaces; don't force callers to know about
  fields/methods they don't use. Pass `Pick<User,'id'|'email'>`, not the whole `User`.
- **D — Dependency Inversion.** Depend on abstractions, not concretions. Call a factory
  (`initSupabase()`), don't `new Client()` inline — so the dependency can be swapped/mocked.

### DRY & clean code
- **DRY.** Each piece of knowledge has one authoritative home. Duplication is a bug waiting to
  happen — fixes and features applied to one copy silently miss the other. (But beware *false* DRY:
  don't couple two things that merely look alike today.)
- **Guard clauses.** Check preconditions at the top and return/throw immediately; keep nesting
  shallow (≤2 levels). The happy path reads top-to-bottom.
- **Small functions.** A function that exceeds a threshold (e.g. ~50 lines) is doing too much —
  treat length as a *hard refactor signal* and extract named helpers.
- **Pure functions.** Data transforms, formatting, and template rendering should be side-effect-
  free — trivially testable and reusable.
- **Immutability.** Prefer `const`; never mutate function arguments; return new objects.
- **Named constants.** No magic strings/numbers — give them names in a constants module.
- **Naming conventions.** Consistent, intention-revealing names (verbs for functions, `is*`/`has*`
  for booleans) so code is self-documenting.

## Pitfalls

- **God functions/modules** that do everything and change for many reasons.
- **Copy-paste programming** → divergent duplicates (the DRY violation).
- **Over-abstraction** → indirection nobody needs, "DRY" that couples unrelated code, premature
  generality. Principles are heuristics, not laws.
- **Deep nesting** instead of guard clauses → unreadable control flow.
- **Magic values** and inconsistent names → code you must decode rather than read.

## Seen in Mythos

These principles aren't aspirational here — they're written into the **Engineering Constitution**
and *enforced in review*.

- **DRY, caught and fixed — `FM#20`.** `uploadFileToS3` was defined **identically in two files**
  (`listing-service.ts` and `profile-service.ts`). The PR's own rationale states the principle:
  "When the same logic is written in two places, they inevitably diverge… a bug fix applied to one
  copy won't be applied to the other." Deduplicated into one shared utility. The constitution
  generalizes it: *logic used in ≥2 places belongs in `src/utils/`.*

- **Single Responsibility + function length — `BE#88`.** The reviewer flagged `meterSession` at
  ~82 lines as a constitution violation ("Service functions ≤ 50 lines… a hard refactor signal")
  and required extracting `resolveIdempotentCharge()` and `callMeteredChargeRpc()` — each now does
  one thing. A live application of "if the function is too long, it's doing too much."

- **Dependency Inversion as policy — constitution + everywhere.** Services call factory singletons
  (`initSupabase()`, `getStripeClient()`, `getResend()`) and *never* `new Client()` inline — so the
  dependency can be swapped or mocked. Adapters for email/storage live in `src/utils/` / `src/aws/`
  (Liskov): swapping a provider touches only the adapter.

- **Interface Segregation in the type migration — `BE#35`, `BE#31`–`#47`.** The TS fan-out's whole
  premise was *narrow typed contracts* per boundary — `Request.user` tightened to exactly its
  fields; DTOs per domain rather than fat shared objects.

- **Guard clauses & typed errors — `BE#88`.** The refactored `consumeSession`/`meterSession` are
  pure top-of-function guard clauses (fetch → `if (!session) throw NotFoundError` → `if (...) throw
  ConflictError` → happy path), max two levels of nesting.

- **Named constants & immutability — constitution.** Magic strings/numbers live in `src/constants/`
  (`LAUNCH_TOKEN_EXPIRY_SECONDS`, etc.); `const` everywhere; no mutation of arguments;
  `console.*` and bare `throw new Error()` in new code are explicit **blocking** review comments.

- **Open/Closed in the email layer — `BE#35`.** Email templates are pure functions returning
  `{ subject, html }`; adding a new email is a new template module, not an edit to a switch.

## See also

- [06 — Error handling & typed errors](06-error-handling-and-typed-errors.md)
- [07 — API design & layered architecture](07-api-design-and-layered-architecture.md)
- [15 — Spec-driven development & governance](15-spec-driven-development-and-governance.md)
