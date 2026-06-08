# 01 — Idempotency & Exactly-Once Semantics

## The idea

An operation is **idempotent** if performing it many times has the same effect as performing
it once. `DELETE /user/42` is naturally idempotent — the user is gone whether you call it once
or five times. `POST /charge $10` is *not* — call it five times and you've charged $50.

The engineering goal is usually **exactly-once semantics**: the real-world effect (a charge, a
credit, an email) happens once and only once, even though the *message* asking for it may
arrive zero, one, or many times.

## Why it exists

Networks are unreliable, and that unreliability is not an edge case — it is the normal
condition of distributed systems. A client sends a request, the server processes it, and the
response is lost in transit. The client cannot tell the difference between "it never arrived"
and "it worked but the receipt was lost," so it **retries**. Without idempotency, every retry
is a duplicate side effect.

The fundamental truth: **you cannot achieve exactly-once *delivery*** (it's provably
impossible in an asynchronous network with failures). What you *can* do is achieve exactly-once
*processing* by making delivery at-least-once (retries) and then de-duplicating on the
receiving side. Idempotency is how you turn "at least once" into "effectively once."

## Patterns & trade-offs

- **Idempotency key.** The client generates a unique ID per logical operation and sends it
  with every retry. The server records which keys it has processed and short-circuits repeats.
  Stripe's own API works exactly this way (`Idempotency-Key` header).
- **Natural idempotency.** Design the operation so repeating it is harmless: `SET balance = 100`
  instead of `balance = balance + 10`. Prefer this when you can — no bookkeeping required.
- **Database uniqueness as the referee.** A `UNIQUE` constraint on the idempotency key makes the
  *database* enforce exactly-once. The second insert fails atomically; you catch the violation
  and treat it as "already done." This is the strongest guarantee because it holds even under
  concurrency (see [doc 02](02-concurrency-locking-and-race-conditions.md)).
- **Dedup table / processed-events ledger.** Record every processed message id; check before
  acting. Trade-off: the table grows forever unless you prune old keys.

Trade-off summary: application-level checks ("SELECT then act") are simple but **not safe under
concurrency**; database constraints are safe but require you to handle the violation gracefully.

## Pitfalls

- **Check-then-act races.** Two concurrent retries both pass the "have I seen this key?" check
  before either writes. Only a DB constraint or a lock closes this window — see doc 02.
- **Catching the violation as a generic error.** If the unique-violation surfaces as a 500, the
  caller thinks it failed and retries again. The idempotency *mechanism* worked but the
  idempotency *contract* (return the original success) was broken.
- **Keys that aren't actually unique per operation.** Reusing a key across two genuinely
  different charges silently drops the second one.
- **Idempotency that doesn't return the original result.** A correct replay returns the *same
  response* the first call produced, not just a bare 200.

## Seen in Mythos

This codebase treats idempotency as a first-class requirement because nearly everything touches
money. Four different mechanisms appear:

- **`charge_id` idempotency for metered billing — `BE#88`.** The `/meter` endpoint takes a
  client-generated `charge_id`. The service first `SELECT`s `wallet_ledger` for that id (fast
  path); the real guarantee is a `UNIQUE` index on `(reference_id, kind, bucket)`. The original
  PR caught the `23505` unique violation as a generic `InternalError` → **500**, which broke the
  contract. The fix catches `23505`, re-reads the ledger row the winning request wrote, and
  returns it as a calm `200` — exactly-once processing under concurrent retries. This is the
  textbook "DB constraint + graceful replay" pattern.

- **Stripe webhook dedup on `stripe_event_id` — `BE#80`.** Stripe may deliver the same
  `checkout.session.completed` event multiple times. The wallet credit is keyed on
  `wallet_ledger.stripe_event_id` (unique), so replayed events do not double-credit. The
  uniqueness of the ledger column *is* the dedup.

- **Cron job idempotency via synthetic keys — `BE#86`.** The hourly `subscription_renew` cron
  could run twice (overlapping executions). Each renewal is tagged with a synthetic id
  `cron_<subscription_id>_<period_epoch>` stored in `stripe_invoice_id`. Two runs in the same
  period produce the same key → no duplicate credit grant. This shows how to make a *scheduled*
  job idempotent when there's no client to supply a key — you derive a deterministic key from
  the work itself.

- **Idempotency as a tested invariant — `BE#70`.** The dual-bucket wallet PR shipped 12
  integration tests, several dedicated to idempotency: "idempotency on `stripe_event_id`,"
  "`subscription_renew` behavior and idempotency," "concurrent debit safety (no balance
  corruption)." Idempotency isn't asserted in prose — it's pinned by tests.

**The throughline:** every money-moving path in Mythos derives a stable key, lets a database
`UNIQUE` constraint be the final referee, and (after `BE#88`) gracefully replays the original
result instead of erroring on the duplicate.

## See also

- [02 — Concurrency, locking & race conditions](02-concurrency-locking-and-race-conditions.md)
  (why check-then-act is unsafe and how locks/constraints fix it)
- [09 — Payments & billing architecture](09-payments-and-billing-architecture.md)
- [14 — Async jobs & scheduling](14-async-jobs-and-scheduling.md)
