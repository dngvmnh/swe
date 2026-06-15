# Module 01 — Idempotency & Exactly-Once Semantics

> **Core question:** How do you make an operation safe to retry?
> **Reference doc:** [../01-idempotency-and-exactly-once-semantics.md](../01-idempotency-and-exactly-once-semantics.md)
> **Prerequisites:** none — this is the foundation. (Module [02 — Concurrency](02-concurrency.md) deepens the "why" behind unique constraints.)

## Why this module
Networks lose messages. Clients retry. If your `POST /charge` runs twice, you charged the customer
twice. By the end of this module you'll be able to take *any* state-changing operation and make it
**safe to call any number of times** while the real-world effect happens **exactly once**.

### What you'll be able to do
- Tell at a glance whether an operation is naturally idempotent.
- Choose the right de-duplication mechanism for a given situation.
- Implement the "DB unique constraint + graceful replay" pattern correctly.
- Make a *scheduled* job idempotent when there's no client to supply a key.

---

## Lesson 1.1 — What "idempotent" actually means

**Goal:** define idempotency precisely and classify operations on sight.

### Concept
An operation is **idempotent** if doing it many times has the same effect as doing it once.

| Operation | Idempotent? | Why |
|-----------|-------------|-----|
| `DELETE /user/42` | ✅ | The user is gone whether you call it once or five times. |
| `PUT /user/42 {name:"Ann"}` | ✅ | Sets an absolute value; re-applying changes nothing. |
| `POST /charge $10` | ❌ | Five calls = $50. |
| `balance = balance + 10` | ❌ | Each call adds again. |
| `balance = 100` | ✅ | Absolute assignment; re-applying is a no-op. |

The engineering goal is usually **exactly-once *processing*** of the *effect* (a charge, a credit,
an email), even though the *message* asking for it may arrive zero, one, or many times.

### Check yourself
1. Is `INSERT INTO logs (...)` idempotent? (No — each call adds a row, unless guarded by a unique key.)
2. Rewrite `SET seats = seats - 1` into a form that is safe to retry. (Hint: you can't with pure
   arithmetic — you need a guard or a dedup key. That tension *is* this module.)

### ✅ Lesson 1.1 checklist
- [ ] 📖 Read & understood — I can define idempotency and classify an operation as idempotent or not.
- [ ] 💻 Applied — I classified 5 operations from a project I know.
- [ ] 🔍 Found in Mythos — n/a (concept lesson).

---

## Lesson 1.2 — Why "exactly-once delivery" is a myth

**Goal:** internalize the core distributed-systems truth that motivates everything else.

### Concept
A client sends a request; the server processes it; the **response is lost** in transit. The client
*cannot tell* "it never arrived" from "it worked but the receipt was lost." So it **retries**.

> **You cannot achieve exactly-once *delivery*** — it's provably impossible in an asynchronous
> network with failures. What you *can* do: make delivery **at-least-once** (retries) and then
> **de-duplicate on the receiving side**. Idempotency is how you turn "at least once" into
> "effectively once."

```
client                         server
  |  POST /charge  (attempt 1)   |
  | ---------------------------> |  charge happens ✅
  |        (response lost) ✗ <-- |
  |  POST /charge  (attempt 2)   |   <-- the retry. Without dedup, you charge AGAIN.
  | ---------------------------> |  charge happens ✅✅  ← BUG
```

### Check yourself
1. Why can't the client just "not retry"? (Because it can't distinguish failure from lost-receipt;
   not retrying means silently dropping real requests.)
2. Where does the dedup have to live — client or server? (Server/receiver — the client is the one
   sending duplicates.)

### ✅ Lesson 1.2 checklist
- [ ] 📖 Read & understood — I can explain why retries are inevitable and dedup must be server-side.
- [ ] 💻 Applied — I drew the lost-response sequence for an endpoint I work on.
- [ ] 🔍 Found in Mythos — see how Stripe (an external client) retries webhooks → Lesson 1.4.

---

## Lesson 1.3 — The four mechanisms (and when to use each)

**Goal:** pick the right tool. There are exactly four common ones.

