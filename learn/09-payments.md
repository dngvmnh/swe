# Module 09 — Payments & Billing Architecture

> **Core question:** How do you move money correctly?
> **Reference doc:** [../09-payments-and-billing-architecture.md](../09-payments-and-billing-architecture.md)
> **Prerequisites:** [01 — Idempotency](01-idempotency.md), [02 — Concurrency](02-concurrency.md), [08 — Security](08-security.md). This is the module that ties them together.

## Why this module
Money bugs are uniquely costly: a double-charge is a refund + a support ticket + lost trust; a missed
charge is lost revenue; a corrupted balance is an accounting nightmare with no clean recovery. And
money operations are exactly the ones that get **retried** (lost responses) and run **concurrently**
(same user, two tabs). Billing concentrates the hardest patterns in one place.

### What you'll be able to do
- Represent money without rounding errors.
- Design an append-only ledger and a credit/token economy.
- Make every money operation exactly-once and lock-safe.
- Split marketplace revenue and model a subscription lifecycle.

---

## Lesson 9.1 — Integer minor units (never floats)

**Goal:** represent money safely.

### Concept
Represent money as **integer cents (or credits), never floating point.** Floats can't represent
`0.1` exactly; summing them drifts.

```ts
// ❌ BAD — float money drifts
0.1 + 0.2 === 0.3            // false! → 0.30000000000000004
let total = 0.0; for (const p of prices) total += p;   // accumulates rounding error

// ✅ GOOD — integer minor units; convert only at the boundary (display / Stripe)
const cents = 1099;                       // $10.99
const display = `$${(cents / 100).toFixed(2)}`;
```

### ✅ Lesson 9.1 checklist
- [ ] 📖 Read & understood — I can explain why money is stored as integers.
- [ ] 💻 Applied — I modeled a price/balance as integer minor units.
- [ ] 🔍 Found in Mythos — wallet balances and credits are integer units throughout.

---

## Lesson 9.2 — Append-only ledger (double-entry flavored)

**Goal:** make every money movement immutable and auditable.

### Concept
Every credit and debit is an **immutable row**. You **never edit a posted entry — you post a new
one** (a compensating entry for a reversal). Balances are maintained alongside, but the **ledger is
the source of truth and the audit trail.** Bonus: the ledger's unique columns double as idempotency
keys (Module [01](01-idempotency.md)).

```sql
-- append-only: INSERT only, never UPDATE/DELETE a posted row
CREATE TABLE wallet_ledger (
  id            bigserial PRIMARY KEY,
  user_id       uuid NOT NULL,
  delta         integer NOT NULL,        -- +credit / -debit, integer units
  kind          text NOT NULL,           -- 'pack_purchase' | 'metered_charge' | ...
  bucket        text NOT NULL,           -- 'subscription' | 'topup'
  reference_id  text NOT NULL,           -- idempotency anchor
  stripe_event_id text,                  -- dedup for webhooks
  created_at    timestamptz NOT NULL DEFAULT now()
);
```

### ✅ Lesson 9.2 checklist
- [ ] 📖 Read & understood — I can explain append-only + compensating entries + audit trail.
- [ ] 💻 Applied — I designed an append-only ledger table.
- [ ] 🔍 Found in Mythos — `BE#70` append-only `wallet_ledger` (nothing mutated).

---

## Lesson 9.3 — Credit economy; processor at the boundary

**Goal:** isolate the payment processor from the rest of the system.

### Concept
A **credit/token economy:** users buy an internal currency (credits) **once**, then spend it across
the product. This decouples "money in" (a few Stripe flows) from "spending it" (many internal
debits) — so **most of the system never touches Stripe at all.**

```
Stripe (the boundary)                 Internal (the rest of the system)
  checkout / subscription  ──webhook──►  wallet_credit(+credits)
                                          wallet_debit(-credits)   ← purely internal, no Stripe
  convert Stripe ↔ internal units ONLY at this boundary
```

### ✅ Lesson 9.3 checklist
- [ ] 📖 Read & understood — I can explain why a credit economy isolates the processor.
- [ ] 💻 Applied — I drew which operations touch the processor vs internal-only.
- [ ] 🔍 Found in Mythos — Stripe touches only three "money in" flows; everything else is internal credit movement.

---

## Lesson 9.4 — Bucketed balances + drain order

**Goal:** make multi-source spending deterministic.

### Concept
When credits come from multiple sources, define an **explicit spend order** so charges are
deterministic — e.g. **subscription credits before purchased top-up.**

```
balance = { subscription: 100, topup: 50 }
charge 120 → drain subscription first (100), then topup (20)
result    = { subscription: 0,  topup: 30 }
```

### ✅ Lesson 9.4 checklist
- [ ] 📖 Read & understood — I can explain bucketed balances and a fixed drain order.
- [ ] 💻 Applied — I implemented a drain-order function over two buckets.
- [ ] 🔍 Found in Mythos — `BE#70` dual-bucket wallet with drain order "subscription first," tested.

---

## Lesson 9.5 — Idempotency + locking + atomic RPC (the core)

