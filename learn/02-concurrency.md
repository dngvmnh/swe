# Module 02 — Concurrency, Locking & Race Conditions

> **Core question:** What breaks when two requests arrive at once?
> **Reference doc:** [../02-concurrency-locking-and-race-conditions.md](../02-concurrency-locking-and-race-conditions.md)
> **Prerequisites:** [01 — Idempotency](01-idempotency.md) (the unique-constraint referee).

## Why this module
"It worked when I tested it alone" is not a guarantee. The moment two requests touch the same row,
timing decides correctness — and timing is something you don't control. This module teaches you to
see races and close them.

### What you'll be able to do
- Recognize the **check-then-act** shape that causes most races.
- Fold a check into a write so the database evaluates them atomically.
- Choose between **optimistic** and **pessimistic** concurrency control.
- Use `SELECT … FOR UPDATE` and unique constraints correctly, and avoid deadlocks.

---

## Lesson 2.1 — What a race condition is

**Goal:** name the bug shape so you can spot it everywhere.

### Concept
A **race condition** is a bug where correctness depends on the *timing* of events you don't control.
The classic shape is **check-then-act**: you *read* state, *decide* based on it, then *write* — but
between the read and the write, someone else changed the state your decision relied on.

```
Request A: read balance (100) ──┐                 ┌── write 100-80 = 20
Request B: read balance (100) ──┴── both see 100 ─┴── write 100-80 = 20
                                                       final balance = 20, but you spent 160. 💥
```
Both debits saw "enough balance"; both succeeded; the account is overdrawn. This is a **lost update**.

### Check yourself
1. Where exactly is the race window? (Between the read and the write — the "gap".)
2. Anything with a "can only happen once" rule (unique username, last seat, one-time token) is a race
   waiting to happen. Name one in a system you know.

### ✅ Lesson 2.1 checklist
- [ ] 📖 Read & understood — I can describe check-then-act and the read-modify-write gap.
- [ ] 💻 Applied — I identified a check-then-act window in code I've written.
- [ ] 🔍 Found in Mythos — `BE#70` tests "concurrent debit safety (no balance corruption)".

---

## Lesson 2.2 — Atomic single-statement updates (the cheapest fix)

**Goal:** eliminate the gap by folding the check *into* the write.

### Concept
The database processes one row-write at a time, so if the guard lives **inside** the `UPDATE`, it
can't be stale. Then you inspect **rows affected** to learn whether you won.

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — read-modify-write; two callers both pass and overdraw
const { balance } = await getWallet(userId);
if (balance >= amount) {
  await setBalance(userId, balance - amount);   // stale `balance` under concurrency
}
```
```sql
-- ✅ GOOD — the guard is part of the write; the DB can't let both win
UPDATE wallets
   SET balance = balance - $amount
 WHERE user_id = $userId
   AND balance >= $amount;          -- guard lives INSIDE the write
-- then in app code: if rowsAffected === 0 → insufficient funds (someone else won, or never enough)
```

The single-use token guard is the same idea:
```sql
UPDATE launch_sessions
   SET consumed_at = now()
 WHERE jti = $jti
   AND consumed_at IS NULL;         -- only the FIRST caller matches
-- rowsAffected === 1 → you consumed it; === 0 → already consumed → return 409
```

### Check yourself
1. Why check `rowsAffected` instead of re-reading the row? (Re-reading reintroduces a gap; rows-affected
   is reported atomically by the same statement.)
2. What does `rowsAffected === 0` mean here — "error" or "someone else won"? (The latter; map it to
   the right outcome, e.g. 409 or 402.)

### ✅ Lesson 2.2 checklist
- [ ] 📖 Read & understood — I can rewrite a read-modify-write as a guarded single-statement update.
- [ ] 💻 Applied — I converted one check-then-act into `UPDATE … WHERE <guard>` + rows-affected.
- [ ] 🔍 Found in Mythos — `BE#88` `consumeSession`: `UPDATE … WHERE jti=? AND consumed_at IS NULL`, then checks rows-affected → 409.

---

## Lesson 2.3 — Optimistic concurrency control (OCC)

**Goal:** handle conflicts cheaply when they're *rare*.

### Concept
**Assume conflicts are rare.** Don't lock. Detect a conflict *at write time* — via a version column,
a `WHERE` guard, or a unique constraint — and have the loser **retry or bail**.

```sql
-- version-column OCC: the write only succeeds if nobody changed the row since you read it
UPDATE documents
   SET body = $newBody, version = version + 1
 WHERE id = $id AND version = $expectedVersion;
-- rowsAffected === 0 → someone edited it first → re-read and retry (or surface a conflict)
```

**Trade-off:** cheap when contention is low (no lock held). Costs you *retry logic* when collisions
do happen. Great default for "users rarely edit the same row at the same instant."

### ✅ Lesson 2.3 checklist
- [ ] 📖 Read & understood — I can explain OCC and when it's the right choice.
- [ ] 💻 Applied — I added a `version` column and a version-guarded update.
- [ ] 🔍 Found in Mythos — `BE#88` `consumeSession` is OCC: the `WHERE … IS NULL` guard *is* the conflict check.

---

## Lesson 2.4 — Pessimistic locking (`SELECT … FOR UPDATE`)

**Goal:** serialize access to a hot row when conflicts are *likely*.

### Concept
**Assume conflicts are likely.** Take a lock *before* reading: `SELECT … FOR UPDATE` locks the
row(s) until your transaction commits, forcing other writers to queue. Safe and easy to reason
about, at a **throughput cost** (access is serialized) and a **deadlock risk** (Lesson 2.6).

