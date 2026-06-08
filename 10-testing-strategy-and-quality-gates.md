# 10 — Testing Strategy & Quality Gates

## The idea

Tests are executable claims about how your system behaves. A **testing strategy** decides *what
kinds* of tests to write, *how many* of each, and *where* they run, so you get the most confidence
per unit of effort. **Quality gates** are the automated checks (tests, type-checks, lint) that a
change must pass before it can merge — turning "we should test" into "you literally cannot ship a
regression."

## Why it exists

Manual testing doesn't scale and doesn't repeat. As a codebase grows, the only way to change it
confidently is to have a machine re-verify the old behavior on every change. Without gates, good
intentions decay; with gates, the standard is enforced uniformly regardless of deadline pressure.

## Patterns & trade-offs

- **The test pyramid.** Many fast **unit** tests (a function in isolation), fewer **integration**
  tests (several real components together), fewer still **end-to-end** tests (the whole system).
  Higher = more realistic but slower, flakier, costlier. Bias toward the base, but cover critical
  flows higher up.
- **Unit vs integration — and when to prefer integration.** For logic with branching, unit tests
  are cheap and precise. For flows whose correctness *is* the interaction with a real database
  (transactions, constraints, RPCs, idempotency), an integration test against a **real** dependency
  catches what a mock would hide.
- **Mock at the network boundary, not the database.** Mock third parties you don't own (Stripe,
  email) at the HTTP boundary so tests are deterministic and don't make live calls. But mocking
  *your own database* hides exactly the constraint/transaction behavior you most need to verify —
  use a dedicated test DB instead.
- **Arrange–act–assert + self-contained fixtures.** Each test seeds the data it needs and cleans
  up after itself, so suites are order-independent and re-runnable.
- **Test the invariants, not just the happy path.** Negative cases (404/409/402), idempotency
  replays, and concurrency safety are where the bugs live — especially on auth and payment flows.
- **Fail loud.** A test that can't verify its precondition must *fail*, not skip silently. A green
  run on un-exercised code is worse than a red one — it manufactures false confidence.
- **Quality gates in CI.** Run tests + type-check + lint on every PR; failures block merge. Encode
  non-negotiables (no `console.*` in new code, no bare `throw new Error()`) as review/lint gates.

Trade-off: tests are code you also maintain. Over-testing trivial glue, or pinning brittle
implementation details, creates drag. Test behavior and contracts, not internals.

## Pitfalls

- **Mocking the database** → tests pass while real constraints/transactions break in prod.
- **Silent skips** (`if (!precondition) return;`) → false green. (See `BE#88` below.)
- **Order-dependent / shared-state tests** → flaky, non-reproducible failures.
- **Only happy-path coverage** on money/auth flows → the dangerous cases ship untested.
- **Slow E2E for everything** → a suite so slow nobody runs it.
- **Asserting implementation details** → tests break on every refactor and get deleted in
  frustration.

## Seen in Mythos

- **Real-DB integration testing as policy.** The constitution: *integration tests boot the Express
  app and hit real HTTP against a dedicated Supabase test project — do not mock the DB; mock
  external services (Stripe, Resend, S3) at the network boundary.* That's exactly the
  "mock the boundary, not your DB" rule.

- **Invariant-driven suite — `BE#70`.** The wallet PR shipped **12 focused integration tests**
  targeting the invariants, not the happy path: free-plan auto-enrollment, `wallet_credit` happy
  path, **idempotency on `stripe_event_id`**, drain order (subscription first), **overdraft
  rejection**, **concurrent debit safety (no balance corruption)**, and `subscription_renew`
  idempotency. The risky behaviors are pinned by tests.

- **Negative-case coverage on a money flow — `BE#88`.** 12 tests spanning happy path plus every
  error code (400 validation, 404 `SESSION_NOT_FOUND`, 409 `SESSION_NOT_STARTED`/
  `ALREADY_CONSUMED`, 402 `INSUFFICIENT_FUNDS`) and the **`charge_id` replay** asserting *no new
  debit and an unchanged balance* — testing the idempotency contract directly.

- **The fail-loud lesson — `BE#88` review.** The suite originally did
  `if (!hasWallet) { console.warn(...); return; }`, so a missing `wallets` table produced a green
  pass on code that never ran. The reviewer required
  `throw new Error('wallets table missing — apply migration…')`. When we actually ran the suite it
  passed **12/12** — and crucially the loud guard *didn't* trip, proving the migration was applied.

- **Unit-test boilerplate, staged separately — `BE#30` (DON'T MERGE).** Jest unit-test scaffolding
  for small services, kept on its own branch — building the testing foundation deliberately.

- **Manual/contract testing too — `BE#51`.** A Postman integration-test collection complements the
  automated suite for endpoint-level contract checks.

- **A real quality-gate story — `BE#88` toolchain.** Running the suite surfaced a `jest@30` /
  `ts-jest@29` incompatibility that prevents the runner from even starting — a reminder that the
  *gate itself* (the toolchain) is part of the system and must be kept healthy, or CI gives
  meaningless results.

## See also

- [06 — Error handling & typed errors](06-error-handling-and-typed-errors.md) (fail loud)
- [01 — Idempotency](01-idempotency-and-exactly-once-semantics.md) (the replay test)
- [15 — Spec-driven development & governance](15-spec-driven-development-and-governance.md) (gates)
