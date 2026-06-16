# Module 18 — Resilience & Failure Modes

> **Core question:** When a step in a multi-system operation fails — a network blip, a declined transfer, a half-applied batch — what should your code *do*?
> **Source:** the Mythos crawl (`mythos-sdk` middleware, `wallet.services.ts`, `payouts.service.ts`, `release-orders.ts`).
> **Prerequisites:** [01 — Idempotency](01-idempotency.md), [02 — Concurrency](02-concurrency.md), [09 — Payments](09-payments.md), [14 — Async jobs](14-async-jobs.md).

## Why this module
The other modules make the *happy path* correct. This one is about the *unhappy path*: a dependency is down, a provider call fails after you already wrote to your DB, one row in a batch is bad. The default behaviours most code falls into — fail open, leave half-written state, abort the whole batch — are exactly the ones that lose money or leak access. Resilience is choosing the *deliberate* failure behaviour instead.

### What you'll be able to do
- Choose fail-closed vs fail-open and know which one a security path demands.
- Undo a partial multi-system write with a compensating transaction (saga).
- Reconcile your state against an external source of truth.
- Process a batch so one bad row doesn't sink the rest.
- Compensate-and-continue instead of rolling back in a sweep, guarded by compare-and-set.
- Make external calls safely retryable with idempotency keys.

---

## Lesson 18.1 — Fail-closed vs fail-open

**Goal:** make the *secure* choice the default when a dependency fails.

### Concept
When a check can't complete (the service it depends on is unreachable), you either **fail open** (allow the action) or **fail closed** (deny it). For anything that gates access or spends a single-use right, **fail closed** — an error must never become a free pass. The Mythos SDK's launch middleware *must* confirm a single-use `consume` before granting access; if it can't reach `consume`, it returns `503`, never the protected resource.

### Worked example — `mythos-sdk/.../middleware.py`
```python
async def require_launch_token(lt: str = Query(..., alias="lt")) -> MythosSession:
    try:
        session = await verify_launch_token(lt)
    except Exception:
        raise HTTPException(status_code=401, detail="Invalid launch token")
    try:
        resp = await consume_session(session.sessionJti)
    except Exception:
        # Network error / unreachable consume — FAIL CLOSED, never grant single-use access unconfirmed.
        raise HTTPException(status_code=503, detail="Could not verify session")
    if resp.status_code == 409:
        raise HTTPException(status_code=401, detail="Token already consumed")
    if not (200 <= resp.status_code < 300):
        # Any other non-2xx (500, 503, ...) is UNCONFIRMED → fail closed.
        raise HTTPException(status_code=503, detail="Could not verify session")
    return session
```

### ❌ Bad → ✅ Good
```python
# ❌ BAD — fail OPEN: a consume error lets the request through, defeating single-use entirely
try:
    await consume_session(session.sessionJti)
except Exception:
    pass            # "best effort" → an attacker just needs consume to flake to replay tokens
return session
```
```python
# ✅ GOOD — fail CLOSED: unconfirmed consume → 503, access denied
except Exception:
    raise HTTPException(status_code=503, detail="Could not verify session")
```
> This was a real audited bug (FINDINGS F11): the SDK consume path failed open, so any `/consume` hiccup turned a single-use token into a replayable one.

### Check yourself
1. Which paths should fail closed, and which can fail open? (Access/spend/auth → closed. A non-critical enrichment (e.g. "load avatar") can fail open and degrade gracefully — Lesson 18.4.)
2. Why is a `503` here better than a `200`? (It tells the caller "couldn't verify — retry," instead of silently granting access on an unverified token.)

### ✅ Lesson 18.1 checklist
- [ ] 📖 Read & understood — I can decide fail-open vs fail-closed for a given path.
- [ ] 💻 Applied — I made a dependency-failure path fail closed on an access check.
- [ ] 🔍 Found in Mythos — `mythos-sdk` `middleware.py` `require_launch_token`; FINDINGS F11.

---

## Lesson 18.2 — Compensating transactions (the saga)

