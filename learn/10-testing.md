# Module 10 — Testing Strategy & Quality Gates

> **Core question:** How do you trust a change before shipping it?
> **Reference doc:** [../10-testing-strategy-and-quality-gates.md](../10-testing-strategy-and-quality-gates.md)
> **Prerequisites:** [06 — Error handling](06-error-handling.md) (fail-loud), [01 — Idempotency](01-idempotency.md) (the replay test).

## Why this module
Manual testing doesn't scale and doesn't repeat. As a codebase grows, the only way to change it
confidently is to have a machine re-verify old behavior on every change. This module teaches you
*what kinds* of tests to write, *where*, and how to turn "we should test" into "you literally
cannot ship a regression."

### What you'll be able to do
- Apply the test pyramid and choose unit vs integration deliberately.
- Mock at the network boundary but use a **real** database.
- Write self-contained, order-independent tests that target invariants.
- Make tests **fail loud** and wire quality gates into CI.

---

## Lesson 10.1 — Tests as executable claims; the pyramid

**Goal:** allocate effort across test types.

### Concept
Tests are **executable claims** about behavior. The **test pyramid:**
```
        /\        E2E         few   — whole system, realistic, slow, flaky, costly
       /  \       integration fewer — several real components together
      /____\      unit        many  — one function in isolation, fast & precise
```
Bias toward the base, but **cover critical flows higher up.**

### ✅ Lesson 10.1 checklist
- [ ] 📖 Read & understood — I can describe the pyramid and the cost/realism trade-off.
- [ ] 💻 Applied — I classified a project's tests by layer.
- [ ] 🔍 Found in Mythos — `BE#30` staged unit-test scaffolding; `BE#70`/`BE#88` shipped integration suites.

---

## Lesson 10.2 — Unit vs integration (and when integration wins)

**Goal:** know which to reach for.

### Concept
- **Unit** tests are cheap and precise for **logic with branching** (a pure function, a drain-order
  calc).
- **Integration** tests win for flows whose **correctness *is* the interaction with a real
  database** — transactions, constraints, RPCs, idempotency. A mock would *hide* exactly the
  behavior you most need to verify.

> Rule: if the bug you fear lives in a `UNIQUE` constraint or a `FOR UPDATE`, only a **real DB**
> test can catch it.

### ✅ Lesson 10.2 checklist
- [ ] 📖 Read & understood — I can decide unit vs integration for a given behavior.
- [ ] 💻 Applied — I wrote one unit test (pure logic) and one integration test (DB behavior).
- [ ] 🔍 Found in Mythos — `BE#70`/`BE#88` test constraints/idempotency against a real Supabase test project.

---

## Lesson 10.3 — Mock the network boundary, not the database

**Goal:** make tests deterministic without hiding real behavior.

### Concept
- **Mock third parties you don't own** (Stripe, email, S3) at the **HTTP boundary** — tests stay
  deterministic and don't make live calls.
- **Don't mock your own database** — that hides the constraint/transaction behavior you most need to
  verify. Use a **dedicated test DB** instead.

```ts
// ✅ mock Stripe at the network boundary; hit a REAL test database
jest.mock("stripe");                                  // external — mock it
const db = createClient(process.env.TEST_SUPABASE_URL!, ...); // your DB — real test instance
```

### ✅ Lesson 10.3 checklist
- [ ] 📖 Read & understood — I can state "mock the boundary, not your DB."
- [ ] 💻 Applied — I set up a test DB and mocked one external service.
- [ ] 🔍 Found in Mythos — constitution: integration tests boot Express + hit real HTTP against a dedicated Supabase test project; mock Stripe/Resend/S3 at the network boundary.

---

## Lesson 10.4 — AAA + self-contained fixtures

**Goal:** keep suites order-independent and re-runnable.

### Concept
**Arrange–Act–Assert.** Each test **seeds the data it needs and cleans up after itself**, so suites
are order-independent and re-runnable.

```ts
test("debit drains subscription bucket first", async () => {
  // Arrange — seed exactly what this test needs
  const user = await seedUser({ subscription: 100, topup: 50 });
  // Act
  const res = await debit(user.id, 120, "ref-1");
  // Assert
  expect(res).toEqual({ subscription: 0, topup: 30 });
  // (teardown removes this user so other tests are unaffected)
});
```

