# Module 07 — API Design & Layered Architecture

> **Core question:** How do you structure a server so it stays changeable?
> **Reference doc:** [../07-api-design-and-layered-architecture.md](../07-api-design-and-layered-architecture.md)
> **Prerequisites:** [06 — Error handling](06-error-handling.md); pairs with [13 — Design principles](13-design-principles.md).

## Why this module
A server that mixes HTTP parsing, business rules, and SQL in one function is impossible to test,
reuse, or change safely. This module teaches you to separate concerns into layers and to expose a
stable, consistent API contract that clients can depend on.

### What you'll be able to do
- Organize a server into route → service → adapter layers with one-way dependencies.
- Keep controllers thin and services fat.
- Design REST endpoints with a consistent response envelope and status-code vocabulary.
- Validate at the boundary, use typed DTOs, and put atomicity at the right layer.

---

## Lesson 7.1 — The three layers (dependencies point down)

**Goal:** internalize the canonical web layering.

### Concept
```
route / controller   →   service             →   data / adapter
(HTTP in & out)          (business logic)         (DB, external APIs)
  validate input          all decisions            factory singletons
  shape the response      throws typed errors      SQL / Stripe / S3
```
**Dependencies point downward only.** Routes call services; services call adapters. *Never* the
reverse, and never skip a layer (**no SQL in a route**). Cross-cutting shared logic goes to a
`utils/` layer both can use — **services never import each other**.

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — SQL and business logic in the route handler
app.post("/apps/:id/meter", async (req, res) => {
  const { rows } = await pg.query("SELECT * FROM wallets WHERE ...");  // DB in a route!
  if (rows[0].balance < req.body.credits) return res.status(402).json({...});
  // ...money math inline...
});
```
```ts
// ✅ GOOD — route delegates; service decides; adapter touches the DB
app.post("/apps/:id/meter", validate, async (req, res) => {
  const result = await meterService.meter(req.params.id, req.body);   // thin controller
  res.json({ success: true, data: result });
});
```

### ✅ Lesson 7.1 checklist
- [ ] 📖 Read & understood — I can draw the three layers and the one-way dependency rule.
- [ ] 💻 Applied — I identified a layering violation (DB in a route, HTTP in a service) somewhere.
- [ ] 🔍 Found in Mythos — the layering is constitutional: "No DB calls in routes. No HTTP logic in services. No cross-service imports."

---

## Lesson 7.2 — Thin controllers, fat services

**Goal:** put all decisions in the service.

### Concept
The route handler **only** validates input and shapes the HTTP response; **all decisions live in the
service.** Keep handlers short (a hard line — e.g. ≤30 lines). This is what lets you unit-test
business logic without HTTP and reuse it from a cron or a queue worker.

```ts
// route: extract + delegate + shape — nothing more
async function meterHandler(req: Request, res: Response) {
  if (!validationResult(req).isEmpty())
    return res.status(400).json({ success: false, error: "VALIDATION", details: validationResult(req).array() });
  const data = await appsService.meterSession(req.params.jti, req.body); // ALL logic here
  res.json({ success: true, data });
}
```

### ✅ Lesson 7.2 checklist
- [ ] 📖 Read & understood — I can explain why handlers stay thin and services hold the logic.
- [ ] 💻 Applied — I extracted business logic out of a fat controller.
- [ ] 🔍 Found in Mythos — `BE#88` `apps.routes.ts` only runs `validationResult`, calls the service, shapes `{ success, data }`.

---

## Lesson 7.3 — REST conventions + a consistent envelope

**Goal:** make the contract predictable.

### Concept
- **REST conventions.** Nouns not verbs, plural resource paths, lowercase-hyphenated. HTTP verbs
  carry the action (`GET` read, `POST` create…); status codes carry the outcome.
- **A consistent response envelope.** Every response has the same shape so clients have **one
  parsing path**.

```ts
// success
{ "success": true,  "data": { /* ... */ } }
// failure
{ "success": false, "error": "INSUFFICIENT_FUNDS", "details": "wallet balance too low" }
```

**Status-code vocabulary you should know cold:**

| Code | Meaning | Example |
|------|---------|---------|
| 200 / 201 | OK / Created | read OK / resource created |
| 400 | Bad request (validation) | missing/invalid field |
| 401 | Unauthenticated | no/invalid token |
| 402 | Payment required | insufficient funds |
| 403 | Forbidden | authenticated but not allowed |
| 404 | Not found | unknown resource |
| 409 | Conflict | already consumed / duplicate |
| 500 | Internal error | unexpected failure |

### ✅ Lesson 7.3 checklist
- [ ] 📖 Read & understood — I can recite the status-code vocabulary and the envelope shape.
- [ ] 💻 Applied — I designed 3 REST endpoints with consistent envelopes.
- [ ] 🔍 Found in Mythos — `POST /api/apps/:id/launch`, `…/sessions/:jti/meter`, `…/wallet/packs/:slug/checkout` — plural nouns, resource-scoped, `{ success, data }`.

