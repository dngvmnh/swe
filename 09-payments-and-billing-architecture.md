# 09 — Payments & Billing Architecture

## The idea

Billing systems move real money, so they are held to a higher correctness bar than almost
anything else in an application. The core design problems are: **representing money without
rounding errors**, **recording every movement immutably** (a ledger), **integrating a payment
processor** (Stripe) without trusting it blindly, and making every money operation **exactly-once**
under retries and concurrency.

A common architecture is the **credit/token economy**: users buy an internal currency (credits)
once, then spend it across the product. This decouples "getting money in" (a few Stripe flows)
from "spending it" (many internal debits), so most of the system never touches Stripe at all.

## Why it exists

Money bugs are uniquely costly: a double-charge is a refund, a support ticket, and lost trust; a
missed charge is lost revenue; a corrupted balance is an accounting nightmare with no clean
recovery. And money operations are exactly the ones that get retried (lost responses) and run
concurrently (the same user, two tabs). So billing concentrates the hardest patterns — idempotency,
locking, immutable audit — in one place.

## Patterns & trade-offs

- **Integer minor units.** Represent money as integer cents (or credits), never floating point.
  Floats can't represent `0.1` exactly; summing them drifts.
- **Append-only ledger (double-entry flavored).** Every credit and debit is an immutable row. You
  never edit a posted entry — you post a new one. Balances are maintained alongside, but the
  ledger is the source of truth and the audit trail.
- **Bucketed balances + drain order.** When credits come from multiple sources, define an explicit
  spend order (e.g. subscription credits before purchased top-up) so charges are deterministic.
- **Processor at the boundary, internal currency inside.** Stripe handles "money in" (checkout,
  subscriptions); a webhook converts a completed payment into an internal credit. Spending is a
  purely internal debit. Convert between Stripe amounts and internal units only at the boundary.
- **Idempotency on every money op.** Webhook events dedup on the processor's event id; internal
  charges dedup on a client `charge_id`; scheduled grants dedup on a synthetic period key. (See
  [doc 01](01-idempotency-and-exactly-once-semantics.md).)
- **Locking for balance math.** Lock the wallet row while debiting so two charges can't both read
  the same balance and overdraw. (See [doc 02](02-concurrency-locking-and-race-conditions.md).)
- **Atomic multi-write via DB function.** Debit wallet + write ledger + pay creator must all
  succeed or all fail → wrap in a single transaction/RPC.
- **Marketplace revenue split.** A two-sided market takes a platform cut and pays the rest to the
  creator, computed with integer math (floor) to avoid fractional credits.
- **Subscription lifecycle.** Create, renew, downgrade, cancel — each a distinct, idempotent
  webhook path. Downgrades typically defer to period end with no mid-cycle clawback.

## Pitfalls

- **Float money** (rounding drift, un-auditable totals).
- **Trusting webhook metadata** without verifying amount and signature (see `BE#18`,
  [doc 08](08-security-engineering.md)).
- **A unique index that's too strict or too loose** — rejecting valid split charges, or allowing
  duplicates.
- **Non-idempotent webhooks/crons** → double-credit on replay.
- **Charging without locking** → overdraft / lost-update.
- **Mutating ledger rows** → no audit trail, no clean recovery.

## Seen in Mythos

Mythos is a two-sided marketplace with a unified credit economy; almost every doc in this guide
intersects billing. The architecture, end to end:

- **Dual-bucket wallet + append-only ledger — `BE#70` (MYT-101).** The foundation: two balance
  buckets (`subscription`, `topup`), an append-only `wallet_ledger` row on every movement,
  `subscription_plans`/`user_subscriptions`, and core `wallet_credit`/`wallet_debit` primitives
  with an explicit **drain order (subscription first)**. Shipped with 12 tests covering drain
  order, overdraft rejection, idempotency, and concurrent-debit safety. Credits/debits carry
  idempotency metadata.

- **Money in #1 — top-up packs — `BE#80` (MYT-103).** `POST /api/wallet/packs/:slug/checkout`
  creates a Stripe Checkout session; the `checkout.session.completed` webhook calls
  `wallet_credit(user, credits, 'topup', 'pack_purchase', stripe_session_id)`. Idempotent on
  `wallet_ledger.stripe_event_id` — replays don't double-credit.

- **Money in #2 — subscriptions — `BE#85` (MYT-103b).** `POST /api/wallet/subscription/checkout`
  in Stripe *subscription* mode for Free/Builder/Pro tiers. Webhooks handle the full lifecycle —
  creation, renewal, downgrade, cancellation — all idempotent on the Stripe event id. **Downgrades
  defer to period end with no immediate clawback** — a deliberate billing-fairness design.

- **Money in #3 — renewal safety-net cron — `BE#86` (MYT-103c).** An hourly job renews free-tier
  subscriptions (self-extend + 100 monthly credits); paid tiers are a safety net for missed Stripe
  webhooks. Idempotent via synthetic key `cron_<subscription_id>_<period_epoch>` stored as
  `stripe_invoice_id`. Per-row failures are logged without aborting the batch.

- **Money out — metered spend — `BE#88` (MYT-293).** Consumers launch a creator's app and get
  metered: `/meter` debits the wallet through the `metered_charge` RPC (subscription bucket first,
  then top-up), allocates **85% of revenue to the creator** (integer floor), and is idempotent on a
  client `charge_id` backed by a `UNIQUE (reference_id, kind, bucket)` index. The RPC locks the
  session and wallet rows `FOR UPDATE`. This single PR is a microcosm of the whole doc: integer
  units, ledger writes, locking, idempotency, atomic RPC, revenue split.

- **Webhook safety — `BE#18`.** Hardens the money-in boundary: makes amount verification
  unconditional and eliminates silent failures (detailed in [doc 08](08-security-engineering.md)).

**The shape to remember:** Stripe touches only the three "money in" flows; everything else is
internal credit movement recorded in one append-only ledger, every operation idempotent and
locked. That isolation is what keeps a payments system auditable and correct.

## See also

- [01 — Idempotency](01-idempotency-and-exactly-once-semantics.md)
- [02 — Concurrency & locking](02-concurrency-locking-and-race-conditions.md)
- [08 — Security engineering](08-security-engineering.md)
- [14 — Async jobs & scheduling](14-async-jobs-and-scheduling.md)