**Goal:** undo earlier steps when a later step in a cross-system operation fails.

### Concept
You can't wrap a DB write *and* a Stripe call in one transaction — they're different systems. So you do them in order and, if the external step fails, run a **compensating action** that reverses the earlier write. This is the **saga** pattern: forward step + a registered "undo." Mythos debits the wallet, then creates the Stripe transfer; if the transfer throws, it **credits the wallet back** (a reversal) before surfacing the error.

### Worked example — `wallet.services.ts` (cashout)
```ts
// 1) forward: debit the wallet
const { error: debitError } = await supabase.rpc('wallet_debit_topup', { p_user_id: userId, p_amount: parsedTokens, ... });
if (debitError) throw new AppError('Failed to update wallet after payout', 500, 'WALLET_UPDATE_ERROR');

// 2) external step: move the money
try {
  transfer = await stripe.transfers.create({ amount: amountMinor, currency: 'sgd', destination: user.stripe_account_id, ... });
} catch (err) {
  // 3) COMPENSATE: the debit already happened but the transfer failed → credit it back
  const { error: creditError } = await supabase.rpc('wallet_credit', {
    p_user_id: userId, p_amount: parsedTokens, p_bucket: 'topup',
    p_kind: 'adjustment', p_reference: 'cashout_transfer_reversal', p_stripe_event_id: null,
  });
  if (creditError) logger.error({ err: creditError }, 'wallet reversal credit error');
  throw new AppError('Failed to create Stripe transfer', 500, 'STRIPE_TRANSFER_ERROR');
}
```
Note the compensation is a *new* ledger entry (`adjustment` / `cashout_transfer_reversal`), not an edit — consistent with the append-only ledger ([Module 09](09-payments.md)).

### Check yourself
1. Why not just "roll back the debit"? (The debit is a committed, audited ledger row; you reverse it with a compensating credit, preserving the trail — you never mutate posted entries.)
2. What if the *compensation itself* fails? (Log it loudly for reconciliation — Lesson 18.3 — and alert; this is why money systems also run a reconciliation sweep.)

### ✅ Lesson 18.2 checklist
- [ ] 📖 Read & understood — I can explain the saga pattern and compensation vs rollback.
- [ ] 💻 Applied — I wrote a forward step + a compensating undo for a cross-system op.
- [ ] 🔍 Found in Mythos — `wallet.services.ts` cashout transfer reversal.

---

## Lesson 18.3 — Reconciliation loops (re-sync against the source of truth)

**Goal:** make your local state eventually match the external system that really owns it.

### Concept
After you fire an external action, your DB records "processing" — but the *external system* (Stripe) is the source of truth for whether it actually succeeded, reversed, or failed. A **reconciliation loop** periodically re-reads the external state and updates your records to match. It's the safety net under compensation (Lesson 18.2): even if a compensation was missed, reconciliation catches the drift.

### Worked example — `payouts.service.ts` (batch reconciliation)
```ts
for (const batch of batches) {
  if (!batch.stripe_transfer_id) continue;
  let transfer: Stripe.Transfer;
  try { transfer = await stripe.transfers.retrieve(batch.stripe_transfer_id); }
  catch (err) { logger.error({ err }, 'retrieve transfer error'); continue; }   // skip, try next sweep

  if (transfer.reversed || safeNum(transfer.amount_reversed) > 0) {
    // external truth says reversed → mark our records failed
    await supabase.from('payout_batches').update({ status: 'failed', failure_reason: 'Transfer was reversed', ... }).eq('batch_ref', batch.batch_ref);
    await supabase.from('payouts').update({ status: 'payout_failed', ... }).eq('batch_ref', batch.batch_ref);
    failed += 1; continue;
  }
  // external truth says success → mark complete + notify
  await supabase.from('payout_batches').update({ status: 'completed', ... }).eq('batch_ref', batch.batch_ref);
  completed += 1;
}
return { checked: batches.length, completed, failed };
```