---

## Lesson 7.4 — Validate at the boundary

**Goal:** reject garbage before any business logic runs.

### Concept
Validate and sanitize **before** any business logic. Reject early with a **400** and machine-readable
details. Keep validation rules in their own layer.

```ts
// apps-validation.ts — runs BEFORE the service
export const meterValidation = [
  param("jti").isUUID(),
  body("credits").isInt({ gt: 0 }),
  body("charge_id").isUUID(),
];
// handler: if (!errors.isEmpty()) return res.status(400).json({ ..., details: errors.array() });
```

### ❌ Bad → ✅ Good
- ❌ Call the service, then discover the input was malformed deep inside → business logic ran on garbage.
- ✅ Validate at the boundary; the service only ever sees well-formed input.

### ✅ Lesson 7.4 checklist
- [ ] 📖 Read & understood — I can explain why validation must precede the service call.
- [ ] 💻 Applied — I added boundary validation that returns 400 with details.
- [ ] 🔍 Found in Mythos — `BE#88` `apps-validation.ts` requires UUID `jti`, positive-int `credits`, UUID `charge_id` before the service runs.

---

## Lesson 7.5 — Typed contracts / DTOs (narrow types)

**Goal:** let the compiler catch shape drift.

### Concept
Shared types describe what crosses each boundary, so the compiler catches drift. Use **narrow**
types — only the fields a function needs — not fat objects.

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — inline anonymous type at the boundary; duplicated and undocumented
function meter(body: { jti: string; credits: number; charge_id: string }) { /* ... */ }
// ❌ BAD — passing the whole User when you need two fields couples unrelated code
function notify(user: User) { send(user.email); }
```
```ts
// ✅ GOOD — a named DTO + a narrow type
// src/types/billing.ts
export interface MeterSessionBody { jti: string; credits: number; charge_id: string; }
function meter(body: MeterSessionBody) { /* ... */ }
function notify(user: Pick<User, "id" | "email">) { send(user.email); } // interface segregation
```

### ✅ Lesson 7.5 checklist
- [ ] 📖 Read & understood — I can explain named DTOs and narrow types vs fat objects.
- [ ] 💻 Applied — I replaced an inline boundary type with a named DTO.
- [ ] 🔍 Found in Mythos — `BE#88` review pushed an inline body type into a named `MeterSessionBody` DTO; `BE#31`–`#47` were "typed contracts."

---

## Lesson 7.6 — Atomicity at the right layer

**Goal:** put multi-step writes where they're safe.

### Concept
Multi-step writes that must **succeed-or-fail together** belong in a **transaction or a database
function (RPC)**, invoked by the service — *not* stitched together with separate calls from a
handler (where a crash mid-way leaves half-applied state).

```ts
// ✅ the service invokes ONE atomic RPC; debit + ledger + creator-payout all-or-nothing
const result = await db.rpc("metered_charge", { jti, credits, charge_id });
```

### ✅ Lesson 7.6 checklist
- [ ] 📖 Read & understood — I can explain why multi-step money writes belong in a transaction/RPC.
- [ ] 💻 Applied — I wrapped a multi-step write in a transaction.
- [ ] 🔍 Found in Mythos — `BE#88` the money mutation lives in the `metered_charge` Postgres RPC, invoked by the service.

---

## 🎯 Module 07 mastery checklist
- [ ] I can structure a server into route → service → adapter with one-way dependencies.
- [ ] I can keep controllers thin and put all logic in services.
- [ ] I can design REST endpoints with a consistent envelope and the right status codes.
- [ ] I can validate at the boundary and use named, narrow DTOs.
- [ ] I can place atomic multi-step writes in a transaction/RPC at the service layer.

## 🛠️ Mini-project
Build a small "todo" or "wallet" API with strict layering:
1. `routes/` (validation + response shaping only), `services/` (all logic, throws typed errors),
   `adapters/` (DB access via a factory).
2. One consistent `{ success, data }` / `{ success, error, details }` envelope across all endpoints.
3. Boundary validation returning 400 + details. Named DTOs for every request body.
4. One endpoint whose write spans 2+ tables — wrap it in a transaction. Add a test that a mid-write
   failure leaves **no** partial state.

## 🔗 Mythos PR map
| PR | What it demonstrates |
|----|----------------------|
| `BE#88` | A clean vertical slice: thin route, fat service, validation layer, atomic RPC, named DTO. |
| `BE#31`–`BE#47` | Typed contracts/DTOs per domain boundary. |
| `FM#20` | DRY shared util instead of cross-layer reach. |

## See also
- [06 — Error handling & typed errors](06-error-handling.md)
- [13 — Design principles: SOLID, DRY](13-design-principles.md)
- [04 — Authentication & authorization](04-auth.md) — the middleware layer.
