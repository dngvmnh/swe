# Module 20 — Performance & Data Access

> **Core question:** Why is a request slow, and how do you make data access fast without guessing?
> **Source:** the Mythos crawl (`backend/specs/high-01-n-plus-one-queries.md`, frontend query hooks, `well-known.routes.ts`, client singletons, the chat PRD).
> **Prerequisites:** [03 — Database & migrations](03-database-migrations.md), [07 — API design](07-api-design.md), [12 — Frontend](12-frontend.md).

## Why this module
Most slowness isn't CPU — it's **waiting on data**: too many round trips, unindexed scans, refetching things that didn't change. The Mythos backend spec for one endpoint went from *7–9 sequential queries (~350–900ms)* to *3–4 query rounds (~100–200ms)* with three mechanical fixes. This module teaches those fixes plus the caching layers that keep them fast — and the discipline to **measure** instead of guess.

### What you'll be able to do
- Recognize and kill N+1 queries (batch, merge, parallelize).
- Index the hot path and paginate with keyset cursors instead of offsets.
- Configure a client-side server-cache (staleTime/gcTime) and dependent queries.
- Add HTTP caching to public, slow-changing endpoints.
- Reuse expensive connections with lazy-singleton client factories.
- Quantify a performance win instead of guessing at it.

---

## Lesson 20.1 — Kill N+1 queries: batch, merge, parallelize

**Goal:** turn many sequential round trips into a few.

### Concept
An **N+1 query** is one query to get a list, then one *more* query per item. The three fixes, in order of preference:
1. **Batch with `.in()`** — fetch all the related rows in one query keyed by a list of ids.
2. **Merge redundant queries** — if two queries hit the same table for overlapping ids, do it once and split in memory.
3. **Parallelize independent reads** — queries that don't depend on each other run with `Promise.all`, so you pay the *max* latency, not the *sum*.

### Worked example — `backend/specs/high-01-n-plus-one-queries.md`
```js
// ❌ N+1 / redundant: fetch interest categories, then a SEPARATE round-trip for the listing's own category
const { data: cats } = await supabase.from("categories").select("category_id, name").in("category_id", ids);
const { data: cat } = await supabase.from("categories").select("name").eq("category_id", listing.category_id).single(); // redundant

// ✅ merge: include the listing's category in the SAME .in() batch, resolve from a Map
const allCategoryIds = [...new Set([...ids, listing.category_id].filter(Boolean))];
const { data: cats } = await supabase.from("categories").select("category_id, name").in("category_id", allCategoryIds);
const nameById = new Map(cats.map(c => [c.category_id, c.name]));
const categoryName = nameById.get(listing.category_id) ?? null;
```
```js
// ✅ parallelize: user, order count, and media don't depend on each other → one round, not three
const [userResult, orderCountResult, mediaResult] = await Promise.all([
  supabase.from("users").select("...").eq("user_id", listing.user_id).single(),
  supabase.from("orders").select("order_id", { count: "exact" }).eq("listing_id", listingId).in("status", ["paid", "fulfilled"]),
  supabase.from("listing_media").select("...").eq("listing_id", listingId).order("order", { ascending: true }),
]);
```

### ❌ Bad → ✅ Good
```js
// ❌ BAD — N+1: a query per id in a loop (N round trips)
const names = [];
for (const id of ids) { const { data } = await db.from("categories").select("name").eq("category_id", id).single(); names.push(data.name); }
```
```js
// ✅ GOOD — one batched query, look up in a Map
const { data } = await db.from("categories").select("category_id,name").in("category_id", ids);
const nameById = new Map(data.map(c => [c.category_id, c.name]));
const names = ids.map(id => nameById.get(id));
```

### Check yourself
1. Why does `Promise.all` over independent reads cut latency? (You wait for the slowest, not the sum; sequential `await`s add up.)
2. When can you *not* parallelize? (When a query needs a value the previous one returned — those stay sequential; only independent reads parallelize.)

### ✅ Lesson 20.1 checklist
- [ ] 📖 Read & understood — I can name the three N+1 fixes and when each applies.
- [ ] 💻 Applied — I replaced a per-item query loop with one `.in()` batch + a Map.
- [ ] 🔍 Found in Mythos — `backend/specs/high-01-n-plus-one-queries.md` (Fix 1/2/3).

---

## Lesson 20.2 — Index the hot path; paginate with keyset cursors

**Goal:** make filtered/sorted reads fast and stable under growth.

### Concept
- **Index the hot path.** Any column you filter or sort on at scale needs an index — often a **composite** index matching the query's `WHERE` + `ORDER BY`. The chat schema indexes exactly its access patterns.
- **Keyset (cursor) pagination over offset.** `OFFSET 10000` makes the DB scan and discard 10k rows — it gets slower as you page deeper, and rows shift if data changes between pages. **Keyset** pagination uses the last row's sort key as a `cursor` (`WHERE created_at < $cursor ORDER BY created_at DESC LIMIT n`) — constant time at any depth, stable under inserts.

