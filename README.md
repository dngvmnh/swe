# Software Engineering Concepts & Architecture — A Field Guide

A knowledge base of the software-engineering concepts and architecture designs — grounded in **real pull requests**.

The goal is to learn the *idea* (which is universal and outlives any framework) and then see
*exactly how it shows up* in production code you have access to. Every document follows the
same shape:

1. **The idea** — the universal concept, stated plainly.
2. **Why it exists** — the problem it was invented to solve.
3. **Patterns & trade-offs** — how practitioners apply it, and what you give up.
4. **Pitfalls** — the ways it goes wrong.
5. **Seen in Mythos** — concrete PR case studies (`repo#NN`) that demonstrate it.

> PR references use the form `BE#88` (backend), `FE#5` (frontend), `FM#43` (frontend-main).

---

## The documents

| # | Document | Core question it answers |
|---|----------|--------------------------|
| 01 | [Idempotency & exactly-once semantics](01-idempotency-and-exactly-once-semantics.md) | How do you make an operation safe to retry? |
| 02 | [Concurrency, locking & race conditions](02-concurrency-locking-and-race-conditions.md) | What breaks when two requests arrive at once? |
| 03 | [Database design & schema migrations](03-database-design-and-schema-migrations.md) | How do you evolve a schema without downtime or data loss? |
| 04 | [Authentication & authorization](04-authentication-and-authorization.md) | How do you know *who* is calling and *what* they may do? |
| 05 | [Incremental migration & the Strangler Fig](05-incremental-migration-strangler-fig.md) | How do you replace a system while it stays live? |
| 06 | [Error handling & typed errors](06-error-handling-and-typed-errors.md) | How do failures travel through a system cleanly? |
| 07 | [API design & layered architecture](07-api-design-and-layered-architecture.md) | How do you structure a server so it stays changeable? |
| 08 | [Security engineering](08-security-engineering.md) | How do you keep attackers and accidents out? |
| 09 | [Payments & billing architecture](09-payments-and-billing-architecture.md) | How do you move money correctly? |
| 10 | [Testing strategy & quality gates](10-testing-strategy-and-quality-gates.md) | How do you trust a change before shipping it? |
| 11 | [DevOps, CI/CD & deployment](11-devops-cicd-and-deployment.md) | How does code get from a laptop to production? |
| 12 | [Frontend architecture & rendering](12-frontend-architecture-and-rendering.md) | How do you build a UI that scales and renders fast? |
| 13 | [Design principles: SOLID, DRY & clean code](13-design-principles-solid-dry.md) | What makes code easy to change six months later? |
| 14 | [Async jobs & scheduling](14-async-jobs-and-scheduling.md) | How do you do work outside the request/response cycle? |
| 15 | [Spec-driven development & governance](15-spec-driven-development-and-governance.md) | How does a team decide and remember *how* it builds? |

---

## How to read this

If you are learning, read in order — the early documents (idempotency, concurrency,
databases) are the foundation that the later ones (payments, testing) build on.

If you are looking something up, jump straight to the relevant document and skim the
**Seen in Mythos** sections; the PR numbers let you open the real diff on GitHub.

## The Mythos system in one paragraph

Mythos is a **two-sided marketplace with a unified token economy**: users buy credits once
and spend them across any product; creators publish "web apps" that consumers launch and get
metered for; 85% of each charge flows to the creator. The backend is Node/Express on Supabase
(Postgres) with Stripe for money and AWS for infrastructure. The frontend migrated from a
Vite/React SPA to Next.js. Most of the engineering difficulty — and therefore most of these
documents — is about **correctness under failure and concurrency**, especially where money is
involved.