### 1) Natural idempotency — *prefer this when you can*
Design the operation so repeating it is harmless. No bookkeeping required.
```sql
-- ❌ not idempotent
UPDATE wallets SET balance = balance + 10 WHERE user_id = $1;
-- ✅ naturally idempotent (absolute value) — but only works when you know the target value
UPDATE wallets SET balance = 100 WHERE user_id = $1;
```

### 2) Idempotency key — *the general-purpose tool*
The **client** generates a unique ID per logical operation and sends it with every retry. The
server records processed keys and short-circuits repeats. (This is exactly how Stripe's own API
works — the `Idempotency-Key` header.)
```ts
// client sends the SAME key on every retry of the SAME logical charge
await fetch("/api/charge", {
  method: "POST",
  headers: { "Idempotency-Key": chargeId },   // chargeId stable across retries
  body: JSON.stringify({ amount: 10 }),
});
```

### 3) Database uniqueness as the referee — *the strongest guarantee*
A `UNIQUE` constraint makes the **database** enforce exactly-once. The second insert fails
atomically; you catch the violation and treat it as "already done." This holds **even under
concurrency** (two retries at the same instant), which application checks do not.
```sql
CREATE UNIQUE INDEX uq_ledger_ref ON wallet_ledger (reference_id, kind, bucket);
```

### 4) Dedup table / processed-events ledger
Record every processed message id; check before acting.
**Trade-off:** the table grows forever unless you prune old keys.

### The decisive trade-off
> Application-level checks ("`SELECT` then act") are simple but **NOT safe under concurrency**.
> Database constraints are safe but require you to **handle the violation gracefully** (Lesson 1.4).

### Check yourself
1. Two of these are safe under concurrency; two are not. Which? (Safe: unique constraint, natural
   idempotency. Unsafe alone: a plain dedup-table `SELECT`-then-`INSERT`, an idempotency key checked
   with `SELECT`-then-act — both have a check-then-act race; the *constraint* under them is what
   makes them safe.)

### ✅ Lesson 1.3 checklist
- [ ] 📖 Read & understood — I can name the four mechanisms and their trade-offs.
- [ ] 💻 Applied — I wrote a `CREATE UNIQUE INDEX` for an idempotency key in a schema I have.
- [ ] 🔍 Found in Mythos — `BE#80` dedups Stripe webhooks on `wallet_ledger.stripe_event_id` (mechanism #3).

---

## Lesson 1.4 — The graceful-replay contract (the part everyone gets wrong)

**Goal:** make a duplicate return the *original success*, not an error.

### Concept
A unique constraint stops the double *effect*. But the **loser** of the race still gets a database
error. If you let that bubble up as a `500`, the caller thinks it failed and **retries again** —
the mechanism worked but the *contract* broke. A correct replay returns the **same response the
first call produced.**

### ❌ Bad → ✅ Good

```ts
// ❌ BAD — the unique violation surfaces as a 500; the caller retries forever
async function meter(chargeId: string, credits: number) {
  try {
    return await db.rpc("metered_charge", { charge_id: chargeId, credits });
  } catch (e) {
    throw new InternalError("charge failed");      // 23505 becomes a 500 — contract broken
  }
}
```

```ts
// ✅ GOOD — catch the specific unique violation, re-read what the winner wrote, replay it as 200
async function meter(chargeId: string, credits: number) {
  try {
    return await db.rpc("metered_charge", { charge_id: chargeId, credits });
  } catch (e: any) {
    if (e.code === "23505") {                       // Postgres unique_violation
      const original = await readLedgerRow(chargeId); // what the winning request committed
      if (!original) throw new InternalError("dedup row vanished"); // never trust an unchecked read
      return original;                              // exactly-once: same result, calm 200
    }
    throw e;
  }
}
```

> `23505` is the Postgres error code for **unique_violation** — memorize it; you'll catch it a lot.

### Check yourself
1. Why re-read the row instead of just returning `{ ok: true }`? (The contract is to return the
   *same response*, e.g. the new balance — a bare 200 hides whether the original actually did the work.)
2. What happens if you catch *all* errors as "already done" instead of just `23505`? (You'd swallow
   genuine failures and report fake success — see Module [06](06-error-handling.md), "fail loud".)

### ✅ Lesson 1.4 checklist
- [ ] 📖 Read & understood — I can describe the replay contract and why a 500 on a dup is a bug.
- [ ] 💻 Applied — I wrote a try/catch that special-cases `23505` and replays the original.
- [ ] 🔍 Found in Mythos — `BE#88` `/meter`: the original PR caught `23505` as `InternalError` (500);
  the fix re-reads the ledger row and returns 200. This is *the* textbook case.

---

## Lesson 1.5 — Idempotent jobs with no client (synthetic keys)

**Goal:** make a cron/queue job idempotent when nobody is there to mint a key.

### Concept
A scheduled job has no client to supply an idempotency key. So **derive one deterministically from
the work itself** — *what* + *when*: `cron_<entity>_<period>`. Two runs in the same period produce
the **same key** → the unique constraint blocks the duplicate.

```ts
// hourly subscription-renewal cron. Two overlapping runs must not double-grant credits.
function renewalKey(subscriptionId: string, periodEnd: Date): string {
  const epoch = Math.floor(periodEnd.getTime() / 1000);
  return `cron_${subscriptionId}_${epoch}`;          // deterministic: same period → same key
}

// stored in stripe_invoice_id (UNIQUE), so a second run in the same period hits 23505 → skipped
```

A queue delivers **at least once**; a cron may **overlap or double-fire**. Same rule, same fix.

### Check yourself
1. Why include the *period* in the key, not just the subscription id? (Because next period's renewal
   is a legitimately *new* grant — the key must change when the work changes.)
2. Your queue redelivers a "send welcome email" message. What's the dedup key? (The message id, or a
   synthetic `welcome_<userId>` — see Module [14](14-async-jobs.md).)

