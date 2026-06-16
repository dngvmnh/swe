# 🎓 The Learning Layer

[![Lessons](https://img.shields.io/endpoint?url=https%3A%2F%2Fraw.githubusercontent.com%2Fdngvmnh%2Fswe%2Fmain%2Fdocs%2Fprogress.json)](PROGRESS.md)
[![Live dashboard](https://img.shields.io/badge/live-dashboard-2f81f7?logo=github)](https://dngvmnh.github.io/swe/)

This folder turns the [reference field guide](../README.md) (the 15 dense docs in the repo root)
into a **course you can work through and track**.

> 📊 **Prefer a visual?** The **[live dashboard](https://dngvmnh.github.io/swe/)** renders your
> committed progress as bars and a completion ring — it reads `PROGRESS.md` straight from GitHub.

- The **reference docs** (`../01-*.md` … `../15-*.md`) answer *"what is the canonical idea, and
  where does it show up in Mythos?"* — they are the encyclopedia.
- The **learning modules** here (`learn/01-*.md` … `learn/15-*.md`) answer *"how do I actually
  learn this, step by step, with examples I can run and a way to prove I've got it?"* — they are
  the classroom.

Track everything from one place: **[PROGRESS.md](PROGRESS.md)** — your live dashboard.

---

## How a module is built

Every module follows the same shape so you always know where you are:

```
Module NN — Topic
├── Why this module / what you'll be able to do   ← the goal
├── Prerequisites                                  ← what to learn first
├── Lesson NN.1, NN.2, …  (3–6 bite-sized lessons) ← the actual learning
│     ├── Concept            (plain-English idea)
│     ├── Worked example     (real TypeScript / SQL you can read & run)
│     ├── ❌ Bad → ✅ Good    (the mistake and the fix, side by side)
│     ├── Check yourself     (a few self-quiz questions)
│     └── ✅ Lesson checklist (3 tick-boxes: read · applied · found-in-Mythos)
├── 🎯 Module mastery checklist                     ← higher-order "I can do X"
├── 🛠️ Mini-project                                 ← build something to prove it
└── 🔗 Mythos PR map + See also                     ← back to the real diffs
```

### The 3-tick learning gradient

Each lesson ends with the same three checkboxes. They are deliberately ordered from *passive* to
*active* — ticking all three is what "I learned it" means:

- [ ] 📖 **Read & understood** — I can explain the idea in my own words.
- [ ] 💻 **Applied** — I wrote/ran the example (or my own version) myself.
- [ ] 🔍 **Found in Mythos** — I opened the cited PR and saw the idea in real code.

> **How to tick a box on GitHub:** open the file on github.com → click the ✏️ (pencil) → change
> `- [ ]` to `- [x]` → *Commit changes*. The box renders as checked for everyone, and your commit
> history becomes a timeline of what you learned and when. (In a local editor, just edit the file
> and push.)

---

## 🗺️ The 21 modules

Read top-to-bottom the first time — the foundations (01–03) hold up everything after them.
Modules **01–15** are the core field guide (each backed by a root reference doc). Modules **16–21**
are deep-dives mined from a full crawl of the Mythos backend, frontends, and SDK — grounded directly
in real files rather than a reference doc.

### Core field guide (01–15)

| #  | Module | Core question | Reference |
|----|--------|---------------|-----------|
| 01 | [Idempotency & exactly-once](01-idempotency.md) | How do you make an operation safe to retry? | [↗](../01-idempotency-and-exactly-once-semantics.md) |
| 02 | [Concurrency, locking & races](02-concurrency.md) | What breaks when two requests arrive at once? | [↗](../02-concurrency-locking-and-race-conditions.md) |
| 03 | [Database design & migrations](03-database-migrations.md) | How do you evolve a schema without downtime? | [↗](../03-database-design-and-schema-migrations.md) |
| 04 | [Auth & authorization](04-auth.md) | Who is calling, and what may they do? | [↗](../04-authentication-and-authorization.md) |
| 05 | [Incremental migration (Strangler Fig)](05-incremental-migration.md) | How do you replace a live system? | [↗](../05-incremental-migration-strangler-fig.md) |
| 06 | [Error handling & typed errors](06-error-handling.md) | How do failures travel cleanly? | [↗](../06-error-handling-and-typed-errors.md) |
| 07 | [API design & layered architecture](07-api-design.md) | How do you keep a server changeable? | [↗](../07-api-design-and-layered-architecture.md) |
| 08 | [Security engineering](08-security.md) | How do you keep attackers out? | [↗](../08-security-engineering.md) |
| 09 | [Payments & billing](09-payments.md) | How do you move money correctly? | [↗](../09-payments-and-billing-architecture.md) |
| 10 | [Testing & quality gates](10-testing.md) | How do you trust a change before shipping? | [↗](../10-testing-strategy-and-quality-gates.md) |
| 11 | [DevOps, CI/CD & deployment](11-devops-cicd.md) | How does code get to production? | [↗](../11-devops-cicd-and-deployment.md) |
| 12 | [Frontend architecture & rendering](12-frontend.md) | How do you build a UI that scales & renders fast? | [↗](../12-frontend-architecture-and-rendering.md) |
| 13 | [SOLID, DRY & clean code](13-design-principles.md) | What makes code easy to change later? | [↗](../13-design-principles-solid-dry.md) |
| 14 | [Async jobs & scheduling](14-async-jobs.md) | How do you work outside the request cycle? | [↗](../14-async-jobs-and-scheduling.md) |
| 15 | [Spec-driven dev & governance](15-spec-governance.md) | How does a team decide & remember how it builds? | [↗](../15-spec-driven-development-and-governance.md) |

### Mythos codebase deep-dives (16–21)

These came out of the codebase crawl — every example is copied or faithfully adapted from a real file.

| #  | Module | Core question | Grounded in |
|----|--------|---------------|-------------|
| 16 | [Cryptography & key management](16-cryptography-key-management.md) | How do you store and rotate signing keys without leaking them? | `crypto/signing-keys.ts`, JWKS, rotation RPC |
| 17 | [SDK & library design](17-sdk-library-design.md) | How do you design a library others integrate? | the Python SDK (`mythos_sdk/*`) |
| 18 | [Resilience & failure modes](18-resilience-failure-modes.md) | What should code do when a step fails? | fail-open/closed, sagas, reconciliation |
| 19 | [Integration & protocol design](19-integration-protocol-design.md) | How do two systems agree on a trustworthy protocol? | launch/consume/meter, ADRs, contract drift |
| 20 | [Performance & data access](20-performance-data-access.md) | Why is a request slow, and how do you fix it? | N+1 spec, keyset pagination, query caching |
| 21 | [Observability & operations](21-observability-operations.md) | How do you see and operate it in production? | structured logging, request IDs, runbooks |

---

## Suggested learning paths

You don't have to go in order. Pick the track that matches why you're here.

- **🧱 Backend correctness (the Mythos core):** 01 → 02 → 03 → 09 → 14 → 06 → 18
- **🔌 Building an API from scratch:** 07 → 06 → 04 → 10 → 08 → 20
- **🎨 Frontend track:** 12 → 13 → 05 → 08 → 20
- **🚢 Ship & operate:** 11 → 10 → 03 → 15 → 21
- **🔐 Platform / SDK / integration (the deep-dives):** 16 → 17 → 19 → 18 → 21
- **📐 "Make me a better engineer" (no order assumed):** 13 → 06 → 07 → 10 → 15

---

## The Mythos system in one paragraph

Mythos is a **two-sided marketplace with a unified token economy**: users buy credits once and
spend them across any product; creators publish "web apps" that consumers launch and get metered
for; 85% of each charge flows to the creator. Backend is **Node/Express on Supabase (Postgres)**
with **Stripe** for money and **AWS** for infra; the frontend migrated from a **Vite/React SPA**
to **Next.js**. Most of the engineering difficulty — and most of these modules — is about
**correctness under failure and concurrency, especially where money is involved.** Code examples
in this course use that stack (TypeScript + SQL) so what you learn maps straight onto the real PRs.

> PR references use the form `BE#88` (backend), `FE#5` (frontend), `FM#43` (frontend-main).
> Every `BE#`/`FM#` tag links to a real diff you can open on GitHub.
