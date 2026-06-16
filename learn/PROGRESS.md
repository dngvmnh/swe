# 📊 Progress Dashboard

[![Lessons](https://img.shields.io/endpoint?url=https%3A%2F%2Fraw.githubusercontent.com%2Fdngvmnh%2Fswe%2Fmain%2Fdocs%2Fprogress.json)](PROGRESS.md)
[![Live dashboard](https://img.shields.io/badge/live-dashboard-2f81f7?logo=github)](https://dngvmnh.github.io/swe/)

Your single source of truth for tracking the course. **122 lessons across 21 modules.**
The badge and the **[live dashboard](https://dngvmnh.github.io/swe/)** update automatically from
this file when you commit a ticked box.

> Modules **01–15** are the original field guide. Modules **16–21** were added from a deep crawl of
> the Mythos backend, both frontends, and the Python SDK — each lesson is grounded in real files.

> **How to tick a box on GitHub:** open this file on github.com → click the ✏️ pencil → change
> `- [ ]` to `- [x]` → *Commit changes*. Your commit history becomes a dated record of what you
> learned. (Locally: edit + push.) Each module file has finer-grained 3-tick checklists
> (📖 read · 💻 applied · 🔍 found-in-Mythos); this dashboard is the high-level roll-up.

**Legend:** `[ ]` not started · `[x]` done · ✅ = module mastery checklist · 🛠️ = mini-project built

---

## Overall progress

The **[live dashboard](https://dngvmnh.github.io/swe/)** computes these automatically from your ticks
below. (Manual tally if you prefer: count your `[x]`s.)

- **Lessons:** `___ / 122`
- **Module mastery checklists:** `___ / 21`
- **Mini-projects built:** `___ / 21`
- **Started on:** `__________`  ·  **Target finish:** `__________`

> Tick the boxes in the per-module sections below (they're the single source of truth — the dashboard
> and badge read them). The table is just a map.

### Core field guide (01–15)
| # | Module | Lessons |
|---|--------|:-------:|
| 01 | [Idempotency](01-idempotency.md) | 5 |
| 02 | [Concurrency & locking](02-concurrency.md) | 6 |
| 03 | [Database & migrations](03-database-migrations.md) | 6 |
| 04 | [Auth & authorization](04-auth.md) | 6 |
| 05 | [Incremental migration](05-incremental-migration.md) | 5 |
| 06 | [Error handling](06-error-handling.md) | 6 |
| 07 | [API design & layers](07-api-design.md) | 6 |
| 08 | [Security engineering](08-security.md) | 6 |
| 09 | [Payments & billing](09-payments.md) | 6 |
| 10 | [Testing & gates](10-testing.md) | 6 |
| 11 | [DevOps, CI/CD](11-devops-cicd.md) | 6 |
| 12 | [Frontend & rendering](12-frontend.md) | 6 |
| 13 | [SOLID, DRY, clean code](13-design-principles.md) | 6 |
| 14 | [Async jobs & scheduling](14-async-jobs.md) | 6 |
| 15 | [Spec-driven dev & governance](15-spec-governance.md) | 6 |

### Mythos codebase deep-dives (16–21)
| # | Module | Lessons |
|---|--------|:-------:|
| 16 | [Cryptography & key management](16-cryptography-key-management.md) | 5 |
| 17 | [SDK & library design](17-sdk-library-design.md) | 6 |
| 18 | [Resilience & failure modes](18-resilience-failure-modes.md) | 6 |
| 19 | [Integration & protocol design](19-integration-protocol-design.md) | 6 |
| 20 | [Performance & data access](20-performance-data-access.md) | 6 |
| 21 | [Observability & operations](21-observability-operations.md) | 5 |

---

## Module 01 — [Idempotency & exactly-once](01-idempotency.md)
- [ ] 1.1 What "idempotent" actually means
- [ ] 1.2 Why "exactly-once delivery" is a myth
- [ ] 1.3 The four mechanisms (and when to use each)
- [ ] 1.4 The graceful-replay contract
- [ ] 1.5 Idempotent jobs with no client (synthetic keys)
- [ ] ✅ Module 01 mastery checklist
- [ ] 🛠️ Mini-project: idempotent `POST /credits/grant`

## Module 02 — [Concurrency, locking & races](02-concurrency.md)
- [ ] 2.1 What a race condition is
- [ ] 2.2 Atomic single-statement updates
- [ ] 2.3 Optimistic concurrency control (OCC)
- [ ] 2.4 Pessimistic locking (`SELECT … FOR UPDATE`)
- [ ] 2.5 Unique constraints & isolation levels
- [ ] 2.6 Pitfalls: deadlocks, TOCTOU, races in time
- [ ] ✅ Module 02 mastery checklist
- [ ] 🛠️ Mini-project: reproduce & fix a double-spend

## Module 03 — [Database design & migrations](03-database-migrations.md)
- [ ] 3.1 Schema as a contract, migration as history
- [ ] 3.2 Forward-only, ordered, immutable
- [ ] 3.3 Additive vs destructive & expand/contract
- [ ] 3.4 Baseline migrations on a live DB
- [ ] 3.5 Modeling: normalization, ledgers, indexing
- [ ] 3.6 The pitfalls that actually bite
- [ ] ✅ Module 03 mastery checklist
- [ ] 🛠️ Mini-project: 3 migrations incl. expand/contract rename

## Module 04 — [Authentication & authorization](04-auth.md)
- [ ] 4.1 Authn vs authz
- [ ] 4.2 Sessions vs tokens
- [ ] 4.3 Symmetric vs asymmetric signing & JWKS
- [ ] 4.4 OAuth/OIDC & password storage
- [ ] 4.5 Required vs optional auth (the bug everyone ships)
- [ ] 4.6 Cookie security, claim shapes & clean logout
- [ ] ✅ Module 04 mastery checklist
- [ ] 🛠️ Mini-project: register/login + optional-auth done right

## Module 05 — [Incremental migration (Strangler Fig)](05-incremental-migration.md)
- [ ] 5.1 Big-bang vs incremental
- [ ] 5.2 Run old & new in parallel (strangler facade)
- [ ] 5.3 Sequencing: umbrella branch + foundation first
- [ ] 5.4 Parallel fan-out (workers)
- [ ] 5.5 The pitfalls
- [ ] ✅ Module 05 mastery checklist
- [ ] 🛠️ Mini-project: migrate one cross-cutting thing incrementally

## Module 06 — [Error handling & typed errors](06-error-handling.md)
- [ ] 6.1 Errors are values: classify, propagate, translate
- [ ] 6.2 A typed error hierarchy (the class owns the status)
- [ ] 6.3 Throw typed, catch central
- [ ] 6.4 Fail fast with guard clauses
- [ ] 6.5 Two audiences; fail loud, not silent
- [ ] 6.6 Translate the specific error to the right outcome
- [ ] ✅ Module 06 mastery checklist
- [ ] 🛠️ Mini-project: error hierarchy + global handler

## Module 07 — [API design & layered architecture](07-api-design.md)
- [ ] 7.1 The three layers (dependencies point down)
- [ ] 7.2 Thin controllers, fat services
- [ ] 7.3 REST conventions + a consistent envelope
- [ ] 7.4 Validate at the boundary
- [ ] 7.5 Typed contracts / DTOs (narrow types)
- [ ] 7.6 Atomicity at the right layer
- [ ] ✅ Module 07 mastery checklist
- [ ] 🛠️ Mini-project: strictly-layered API

## Module 08 — [Security engineering](08-security.md)
- [ ] 8.1 Trust boundaries, never trust input, defense in depth
- [ ] 8.2 Input validation & output encoding (XSS, injection)
- [ ] 8.3 Secrets management
- [ ] 8.4 Encryption in transit & at rest
- [ ] 8.5 Least privilege, security headers, rate limiting
- [ ] 8.6 Verify, don't trust, callbacks (webhooks)
- [ ] ✅ Module 08 mastery checklist
- [ ] 🛠️ Mini-project: fix a stored-XSS hole + harden a webhook

## Module 09 — [Payments & billing](09-payments.md)
- [ ] 9.1 Integer minor units (never floats)
- [ ] 9.2 Append-only ledger (double-entry flavored)
- [ ] 9.3 Credit economy; processor at the boundary
- [ ] 9.4 Bucketed balances + drain order
- [ ] 9.5 Idempotency + locking + atomic RPC (the core)
- [ ] 9.6 Revenue split & subscription lifecycle
- [ ] ✅ Module 09 mastery checklist
- [ ] 🛠️ Mini-project: a minimal correct wallet

## Module 10 — [Testing strategy & quality gates](10-testing.md)
- [ ] 10.1 Tests as executable claims; the pyramid
- [ ] 10.2 Unit vs integration (and when integration wins)
- [ ] 10.3 Mock the network boundary, not the database
- [ ] 10.4 AAA + self-contained fixtures
- [ ] 10.5 Test invariants, not just the happy path; fail loud
- [ ] 10.6 Quality gates in CI
- [ ] ✅ Module 10 mastery checklist
- [ ] 🛠️ Mini-project: test suite for the wallet

## Module 11 — [DevOps, CI/CD & deployment](11-devops-cicd.md)
- [ ] 11.1 CI vs CD
- [ ] 11.2 Dev/prod parity via local stacks
- [ ] 11.3 Hosting model must match rendering model
- [ ] 11.4 Config & secrets per environment
- [ ] 11.5 Migrations in the pipeline
- [ ] 11.6 Deploy strategies & rollback
- [ ] ✅ Module 11 mastery checklist
- [ ] 🛠️ Mini-project: CI workflow + local stack + runbook

## Module 12 — [Frontend architecture & rendering](12-frontend.md)
- [ ] 12.1 Rendering models (CSR / SSR / SSG)
- [ ] 12.2 Hydration & mismatches
- [ ] 12.3 Component composition & shared UI
- [ ] 12.4 Design tokens & theming
- [ ] 12.5 State, data fetching & decoupling
- [ ] 12.6 Error boundaries, responsive, timezones
- [ ] ✅ Module 12 mastery checklist
- [ ] 🛠️ Mini-project: 3-page app (CSR/SSR/SSG + tokens + boundary)

## Module 13 — [SOLID, DRY & clean code](13-design-principles.md)
- [ ] 13.1 Why principles exist (the cost of change)
- [ ] 13.2 SOLID, part 1: S, O, L
- [ ] 13.3 SOLID, part 2: I, D
- [ ] 13.4 DRY (and false DRY)
- [ ] 13.5 Clean-code heuristics
- [ ] 13.6 The over-abstraction pitfall
- [ ] ✅ Module 13 mastery checklist
- [ ] 🛠️ Mini-project: refactor a messy function step by step

## Module 14 — [Async jobs & scheduling](14-async-jobs.md)
- [ ] 14.1 Why async (slow / deferred / periodic)
- [ ] 14.2 Producer → queue → worker
- [ ] 14.3 Idempotent jobs & synthetic keys
- [ ] 14.4 Secured trigger endpoints
- [ ] 14.5 Batch resilience & observability
- [ ] 14.6 Safety-net design (no double-action)
- [ ] ✅ Module 14 mastery checklist
- [ ] 🛠️ Mini-project: a "daily digest" job

## Module 15 — [Spec-driven dev & governance](15-spec-governance.md)
- [ ] 15.1 Spec → plan → implement
- [ ] 15.2 Definition of Done & scope boundaries
- [ ] 15.3 Architecture Decision Records (ADRs)
- [ ] 15.4 The coding constitution
- [ ] 15.5 Conventional commits & meaningful history
- [ ] 15.6 Review as enforcement
- [ ] ✅ Module 15 mastery checklist
- [ ] 🛠️ Mini-project: spec + plan + ADR + mini-constitution

---

## Module 16 — [Cryptography & key management](16-cryptography-key-management.md)
- [ ] 16.1 Authenticated encryption (AES-256-GCM) & ciphertext framing
- [ ] 16.2 Envelope encryption with a key-encryption-key (KEK)
- [ ] 16.3 Public/private separation & the JWKS document
- [ ] 16.4 Pluggable encryption with a versioned method (planning for KMS)
- [ ] 16.5 Atomic key rotation with zero verification downtime
- [ ] ✅ Module 16 mastery checklist
- [ ] 🛠️ Mini-project: a secret vault (encrypt · JWKS · rotate)

## Module 17 — [SDK & library design](17-sdk-library-design.md)
- [ ] 17.1 A thin public facade over private internals
- [ ] 17.2 Cache an expensive remote resource (JWKS) with a TTL
- [ ] 17.3 Verify JWTs safely: allow-list algorithms, validate every audience
- [ ] 17.4 Map transport errors to typed domain errors
- [ ] 17.5 12-factor config (fail-fast) + a capability handshake
- [ ] 17.6 Fragile couplings: string-matching a dependency, and pinning against it
- [ ] ✅ Module 17 mastery checklist
- [ ] 🛠️ Mini-project: a verification SDK

## Module 18 — [Resilience & failure modes](18-resilience-failure-modes.md)
- [ ] 18.1 Fail-closed vs fail-open
- [ ] 18.2 Compensating transactions (the saga)
- [ ] 18.3 Reconciliation loops (re-sync against the source of truth)
- [ ] 18.4 Resilient batch processing (per-item isolation)
- [ ] 18.5 Compensate-don't-rollback in a sweep + compare-and-set
- [ ] 18.6 Idempotency keys make external calls retry-safe
- [ ] ✅ Module 18 mastery checklist
- [ ] 🛠️ Mini-project: a flaky payout (saga + reconcile)

## Module 19 — [Integration & protocol design](19-integration-protocol-design.md)
- [ ] 19.1 Invert the integration direction to dissolve the hard problem
- [ ] 19.2 The launch token as an OAuth authorization-code analog
- [ ] 19.3 Token-delivery trade-offs and a documented upgrade path
- [ ] 19.4 Place the invariant on the right side of the trust boundary
- [ ] 19.5 System-driven state transitions (API is the primitive)
- [ ] 19.6 Catch contract drift with an independent reference
- [ ] ✅ Module 19 mastery checklist
- [ ] 🛠️ Mini-project: a mini launch protocol

## Module 20 — [Performance & data access](20-performance-data-access.md)
- [ ] 20.1 Kill N+1 queries: batch, merge, parallelize
- [ ] 20.2 Index the hot path; paginate with keyset cursors
- [ ] 20.3 Client-side server-cache (staleTime/gcTime, dependent queries, invalidation)
- [ ] 20.4 HTTP caching for public, slow-changing endpoints
- [ ] 20.5 Reuse connections with lazy-singleton client factories
- [ ] 20.6 Measure, don't guess
- [ ] ✅ Module 20 mastery checklist
- [ ] 🛠️ Mini-project: speed up a slow endpoint

## Module 21 — [Observability & operations](21-observability-operations.md)
- [ ] 21.1 Structured logging with secret redaction
- [ ] 21.2 Correlation IDs (one request, one thread of logs)
- [ ] 21.3 Middleware ordering is a contract
- [ ] 21.4 Monitor unit economics as SLOs
- [ ] 21.5 Runbooks as executable documentation
- [ ] ✅ Module 21 mastery checklist
- [ ] 🛠️ Mini-project: operationalize an API

---

## 🏁 Capstone (optional)
Tie it all together — extend the wallet from Modules 09/10 into a tiny marketplace **with a launch protocol**:
- [ ] An idempotent, locked, atomic metered-charge endpoint (01, 02, 06, 07, 09)
- [ ] A Stripe-style webhook that's signature-verified and amount-checked (08, 09)
- [ ] A renewal cron with a synthetic key + safety-net (14)
- [ ] A real-DB integration suite covering the invariants (10)
- [ ] A CI gate + a migration applied through the pipeline (03, 11)
- [ ] A spec, an ADR, and conventional commits for the whole thing (15)
- [ ] Per-listing signing keys (AES-GCM at rest) + a JWKS endpoint with rotation (16)
- [ ] A launch token (short-lived, single-use, signed, audience-scoped) verified by a small SDK that fails closed (17, 18, 19)
- [ ] N+1-free reads, keyset pagination, and structured logs with a request id (20, 21)