**Goal:** combine the foundations into one correct money op.

### Concept
Every money op must be:
- **Idempotent** — webhooks dedup on the processor event id; internal charges on a client
  `charge_id`; crons on a synthetic period key.
- **Lock-safe** — lock the wallet row while debiting so two charges can't both read the same balance
  and overdraw.
- **Atomic** — debit wallet + write ledger + pay creator must all succeed or all fail → wrap in one
  transaction/RPC.

```sql
-- metered_charge: the whole money mutation, atomic and locked
CREATE FUNCTION metered_charge(p_jti uuid, p_credits int, p_charge_id text)
RETURNS TABLE(new_balance int) SECURITY DEFINER SET search_path = public AS $$
BEGIN
  PERFORM 1 FROM launch_sessions WHERE jti = p_jti FOR UPDATE;     -- lock session
  PERFORM 1 FROM wallets WHERE user_id = (...)        FOR UPDATE;  -- lock wallet
  -- drain subscription bucket first, then topup; floor 85% to creator
  INSERT INTO wallet_ledger (reference_id, kind, bucket, delta, ...) VALUES (p_charge_id, 'metered_charge', 'subscription', ...);
  -- UNIQUE (reference_id, kind, bucket) is the idempotency referee
  RETURN QUERY SELECT subscription_balance + topup_balance FROM wallets WHERE user_id = (...);
END $$ LANGUAGE plpgsql;
```
On the app side, catch `23505` and **replay the original result** (Module [01](01-idempotency.md), Lesson 1.4).

### ✅ Lesson 9.5 checklist
- [ ] 📖 Read & understood — I can list the three guarantees every money op needs.
- [ ] 💻 Applied — I wrote a debit that is idempotent, locked, and atomic.
- [ ] 🔍 Found in Mythos — `BE#88` `/meter`: integer units, ledger writes, `FOR UPDATE` locking, `charge_id` idempotency, atomic RPC.

---

## Lesson 9.6 — Revenue split & subscription lifecycle

**Goal:** handle the marketplace and recurring-billing specifics.

### Concept
- **Marketplace revenue split.** A two-sided market takes a platform cut and pays the rest to the
  creator, computed with **integer math (floor)** to avoid fractional credits. (Mythos: 85% to creator.)
```ts
const creatorShare = Math.floor(charge * 85 / 100);  // integer floor — no fractional credits
const platformCut  = charge - creatorShare;          // remainder stays with the platform
```
- **Subscription lifecycle.** Create, renew, downgrade, cancel — each a **distinct, idempotent
  webhook path.** Downgrades typically **defer to period end with no mid-cycle clawback** (a
  deliberate fairness design).

### ✅ Lesson 9.6 checklist
- [ ] 📖 Read & understood — I can explain integer-floor revenue split and the downgrade-at-period-end rule.
- [ ] 💻 Applied — I wrote a floor-based split and reasoned about where the remainder goes.
- [ ] 🔍 Found in Mythos — `BE#88` 85% creator floor; `BE#85` subscription lifecycle with deferred downgrades.

---

## 🎯 Module 09 mastery checklist
- [ ] I can represent money as integer minor units and convert only at the boundary.
- [ ] I can design an append-only ledger with compensating entries.
- [ ] I can explain why a credit economy isolates the payment processor.
- [ ] I can implement bucketed balances with a deterministic drain order.
- [ ] I can make a money op idempotent, lock-safe, and atomic in one RPC/transaction.
- [ ] I can compute an integer-floor revenue split and model a subscription lifecycle.

## 🛠️ Mini-project
Build a minimal wallet:
1. `wallets(subscription int, topup int)` + append-only `wallet_ledger` with `UNIQUE (reference_id, kind, bucket)`.
2. `credit(amount, bucket, ref)` (idempotent on `ref`) and `debit(amount, ref)` (drain subscription
   first, then topup; reject with 402 if insufficient).
3. Wrap `debit` in a transaction with `SELECT … FOR UPDATE`. Test: concurrent debits don't overdraw;
   the same `ref` debits once; an over-balance debit returns 402.
4. Add an 85/15 integer-floor split recorded as two ledger rows.

## 🔗 Mythos PR map
| PR | What it demonstrates |
|----|----------------------|
| `BE#70` | Dual-bucket wallet + append-only ledger + 12 invariant tests. |
| `BE#80` | Money in #1 — top-up packs (webhook credit, idempotent on event id). |
| `BE#85` | Money in #2 — subscriptions + full lifecycle, deferred downgrades. |
| `BE#86` | Money in #3 — renewal safety-net cron (synthetic key). |
| `BE#88` | Money out — metered spend: integer units, lock, idempotency, atomic RPC, 85% split. |
| `BE#18` | Webhook amount-verification hardening. |

## See also
- [01 — Idempotency](01-idempotency.md)
- [02 — Concurrency & locking](02-concurrency.md)
- [08 — Security engineering](08-security.md)
- [14 — Async jobs & scheduling](14-async-jobs.md) — renewal grants.