### Worked example — `CHAT_FUNCTION_PRD.md`
```sql
-- composite indexes that match the query shape (filter + sort)
CREATE INDEX ON messages (conversation_id, created_at desc);
CREATE INDEX ON conversations (last_message_at desc);
```
```
GET /api/chat/conversations/:id/messages?cursor=<message_id|created_at>&limit=30
-- cursor pagination by message_id/created_at — not ?page=N&offset=...
```

### ❌ Bad → ✅ Good
```sql
-- ❌ BAD — offset pagination: scans+discards `offset` rows; slower as you page deeper, unstable under inserts
SELECT * FROM messages WHERE conversation_id = $1 ORDER BY created_at DESC LIMIT 30 OFFSET 9000;
-- ✅ GOOD — keyset: jump straight to the cursor, constant time, stable
SELECT * FROM messages WHERE conversation_id = $1 AND created_at < $cursor ORDER BY created_at DESC LIMIT 30;
```

### Check yourself
1. Why does an index on `(conversation_id, created_at desc)` serve `WHERE conversation_id=? ORDER BY created_at DESC` well? (The composite matches both the filter and the sort, so the DB walks the index instead of sorting a scan.)
2. What breaks with offset pagination on a busy feed? (New inserts shift offsets → users see duplicates/skips; and deep offsets get slow.)

### ✅ Lesson 20.2 checklist
- [ ] 📖 Read & understood — I can explain composite indexes and keyset vs offset pagination.
- [ ] 💻 Applied — I added a composite index and converted an offset pager to a cursor pager.
- [ ] 🔍 Found in Mythos — `CHAT_FUNCTION_PRD.md` §8.2 indexes + §9.3 cursor pagination.

---

## Lesson 20.3 — Client-side server-cache (staleTime/gcTime, dependent queries, invalidation)

**Goal:** stop the frontend from refetching unchanged server data.

### Concept
A server-state cache (TanStack Query) keys data by a **`queryKey`** and reuses it. Two knobs matter:
- **`staleTime`** — how long data is considered fresh (no refetch on remount/refocus).
- **`gcTime`** — how long unused data stays cached before garbage collection.

Plus two patterns: **dependent queries** gate a fetch behind `enabled` until its input exists, and **invalidation** by `queryKey` refetches after a mutation.

### Worked example — `frontend-main/src/hooks/listing/use-listing.ts`
```ts
export const useListing = (listingId?: string) =>
  useQuery({
    queryKey: ['listing', listingId],            // identity for caching + invalidation
    queryFn: () => fetchListing(listingId!),
    staleTime: 5 * 60 * 1000,  // fresh for 5 min — no refetch on remount/refocus
    gcTime: 10 * 60 * 1000,    // keep cached 10 min after last use
    retry: 1,
  });
```
```ts
// dependent query: don't fire until we have the id
useQuery({ queryKey: ['profile', userId], queryFn: () => fetchProfile(userId!), enabled: !!userId });
// after a mutation, invalidate the key to refetch
queryClient.invalidateQueries({ queryKey: ['listing', listingId] });
```

### ✅ Lesson 20.3 checklist
- [ ] 📖 Read & understood — I can explain staleTime/gcTime, dependent queries, and queryKey invalidation.
- [ ] 💻 Applied — I configured a cached query with a stale time and invalidated it after a mutation.
- [ ] 🔍 Found in Mythos — `use-listing.ts` (staleTime/gcTime/retry); `use-profile.ts`/`use-dashboard-role.ts` (`enabled`).

---

## Lesson 20.4 — HTTP caching for public, slow-changing endpoints

**Goal:** let the network cache what rarely changes.

### Concept
A public, slow-changing feed (like JWKS — keys change only on rotation) should send a **`Cache-Control`** header so browsers/CDNs/clients cache it instead of refetching every time. This is the server-side complement to the SDK's in-memory JWKS cache ([Module 17](17-sdk-library-design.md)) — both say "this is good for ~10 minutes."

### Worked example — `backend/src/routes/well-known.routes.ts`
```ts
router.get('/jwks.json', async (_req, res, next) => {
  try {
    const jwks = await getGlobalActiveJwks();
    res.set('Cache-Control', 'public, max-age=600');   // cacheable for 10 min by clients/CDNs
    return res.json(jwks);
  } catch (err) { return next(err); }
});
```

### Check yourself
1. Why is `Cache-Control: public, max-age=600` safe for JWKS but wrong for a wallet balance? (Keys change rarely and overlap during rotation; a balance changes per transaction and must never be cached stale.)

### ✅ Lesson 20.4 checklist
- [ ] 📖 Read & understood — I can choose Cache-Control for a public slow-changing resource and explain when *not* to cache.
- [ ] 💻 Applied — I added a `Cache-Control` header to a cacheable endpoint.
- [ ] 🔍 Found in Mythos — `well-known.routes.ts` (`Cache-Control: public, max-age=600`).