### ❌ Bad → ✅ Good
- ❌ Test B depends on rows Test A left behind → flaky, non-reproducible, order-dependent.
- ✅ Each test seeds + tears down its own fixture.

### ✅ Lesson 10.4 checklist
- [ ] 📖 Read & understood — I can explain AAA and self-contained fixtures.
- [ ] 💻 Applied — I wrote an order-independent test that seeds and cleans up.
- [ ] 🔍 Found in Mythos — the wallet suites seed per-test data and assert specific balances.

---

## Lesson 10.5 — Test invariants, not just the happy path; fail loud

**Goal:** cover where the bugs actually live.

### Concept
Negative cases (404/409/402), **idempotency replays**, and **concurrency safety** are where bugs
live — especially on auth and payment flows. And a test that can't verify its precondition must
**fail**, not skip silently. **A green run on un-exercised code is worse than a red one — it
manufactures false confidence.**

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — false green: a missing table makes the test pass on code that never ran
if (!hasWalletsTable) { console.warn("no wallets table"); return; }
```
```ts
// ✅ GOOD — fail loud so the gate is meaningful
if (!hasWalletsTable) throw new Error("wallets table missing — apply migration before testing");
```
```ts
// ✅ the idempotency contract, tested directly
test("same charge_id does not double-debit", async () => {
  const before = await balance(user.id);
  await meter({ charge_id: "c1", credits: 10 });
  await meter({ charge_id: "c1", credits: 10 });        // replay
  expect(await balance(user.id)).toBe(before - 10);     // debited ONCE
});
```

### ✅ Lesson 10.5 checklist
- [ ] 📖 Read & understood — I can explain why false-green skips are worse than failures.
- [ ] 💻 Applied — I wrote a negative-case test and an idempotency-replay test.
- [ ] 🔍 Found in Mythos — `BE#88` 12 tests across every error code + a `charge_id` replay; the `console.warn; return` → `throw` fix.

---

## Lesson 10.6 — Quality gates in CI

**Goal:** enforce the standard automatically.

### Concept
Run **tests + type-check + lint on every PR; failures block merge.** Encode non-negotiables (no
`console.*` in new code, no bare `throw new Error()`) as review/lint gates. **The gate itself (the
toolchain) is part of the system** — keep it healthy or CI gives meaningless results.

> Mythos hit a `jest@30` / `ts-jest@29` incompatibility that prevented the runner from even
> starting — a reminder that a broken toolchain = a broken gate.

### ✅ Lesson 10.6 checklist
- [ ] 📖 Read & understood — I can explain quality gates and why the toolchain must stay healthy.
- [ ] 💻 Applied — I added a CI step that runs tests + type-check + lint and blocks on failure.
- [ ] 🔍 Found in Mythos — `BE#88` surfaced a jest/ts-jest incompatibility; the constitution lists blocking gates.

---

## 🎯 Module 10 mastery checklist
- [ ] I can apply the test pyramid and justify unit vs integration per behavior.
- [ ] I can mock external services at the boundary while using a real test DB.
- [ ] I can write AAA, self-contained, order-independent tests.
- [ ] I can test invariants (negative cases, idempotency, concurrency), not just the happy path.
- [ ] I can make tests fail loud and wire quality gates into CI.

## 🛠️ Mini-project
Take your wallet from Module 09 and build its test suite:
1. Unit-test the drain-order calc (pure function).
2. Integration-test against a real test DB: happy credit/debit, **overdraft → 402**, **same `ref`
   debits once**, and **two concurrent debits don't corrupt the balance**.
3. Mock Stripe at the boundary for a webhook-credit test.
4. Add a precondition guard that **throws** (not skips) if the schema isn't applied.
5. Wire a CI workflow that runs tests + type-check + lint and blocks merge on failure.

## 🔗 Mythos PR map
| PR | What it demonstrates |
|----|----------------------|
| `BE#70` | 12 invariant-driven integration tests (drain order, overdraft, idempotency, concurrency). |
| `BE#88` | Negative-case coverage + `charge_id` replay + the fail-loud fix + the toolchain-as-gate lesson. |
| `BE#30` | Unit-test scaffolding staged on its own branch. |
| `BE#51` | Postman collection for endpoint contract checks. |

## See also
- [06 — Error handling & typed errors](06-error-handling.md) — fail loud.
- [01 — Idempotency](01-idempotency.md) — the replay test.
- [15 — Spec-driven development & governance](15-spec-governance.md) — gates.
