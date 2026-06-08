# 02 — Concurrency, Locking & Race Conditions

## The idea

A **race condition** is a bug where the correctness of a result depends on the *timing* of
events you don't control. When two operations interleave, the outcome differs from running them
one after another. The classic shape is **check-then-act**: you read state, decide based on it,
then write — but between the read and the write, someone else changed the state your decision
relied on.

**Concurrency control** is the set of techniques for making concurrent operations behave as if
they ran in some serial order (the formal property is *serializability*).

## Why it exists

A server handles many requests at once. A database serves many connections. The moment two of
them touch the same row, "it worked when I tested it alone" stops being a guarantee. Money,
inventory, seat reservations, unique usernames — anything with a "can only happen once" rule is
a race waiting to happen.

## Patterns & trade-offs

- **Atomic single-statement updates.** Fold the check *into* the write so the database
  evaluates them together: `UPDATE ... SET x WHERE <guard>`. The DB processes one row-write at a
  time, so the guard can't be stale. Then inspect *rows affected* to learn whether you won.
- **Optimistic concurrency control (OCC).** Assume conflicts are rare. Don't lock; instead
  detect a conflict at write time (via a version column, or a `WHERE` guard, or a unique
  constraint) and have the loser retry or bail. Cheap when contention is low.
- **Pessimistic locking.** Assume conflicts are likely. Take a lock *before* reading:
  `SELECT ... FOR UPDATE` locks the row until your transaction commits, forcing other writers
  to queue. Safe and simple to reason about, but serializes access (throughput cost) and risks
  deadlocks if lock ordering is inconsistent.
- **Unique constraints as a concurrency primitive.** A `UNIQUE` index is a race-free "only one
  of these may exist" enforced by the DB — useful for idempotency keys and natural keys alike.
- **Transactions & isolation levels.** Wrapping multiple statements in a transaction gives
  atomicity; the *isolation level* (Read Committed → Serializable) controls which anomalies
  (dirty/non-repeatable reads, phantoms, write skew) are possible.

Trade-off: pessimistic locking trades throughput for simplicity; optimistic trades retry
complexity for throughput. Pick based on how often you actually collide.

## Pitfalls

- **The read-modify-write gap.** `bal = read(); write(bal - 10)` loses updates under
  concurrency. Use `SET bal = bal - 10` (atomic) or a lock.
- **Lost updates / double-spend.** Two debits both see "enough balance" and both succeed,
  overdrawing the account.
- **Deadlocks.** Two transactions grab locks in opposite order and wait forever. Prevent by
  always acquiring locks in a consistent order.
- **TOCTOU (time-of-check to time-of-use).** Especially common in auth and filesystem code.
- **Assuming `SELECT` before `INSERT` prevents duplicates.** It doesn't — only the constraint
  does. The `SELECT` is an optimization, not a guarantee.

## Seen in Mythos

- **Optimistic single-use guard — `BE#88` (`consumeSession`).** A launch session must be
  consumed exactly once. Instead of "read `consumed_at`, if null then update," the code does one
  atomic statement: `UPDATE launch_sessions SET consumed_at = now() WHERE jti = ? AND consumed_at
  IS NULL` and then checks rows-affected. The first request matches and updates; the second
  finds zero rows because the guard `consumed_at IS NULL` no longer holds → it returns `409`. The
  race window is closed because the check lives *inside* the write.

- **Pessimistic row locks in the charge RPC — `BE#88` migration.** The `metered_charge` Postgres
  function does `SELECT * FROM launch_sessions WHERE jti = ? FOR UPDATE` and locks the wallet row
  too. Concurrent meter calls for the same session serialize behind the lock, so the
  subscription-first / topup-second drain and balance math can't interleave and corrupt the
  balance. This is the deliberate choice of pessimistic locking for a hot, money-critical path.

- **The idempotency race and its fix — `BE#88`.** Two concurrent retries with the same
  `charge_id` both pass the application-level `SELECT` (neither has written yet) and both call
  the RPC. The `UNIQUE (reference_id, kind, bucket)` index lets exactly one win; the loser hits
  `23505`. The reviewer's key insight: the unique index prevents the double *debit*, but the
  loser still needs to be caught and replayed, or the caller sees a 500. This is the canonical
  example of "the DB constraint is the referee; the application handles the loser."

- **Concurrent debit safety as a test — `BE#70`.** The wallet PR explicitly tests "concurrent
  debit safety (no balance corruption)" and "overdraft rejection." Race safety is treated as a
  behavior to verify, not assume.

- **Migration ordering as a race in *time* — `BE#87`.** A subtler concurrency-of-deployment
  problem: migration `20260514` ran *before* MYT-106's `20260528`, creating `launch_sessions`
  with stale columns. Because MYT-106 used `CREATE TABLE IF NOT EXISTS`, the corrected schema was
  a silent no-op and the stale `NOT NULL` columns survived, causing `23502` violations on every
  launch. Ordering and "this op is a no-op if the thing already exists" are the same hazards that
  bite concurrent code — here applied to schema evolution.

## See also

- [01 — Idempotency](01-idempotency-and-exactly-once-semantics.md) (the unique-constraint referee)
- [03 — Database design & migrations](03-database-design-and-schema-migrations.md)
- [09 — Payments & billing architecture](09-payments-and-billing-architecture.md)