---

## Lesson 20.5 — Reuse connections with lazy-singleton client factories

**Goal:** create expensive clients once, not per request.

### Concept
Constructing a DB/HTTP/SDK client (TLS handshake, pool setup, key validation) is expensive. A **lazy singleton** creates it on first use and reuses it forever: a module-level `let client = null`, returned if present, otherwise built once. This also gives you a single place to validate config (fail-fast) — and it's the Dependency-Inversion factory from [Module 13](13-design-principles.md) (`getResend()`, `getStripeClient()`), so services never `new Client()` inline.

### Worked example — `backend/src/utils/email/resendClient.ts`
```ts
let client: Resend | null = null;
async function getResend(): Promise<Resend> {
  if (client) return client;                  // reuse — no per-call construction
  await requireEnv(['RESEND_API_KEY']);       // fail fast with a clear error if missing
  client = new Resend(await getEnv('RESEND_API_KEY') as string);
  return client;
}
```

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — a fresh client per call: repeated setup, no pooling, scattered config reads
function sendEmail(...) { const resend = new Resend(process.env.RESEND_API_KEY!); /* ... */ }
```
```ts
// ✅ GOOD — one shared instance via a factory
function sendEmail(...) { const resend = await getResend(); /* ... */ }
```

### ✅ Lesson 20.5 checklist
- [ ] 📖 Read & understood — I can explain lazy singletons and why per-request client creation is wasteful.
- [ ] 💻 Applied — I wrapped a client in a lazy-singleton factory with fail-fast config.
- [ ] 🔍 Found in Mythos — `resendClient.ts` (`getResend`), `stripeClient.ts` (`getStripeClient`).

---

## Lesson 20.6 — Measure, don't guess

**Goal:** justify an optimization with numbers.

### Concept
Performance work without measurement is superstition. State the **before/after** so the win is real and the effort is justified — and only optimize a confirmed bottleneck (the frontend constitution: *"memoisation only after profiling confirms a real bottleneck"*). The N+1 spec quantifies its fix explicitly.

### Worked example — the N+1 spec's impact table
```
| Before                      | After                       |
| 7–9 sequential queries      | 3–4 query rounds (parallel) |
| ~350–900ms DB wait          | ~100–200ms DB wait          |
```
The fix is mechanical; the *table* is what makes it a decision instead of a hunch.

### ✅ Lesson 20.6 checklist
- [ ] 📖 Read & understood — I can explain "measure before/after; optimize confirmed bottlenecks only."
- [ ] 💻 Applied — I measured a request's query count or latency before and after a change.
- [ ] 🔍 Found in Mythos — the impact table in `high-01-n-plus-one-queries.md`; FE constitution VII.

---

## 🎯 Module 20 mastery checklist
- [ ] I can spot an N+1 and fix it by batching, merging, and parallelizing.
- [ ] I can add composite indexes for a query's filter+sort and use keyset pagination.
- [ ] I can configure a client server-cache (staleTime/gcTime), dependent queries, and invalidation.
- [ ] I can add HTTP caching to a public slow-changing endpoint (and know when not to).
- [ ] I can reuse expensive clients with a lazy-singleton factory.
- [ ] I can quantify a performance change with before/after numbers.

## 🛠️ Mini-project
Take a slow "listing detail" endpoint:
1. Instrument it to count DB round trips. Introduce an N+1 (per-tag query), then fix it with `.in()` + a Map.
2. Parallelize the independent reads with `Promise.all`; record before/after query count + latency.
3. Add a composite index for its hottest list query and convert that list to keyset pagination.
4. On the client, wrap the fetch in a cached query (staleTime 5 min) and invalidate it after an edit.
5. Add `Cache-Control` to one public, slow-changing route, and put one client behind a lazy singleton.

## 🔗 Mythos source map
| File / PR | What it demonstrates |
|-----------|----------------------|
| `backend/specs/high-01-n-plus-one-queries.md` | Batch `.in()`, merge redundant, parallelize; before/after table. |
| `CHAT_FUNCTION_PRD.md` (§8.2, §9.3) | Composite indexes + keyset/cursor pagination. |
| `frontend-main/src/hooks/listing/use-listing.ts` | TanStack Query staleTime/gcTime; dependent queries via `enabled`. |
| `backend/src/routes/well-known.routes.ts` | `Cache-Control` on a public feed. |
| `backend/src/utils/email/resendClient.ts`, `stripeClient.ts` | Lazy-singleton client factories. |

## See also
- [03 — Database design & migrations](03-database-migrations.md) — indexing as a business rule.
- [07 — API design](07-api-design.md) · [12 — Frontend](12-frontend.md) — where these reads live.
- [21 — Observability & operations](21-observability-operations.md) — measuring in production.
