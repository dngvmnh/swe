# 05 — Incremental Migration & the Strangler Fig

## The idea

When you need to replace a language, framework, or whole system, the instinct is the **big-bang
rewrite**: stop, rebuild everything, switch over. It almost always fails — it's risky, it
freezes feature work for months, and the new thing is never quite at parity on day one.

The proven alternative is **incremental migration**: the old and new systems run *side by side*,
and you move functionality across one slice at a time until the old system is fully "strangled."
The name comes from the **Strangler Fig** — a vine that grows around a host tree and gradually
replaces it, with the tree still standing the whole time.

## Why it exists

A running system has value and users *today*. A migration that takes the system offline, or that
must be merged in one enormous untestable PR, puts that value at risk for an uncertain payoff.
Incremental migration keeps the system shippable at every step, shrinks each change to something
reviewable, and lets you stop or course-correct partway.

## Patterns & trade-offs

- **Run old and new in parallel.** Install the new framework *alongside* the old; route or
  compile only the migrated slices through it. The old code stays as a working reference.
- **Strangler facade.** A routing layer sends some requests to the new implementation and the
  rest to the old, so the switch is invisible to callers and can move slice by slice.
- **Branch-per-effort + umbrella branch.** Open a long-lived integration branch for the
  migration; merge small, reviewed slices into it; keep the umbrella PR marked "DON'T MERGE" to
  `main` until parity is reached.
- **Parallel fan-out (workers).** Scaffold a typed/structural *foundation* first, then split the
  remaining work into many independent domain slices that different people (or sessions) execute
  in parallel against that foundation.
- **Foundation-first sequencing.** Shared types, adapters, and config must land before the
  slices that depend on them, or every slice re-invents them and they conflict.
- **Tolerate both worlds.** During the overlap, code (and schema — see expand/contract in
  [doc 03](03-database-design-and-schema-migrations.md)) must accept old and new shapes.

Trade-off: incremental migration is *slower in total* and you live with two systems for a while
(extra cognitive load, glue code). You buy safety and continuous shippability with that cost.

## Pitfalls

- **No foundation.** Starting slices before shared scaffolding exists → merge conflicts and
  divergent re-implementations.
- **The umbrella branch merged early.** Shipping a half-migrated system to `main`.
- **Behavior drift during a "pure refactor."** A migration slice that *also* changes behavior is
  no longer a safe, reviewable refactor — keep them byte-identical or flag the change loudly.
- **The reference copy rotting.** If the old tree keeps changing while you migrate, you chase a
  moving target. Freeze it or migrate fast.

## Seen in Mythos

Mythos ran **two** large incremental migrations, both textbook executions.

### JavaScript → TypeScript (backend)

- **The umbrella — `BE#29` (`refactor(ts-migration)!`, base `main`, marked DON'T MERGE).** A
  long-lived `r/migrate-to-typescript` branch is the integration target; it is deliberately *not*
  merged to `main` until parity.
- **Foundation first — `BE#35` ("Foundation scaffolding for strict-typing fan-out").** Explicitly
  "Phase 0… the typed foundation that **12 parallel domain workers** branch from." It lands shared
  pieces: `InternalError`, tightened `Request.user`/`Request.id` types, typed auth middleware,
  typed Supabase client, an `EmailTemplate` type, and **empty DTO scaffolds** for 13 domains so
  each worker has a place to fill in.
- **The parallel fan-out — `BE#31`–`BE#43`, `BE#45`, `BE#47`.** Thirteen near-identical PRs, one
  per domain: `reset-password`, `auth` ("Worker 2"), `waitlist`, `profile`, `bundle`, `creator`,
  `cron-utils`, `dashboard`, `review`, `payouts-wallet`, `listing`, `purchases`, `schedule`. Each
  is "typed contracts + remove `@ts-nocheck`." `BE#35` even notes a behavior must stay
  "byte-identical on purpose" — refactor discipline.

### React/Vite SPA → Next.js (frontend-main)

- **The umbrella — `FM#43` (`refactor(FE)!: Migrate React to Next.js`, DON'T MERGE).** Installs
  Next.js 16 (App Router) **alongside** the existing Vite/React SPA; the old `src/routes/` tree is
  *excluded from compilation but left intact as a reference* — the Strangler Fig living tree.
- **Foundation first.** `FM#43` wires shared infra before any route moves: providers, MUI SSR
  emotion cache, an auth **event bus** (decoupling the 401 handler from `window.location`),
  `process.env`-based API config, and proxy auth guards. Then it migrates the 14 shared composite
  components from `@tanstack/react-router` to `next/navigation`.
- **Slices on top.** `FM#44` migrates router patterns to Next navigation; `FM#47`/`FM#48` migrate
  auth to NextAuth sessions; `FM#49` migrates deployment to ECS; later feature PRs (`FM#64`–`#71`)
  build new pages directly on the Next foundation.

**The lesson made concrete:** both migrations created an umbrella branch, landed a shared
foundation *first*, kept the old system running as a reference, and then fanned the remaining work
out into many small, independently reviewable slices — never a big-bang rewrite.

## See also

- [03 — Database design & migrations](03-database-design-and-schema-migrations.md) (expand/contract)
- [13 — Design principles](13-design-principles-solid-dry.md) (why small reviewable changes win)
- [15 — Spec-driven development & governance](15-spec-driven-development-and-governance.md)