### ✅ Lesson 18.3 checklist
- [ ] 📖 Read & understood — I can explain a reconciliation loop and why the external system is the source of truth.
- [ ] 💻 Applied — I wrote a job that re-reads external state and updates local records to match.
- [ ] 🔍 Found in Mythos — `payouts.service.ts` batch reconciliation against `stripe.transfers.retrieve`.

---

## Lesson 18.4 — Resilient batch processing (per-item isolation)

**Goal:** one bad row must not sink the batch.

### Concept
When processing many items, **isolate each one**: wrap per-item work in try/catch, on failure record it in a `skipped` list with a reason and `continue`, and return structured `{ processed, skipped }` accounting at the end. The batch always finishes; nothing is silently dropped (each skip has a reason — observable, see [Module 21](21-observability-operations.md)).

### Worked example — `payouts.service.ts` (batch payout)
```ts
const processed: ProcessedBatch[] = [];
const skipped: SkippedCreator[] = [];
for (const group of grouped.values()) {
  const creator = userMap.get(group.creator_id);
  if (!creator?.stripe_account_id) { skipped.push({ creator_id: group.creator_id, reason: 'No connected Stripe account' }); continue; }
  if (creator.stripe_payouts_enabled === false) { skipped.push({ ..., reason: 'Stripe payouts not enabled' }); continue; }
  try { transfer = await stripe.transfers.create({ ... }, { idempotencyKey: buildBatchIdempotencyKey(batchRef) }); }
  catch (err) {
    const reason = err instanceof Error ? err.message : 'Stripe transfer failed';
    await supabase.from('payout_batches').update({ status: 'failed', failure_reason: reason }).eq('batch_ref', batchRef);
    skipped.push({ creator_id: group.creator_id, reason }); continue;     // isolate + record + keep going
  }
  processed.push({ creator_id: group.creator_id, batchRef });
}
return { processed, skipped };
```

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — one throw aborts the whole batch; everyone after the bad row goes unpaid
for (const group of groups) { await payOut(group); }   // first failure kills the loop
```
```ts
// ✅ GOOD — per-item try/catch + skip/collect; the batch completes, failures are accounted for
for (const group of groups) { try { await payOut(group); processed.push(...); } catch (e) { skipped.push({ id: group.id, reason: String(e) }); } }
```

### ✅ Lesson 18.4 checklist
- [ ] 📖 Read & understood — I can explain per-item isolation and skip/collect accounting.
- [ ] 💻 Applied — I converted an abort-on-first-error loop into a resilient batch.
- [ ] 🔍 Found in Mythos — `payouts.service.ts` `processed`/`skipped` batch loop.

---

## Lesson 18.5 — Compensate-don't-rollback in a sweep + compare-and-set

**Goal:** advance state safely in a recurring sweep, even when a follow-up write fails.

### Concept
A cron sweep that flips order state shouldn't try to "roll back" across rows on a partial failure — it should use an **atomic compare-and-set** to claim each row, and if a *secondary* write fails, **log and move on** (a later sweep or reconciliation fixes it). The compare-and-set (`UPDATE … WHERE status = 'paid'`) guarantees each order is advanced exactly once even if the sweep overlaps ([Module 02](02-concurrency.md)).

### Worked example — `release-orders.ts`
```ts
for (const order of orders) {
  // compare-and-set: only the run that flips paid→fulfilled "wins" this order
  const { data: updatedOrder, error } = await supabase
    .from('orders').update({ status: 'fulfilled', updated_at: nowISO })
    .eq('order_id', order.order_id).eq('status', 'paid')   // guard: still 'paid'
    .select('order_id').maybeSingle();
  if (error) { logger.error({ orderId: order.order_id, err: error }, 'order update failed'); continue; }
  if (!updatedOrder) continue;   // someone else already advanced it — skip

  const { error: payoutErr } = await supabase.from('payouts').update({ status: 'fulfilled', ... }).eq('order_id', order.order_id);
  if (payoutErr) {
    logger.error({ orderId: order.order_id, err: payoutErr }, 'payout update failed');
    // don't rollback the order; log and move on — a later sweep reconciles
  }
}
```

### ✅ Lesson 18.5 checklist
- [ ] 📖 Read & understood — I can explain compare-and-set claiming and compensate-don't-rollback in a sweep.
- [ ] 💻 Applied — I wrote a sweep that claims rows atomically and continues past a secondary failure.
- [ ] 🔍 Found in Mythos — `release-orders.ts` `markDisputeWindowOver` (`.eq('status','paid')` guard + "don't rollback, log and move on").

---

## Lesson 18.6 — Idempotency keys make external calls retry-safe

**Goal:** ensure a retried external call doesn't double-act.

### Concept
Retries (Lessons 18.2–18.5 all retry) are only safe if the external call is **idempotent**. Payment processors support an **idempotency key** per logical operation: send the same key on a retry and the provider returns the original result instead of creating a second transfer. This is [Module 01](01-idempotency.md) applied at the *external* boundary — and it's what lets a webhook **and** a cron both attempt the same settlement without double-paying (two competing drivers reconciled by a shared key).

### Worked example — `payouts.service.ts`
```ts
transfer = await stripe.transfers.create(
  { amount: group.total_amount, currency: group.currency, destination: creator.stripe_account_id, ... },
  { idempotencyKey: buildBatchIdempotencyKey(batchRef) },   // same batch → same key → no double transfer on retry
);
```

### Check yourself
1. Why does a stable idempotency key make a webhook + cron pair safe? (Both derive the same key for the same settlement; whichever runs second is a no-op replay, not a second payout — see [Module 14](14-async-jobs.md) safety-net design.)

### ✅ Lesson 18.6 checklist
- [ ] 📖 Read & understood — I can explain provider idempotency keys and two-driver reconciliation.
- [ ] 💻 Applied — I added an idempotency key to a retried external call.
- [ ] 🔍 Found in Mythos — `payouts.service.ts` `buildBatchIdempotencyKey`; subscription-renew cron safety-net.

---

## 🎯 Module 18 mastery checklist
- [ ] I can choose fail-closed for access paths and fail-open for non-critical enrichment.
- [ ] I can implement a saga with a compensating action for a cross-system write.
- [ ] I can write a reconciliation loop that re-syncs against an external source of truth.
- [ ] I can process a batch with per-item isolation and skip/collect accounting.
- [ ] I can advance sweep state with compare-and-set and compensate-don't-rollback.
- [ ] I can make external calls retry-safe with idempotency keys.

## 🛠️ Mini-project
Simulate a flaky "payout" against a mock provider:
1. `debit → transfer`; if the transfer throws, compensate with a credit (append a reversal entry). Test the reversal fires.
2. Process a batch of 5 payouts where #3 always fails — assert the batch returns `{ processed: 4, skipped: [#3 with reason] }`.
3. Add a reconciliation job that reads the mock provider's final state and marks each record completed/failed.
4. Make the transfer idempotent: retry the same logical payout twice with the same key and assert only one transfer happens.
5. Add a `require_access` guard that fails **closed** when the mock auth service is down.

## 🔗 Mythos source map
| File / PR | What it demonstrates |
|-----------|----------------------|
| `mythos-sdk/.../middleware.py` | Fail-closed single-use access (FINDINGS F11). |
| `backend/src/services/wallet.services.ts` | Compensating transaction (cashout reversal). |
| `backend/src/services/payouts.service.ts` | Reconciliation loop, resilient batch, idempotency key. |
| `backend/src/utils/release-orders.ts` | Compare-and-set sweep, compensate-don't-rollback. |

## See also
- [01 — Idempotency](01-idempotency.md) · [02 — Concurrency](02-concurrency.md)
- [09 — Payments & billing](09-payments.md) — the ledger these compensations write to.
- [14 — Async jobs & scheduling](14-async-jobs.md) — sweeps, safety nets, batch resilience.
- [17 — SDK & library design](17-sdk-library-design.md) — where fail-closed lives in the SDK.