```sql
BEGIN;
  -- lock the session and the wallet so concurrent meter calls serialize behind us
  SELECT * FROM launch_sessions WHERE jti = $jti FOR UPDATE;
  SELECT * FROM wallets         WHERE user_id = $uid FOR UPDATE;

  -- now the multi-step balance math (subscription bucket first, then top-up) can't interleave
  UPDATE wallets SET subscription_balance = subscription_balance - $a WHERE user_id = $uid;
  INSERT INTO wallet_ledger (...) VALUES (...);
COMMIT;   -- locks released here
```

Use this for **hot, money-critical paths** where two callers really do hit the same row and the math
spans several statements.

### Check yourself
1. Optimistic vs pessimistic — what determines the choice? (How often you actually collide. Rare →
   optimistic; frequent/critical multi-step → pessimistic.)
2. When are the locks released? (At `COMMIT`/`ROLLBACK` — the end of the transaction.)

### ✅ Lesson 2.4 checklist
- [ ] 📖 Read & understood — I can describe `FOR UPDATE` and its throughput trade-off.
- [ ] 💻 Applied — I wrote a transaction that locks a row, mutates it, and commits.
- [ ] 🔍 Found in Mythos — `BE#88` `metered_charge` RPC locks the session and wallet rows `FOR UPDATE`.

---

## Lesson 2.5 — Unique constraints & isolation levels

**Goal:** use the DB's built-in race-free primitives.

### Concept
- **A `UNIQUE` index is a race-free "only one of these may exist,"** enforced by the DB. It's the
  referee for idempotency keys (Module 01) and natural keys alike — `SELECT`-before-`INSERT` does
  *not* prevent duplicates, only the constraint does.
- **Transactions + isolation levels.** Wrapping statements in a transaction gives atomicity; the
  *isolation level* controls which anomalies are possible:

| Level | Prevents | Still allows |
|-------|----------|--------------|
| Read Committed (PG default) | dirty reads | non-repeatable reads, phantoms, write skew |
| Repeatable Read | + non-repeatable reads, phantoms (in PG) | write skew |
| Serializable | everything (behaves as some serial order) | nothing — but may abort & need retry |

### Check yourself
1. "I `SELECT` first to check the key doesn't exist, then `INSERT`." Safe? (No — the `SELECT` is an
   optimization, not a guarantee. Only the `UNIQUE` constraint closes the window.)
2. You see "write skew" between two rows that must satisfy a joint rule. What level fixes it?
   (Serializable — or an explicit lock.)

### ✅ Lesson 2.5 checklist
- [ ] 📖 Read & understood — I can explain why a constraint, not a `SELECT`, prevents duplicates.
- [ ] 💻 Applied — I named the isolation level of a DB I use and one anomaly it still allows.
- [ ] 🔍 Found in Mythos — `BE#88`: `UNIQUE (reference_id, kind, bucket)` lets exactly one concurrent retry win.

---

## Lesson 2.6 — Pitfalls: deadlocks, TOCTOU, and races in *time*

**Goal:** avoid the classic ways locking goes wrong.

### Concept
- **Deadlocks.** Two transactions grab locks in *opposite order* and wait forever.
  **Fix:** always acquire locks in a consistent order (e.g. always session-then-wallet).
- **TOCTOU (time-of-check to time-of-use).** Check a permission/file, then act — but it changed in
  between. Common in auth and filesystem code.
- **Races in *time* (deployment/migration).** A subtler one: migration ordering. If migration A runs
  before migration B that was meant to correct A's shape — and B uses `CREATE TABLE IF NOT EXISTS` —
  B silently no-ops and the wrong shape survives. Same hazard ("this op is a no-op if it already
  exists"), applied to schema evolution. (See Module [03](03-database-migrations.md).)

### ✅ Lesson 2.6 checklist
- [ ] 📖 Read & understood — I can explain deadlock prevention via consistent lock ordering and TOCTOU.
- [ ] 💻 Applied — I picked a fixed lock-acquisition order for a multi-row transaction.
- [ ] 🔍 Found in Mythos — `BE#87`: migration `20260514` ran before `20260528`; `IF NOT EXISTS` masked the fix → `23502` on every launch.

---

## 🎯 Module 02 mastery checklist
- [ ] I can spot a check-then-act race and explain its window.
- [ ] I can fix a lost-update with an atomic guarded `UPDATE` + rows-affected.
- [ ] I can choose optimistic vs pessimistic concurrency control for a given workload.
- [ ] I can write a `FOR UPDATE` transaction and explain when locks release.
- [ ] I can explain why only a `UNIQUE` constraint (not a `SELECT`) prevents duplicates.
- [ ] I can prevent deadlocks with consistent lock ordering.

## 🛠️ Mini-project
Reproduce and then fix a double-spend:
1. Make a `wallets(user_id, balance)` table seeded at 100.
2. Write a naive `debit(80)` using read-modify-write. Fire **two** debits concurrently (e.g.
   `Promise.all`) and observe the balance go to 20 (you spent 160). 
3. Fix it two ways: (a) atomic `UPDATE … WHERE balance >= $amount` + rows-affected; (b) a
   transaction with `SELECT … FOR UPDATE`. Prove both reject the second debit.

## 🔗 Mythos PR map
| PR | What it demonstrates |
|----|----------------------|
| `BE#88` | OCC single-use guard (`consumeSession`), pessimistic `FOR UPDATE` in `metered_charge`, the idempotency race & 23505 fix. |
| `BE#70` | "Concurrent debit safety" and "overdraft rejection" as explicit tests. |
| `BE#87` | A race in *time* — migration ordering + `IF NOT EXISTS` masking. |

## See also
- [01 — Idempotency](01-idempotency.md) — the unique-constraint referee.
- [03 — Database design & migrations](03-database-migrations.md) — ordering hazards.
- [09 — Payments & billing](09-payments.md) — locking the wallet row.