### ✅ Lesson 1.5 checklist
- [ ] 📖 Read & understood — I can derive a deterministic key for a job with no client.
- [ ] 💻 Applied — I designed a synthetic key for a periodic task in my own project.
- [ ] 🔍 Found in Mythos — `BE#86` tags each renewal `cron_<subscription_id>_<period_epoch>` in `stripe_invoice_id`.

---

## 🎯 Module 01 mastery checklist
Tick these when you can do them *without looking back*:
- [ ] I can classify any operation as idempotent or not, and explain why.
- [ ] I can explain why exactly-once *delivery* is impossible and exactly-once *processing* is achievable.
- [ ] I can choose between natural idempotency, an idempotency key, a unique constraint, and a dedup table.
- [ ] I can implement "unique constraint + catch `23505` + graceful replay of the original result."
- [ ] I can make a cron/queue job idempotent with a synthetic key.
- [ ] I can spot the "catching the violation as a generic 500" bug in a code review.

## 🛠️ Mini-project
Build a tiny `POST /credits/grant` endpoint (any language) that grants N credits to a user:
1. Accept a client `grant_id`. Store grants in a table with `UNIQUE (grant_id)`.
2. On a duplicate `grant_id`, **return the original grant** (same response), not an error.
3. Write two tests: (a) the same `grant_id` twice → balance increases **once**; (b) two *concurrent*
   requests with the same `grant_id` → still once (fire them in parallel to force the race).
4. Bonus: add a `/credits/renew` cron with a synthetic period key and test that running it twice in
   one period grants once.

## 🔗 Mythos PR map
| PR | What it demonstrates |
|----|----------------------|
| `BE#88` | `charge_id` idempotency: `UNIQUE (reference_id, kind, bucket)` + the 23505→500 bug and its 200 replay fix. |
| `BE#80` | Stripe webhook dedup on `wallet_ledger.stripe_event_id`. |
| `BE#86` | Cron idempotency via synthetic `cron_<sub>_<period>` key. |
| `BE#70` | Idempotency pinned as a tested invariant (12 integration tests). |

## See also
- [02 — Concurrency, locking & race conditions](02-concurrency.md) — *why* check-then-act is unsafe.
- [09 — Payments & billing](09-payments.md) — idempotency on every money path.
- [14 — Async jobs & scheduling](14-async-jobs.md) — synthetic keys for periodic work.
