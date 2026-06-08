# 07 — API Design & Layered Architecture

## The idea

Two related disciplines:

- **API design** — the contract your server exposes to the outside world: the URLs, verbs,
  payload shapes, status codes, and error formats. It's the part clients depend on and the part
  you can't change casually.
- **Layered architecture** — how you organize the code *behind* that contract into layers with
  one-directional dependencies, so each layer has a single concern and changes stay contained.

The classic web layering: **route/controller** (HTTP in/out) → **service** (business logic) →
**data/adapter** (DB, external APIs). Dependencies point downward only.

## Why it exists

A server that mixes HTTP parsing, business rules, and SQL in one function is impossible to test,
reuse, or change safely — every edit risks unrelated breakage. Separating concerns means you can
unit-test business logic without HTTP, swap the database without rewriting rules, and reason about
one layer at a time. A consistent API contract means clients don't break every time you refactor
internals.

## Patterns & trade-offs

- **Strict layering, one-way dependencies.** Routes call services; services call adapters.
  *Never* the reverse, and never skip a layer (no SQL in a route). Cross-cutting shared logic goes
  to a `utils/` layer both can use — services never import each other.
- **Thin controllers, fat services.** The route handler only validates input and shapes the HTTP
  response; all decisions live in the service. Keep handlers short (a hard line — e.g. ≤30 lines).
- **REST conventions.** Nouns not verbs, plural resource paths, lowercase-hyphenated; HTTP verbs
  carry the action (`GET` read, `POST` create, etc.); status codes carry the outcome (200/201/400/
  401/403/404/409/500).
- **A consistent response envelope.** Every response has the same shape — e.g.
  `{ success, data }` or `{ success: false, error, details }` — so clients have one parsing path.
- **Input validation at the boundary.** Validate and sanitize *before* any business logic runs;
  reject early with a 400 and machine-readable details. Keep validation rules in their own layer.
- **Typed contracts / DTOs.** Shared types describe what crosses each boundary, so the compiler
  catches shape drift. Use *narrow* types (only the fields a function needs).
- **Atomicity at the right layer.** Multi-step writes that must succeed-or-fail together belong in
  a transaction or a database function (RPC), invoked by the service — not stitched together with
  separate calls from a handler.

Trade-off: layering adds indirection and boilerplate (a tiny CRUD endpoint touches three files).
You pay that to keep large systems changeable; for a throwaway script it's overkill.

## Pitfalls

- **DB calls in route handlers / HTTP logic in services** — the layering violation that erases all
  the benefits.
- **Services returning `{success:false}` instead of throwing** — mixes control-flow styles and
  forces every caller to re-check (see [doc 06](06-error-handling-and-typed-errors.md)).
- **Inline anonymous types at boundaries** — duplicated, undocumented contracts that drift.
- **Inconsistent envelopes / status codes** — every endpoint a special case for the client.
- **Fat objects passed around** when a function needs two fields — couples unrelated code.
- **Validation after the service call** — business logic runs on garbage input.

## Seen in Mythos

- **The layering is constitutional.** Mythos mandates: *Route handler → validation + HTTP response
  only; Service function → all business logic, DB queries, external calls; DB/External adapter →
  factory singletons. No DB calls in routes. No HTTP logic in services. No cross-service imports —
  shared logic goes to `src/utils/`.* This is exactly the three-layer model above.

- **A clean vertical slice — `BE#88`.** The billing feature shows every layer doing its job:
  `apps.routes.ts` only runs `validationResult`, extracts the body, calls the service, and shapes
  `{ success, data }`; `apps.service.ts` holds all the logic and throws typed errors; the
  multi-step money mutation lives in a Postgres **RPC** (`metered_charge`) for atomicity, invoked
  by the service. The review even pushed an inline body type into a named `MeterSessionBody` DTO in
  `src/types/billing.ts` — the "no inline contracts" rule.

- **Validation at the boundary — `BE#88`.** `apps-validation.ts` uses `express-validator` to
  require a UUID `jti`, a positive-integer `credits`, and a UUID `charge_id` *before* the service
  is ever called; the handler returns 400 with `details: errors.array()` on failure.

- **Typed contracts as the migration's whole point — `BE#31`–`BE#47`.** The TS fan-out
  (see [doc 05](05-incremental-migration-strangler-fig.md)) was titled "typed contracts" precisely
  because it gave every layer boundary an explicit, compiler-checked DTO. `BE#35` scaffolded empty
  DTO files for 13 domains as the foundation.

- **REST + envelope conventions.** The endpoints follow the convention: `POST /api/apps/:id/launch`,
  `POST /api/apps/sessions/:jti/consume`, `POST /api/apps/sessions/:jti/meter`,
  `POST /api/wallet/packs/:slug/checkout` (`BE#80`), `POST /api/wallet/subscription/checkout`
  (`BE#85`) — plural nouns, resource-scoped, consistent `{ success, data }` responses and the full
  status-code vocabulary (200/400/401/402/404/409).

- **DRY shared logic, not cross-imports — `FM#20`.** A duplicated `uploadFileToS3` in two API
  modules was deduplicated into one shared utility — the "shared logic lives in one place, layers
  don't reach sideways" rule. (More in [doc 13](13-design-principles-solid-dry.md).)

## See also

- [06 — Error handling & typed errors](06-error-handling-and-typed-errors.md)
- [13 — Design principles: SOLID, DRY](13-design-principles-solid-dry.md)
- [04 — Authentication & authorization](04-authentication-and-authorization.md) (middleware layer)
