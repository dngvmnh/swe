# Module 05 — Incremental Migration & the Strangler Fig

> **Core question:** How do you replace a system while it stays live?
> **Reference doc:** [../05-incremental-migration-strangler-fig.md](../05-incremental-migration-strangler-fig.md)
> **Prerequisites:** helps to know [03 — expand/contract](03-database-migrations.md) and [13 — small reviewable changes](13-design-principles.md).

## Why this module
A running system has value and users *today*. The instinct to "stop and rewrite everything" almost
always fails — it's risky, freezes feature work for months, and is never at parity on day one. This
module teaches the proven alternative: replace a system **one slice at a time** while it keeps running.

### What you'll be able to do
- Explain why big-bang rewrites fail and incremental migration wins.
- Set up the Strangler Fig pattern: old + new running side by side.
- Sequence a migration: umbrella branch → foundation first → parallel slices.
- Avoid the traps that turn a "safe refactor" into a risky one.

---

## Lesson 5.1 — Big-bang vs incremental

**Goal:** know why you almost never rewrite from scratch.

### Concept
The **big-bang rewrite** stops the world, rebuilds everything, switches over. It fails because the
new thing is never at parity on day one, feature work freezes, and the change is one enormous
untestable PR. **Incremental migration** keeps the system shippable at every step, shrinks each
change to something reviewable, and lets you stop or course-correct partway.

**The Strangler Fig** metaphor: a vine grows around a host tree and gradually replaces it — *with
the tree still standing the whole time.*

> **Trade-off:** incremental is *slower in total* and you live with two systems for a while (extra
> glue code, cognitive load). You buy **safety and continuous shippability** with that cost.

### ✅ Lesson 5.1 checklist
- [ ] 📖 Read & understood — I can argue for incremental over big-bang.
- [ ] 💻 Applied — I sketched how I'd slice a rewrite I've considered.
- [ ] 🔍 Found in Mythos — two textbook migrations: JS→TS (backend) and React/Vite→Next.js (frontend).

---

## Lesson 5.2 — Run old & new in parallel (the strangler facade)

**Goal:** make the switch invisible to callers.

### Concept
- **Install the new framework *alongside* the old**; route or compile only migrated slices through
  it. The old code stays as a **working reference**.
- A **strangler facade** is a routing layer that sends some requests to the new implementation and
  the rest to the old — so the cutover moves slice by slice and callers never notice.
- **Tolerate both worlds** during the overlap (same idea as expand/contract in
  [Module 03](03-database-migrations.md)).

```
        ┌────────── facade / router ──────────┐
request │  /new-page   → Next.js (migrated)    │
 ──────►│  /everything → old Vite SPA (ref)    │   ← old tree still live, excluded from build
        └─────────────────────────────────────┘
```

### ✅ Lesson 5.2 checklist
- [ ] 📖 Read & understood — I can explain the strangler facade and "tolerate both worlds".
- [ ] 💻 Applied — I described how a facade would route between old/new in a system I know.
- [ ] 🔍 Found in Mythos — `FM#43` installed Next.js 16 *alongside* the Vite SPA; old `src/routes/` left intact as a reference.

---

## Lesson 5.3 — Sequencing: umbrella branch + foundation first

**Goal:** order the work so slices don't collide.

### Concept
- **Branch-per-effort + umbrella branch.** Open a long-lived integration branch; merge small,
  reviewed slices into it; keep the umbrella PR marked **"DON'T MERGE"** to `main` until parity.
- **Foundation-first sequencing.** Shared types, adapters, and config must land **before** the
  slices that depend on them — otherwise every slice re-invents them and they conflict.

```
r/migrate-to-typescript  (umbrella, base main, "DON'T MERGE")
   └─ Phase 0: foundation  (shared types, typed middleware, DTO scaffolds)  ← lands FIRST
        ├─ slice: auth
        ├─ slice: profile
        ├─ slice: wallet
        └─ … 13 domains, each "typed contracts + remove @ts-nocheck"
```

### Check yourself
1. Why must the foundation land before the slices? (Otherwise 13 workers each invent their own
   `InternalError`/DTOs and merge-conflict.)
2. Why keep the umbrella "DON'T MERGE"? (To avoid shipping a half-migrated system to `main`.)

### ✅ Lesson 5.3 checklist
- [ ] 📖 Read & understood — I can explain umbrella branches and foundation-first ordering.
- [ ] 💻 Applied — I listed the "foundation" pieces a migration I know would need first.
- [ ] 🔍 Found in Mythos — `BE#29` umbrella ("DON'T MERGE"); `BE#35` "Phase 0… the typed foundation 12 parallel workers branch from."

---

## Lesson 5.4 — Parallel fan-out (workers)

**Goal:** scale the work across many independent slices.

### Concept
After the foundation lands, split the remaining work into **many independent domain slices** that
different people (or sessions) execute in parallel against that foundation. Each slice is small and
**independently reviewable**.

> Mythos JS→TS: **13 near-identical PRs** (`BE#31`–`BE#43`, `BE#45`, `BE#47`), one per domain —
> `reset-password`, `auth`, `waitlist`, `profile`, `bundle`, `creator`, `cron-utils`, `dashboard`,
> `review`, `payouts-wallet`, `listing`, `purchases`, `schedule`.

### ✅ Lesson 5.4 checklist
- [ ] 📖 Read & understood — I can describe parallel fan-out after a shared foundation.
- [ ] 💻 Applied — I decomposed a migration into independent slices.
- [ ] 🔍 Found in Mythos — the 13 domain PRs `BE#31`–`BE#47`.

---

## Lesson 5.5 — The pitfalls

**Goal:** keep a "safe refactor" actually safe.

### Concept
- **No foundation** → merge conflicts and divergent re-implementations.
- **Umbrella merged early** → a half-migrated system on `main`.
- **Behavior drift during a "pure refactor."** A slice that *also* changes behavior is no longer a
  safe, reviewable refactor — keep it **byte-identical** or flag the change loudly.
- **The reference copy rotting.** If the old tree keeps changing while you migrate, you chase a
  moving target. Freeze it or migrate fast.

### ✅ Lesson 5.5 checklist
- [ ] 📖 Read & understood — I can name the four pitfalls and their fixes.
- [ ] 💻 Applied — I reviewed a refactor PR for hidden behavior drift.
- [ ] 🔍 Found in Mythos — `BE#35` notes a behavior must stay "byte-identical on purpose."

---

## 🎯 Module 05 mastery checklist
- [ ] I can argue why incremental migration beats a big-bang rewrite.
- [ ] I can set up old + new running in parallel behind a facade.
- [ ] I can sequence an umbrella branch with a foundation-first phase, then parallel slices.
- [ ] I can keep refactor slices behavior-preserving (or flag drift loudly).
- [ ] I can spot "no foundation" and "umbrella merged early" risks.

## 🛠️ Mini-project
Pick a small app and migrate one cross-cutting thing incrementally (e.g. a styling system, a router,
JS→TS on a few modules):
1. Create an umbrella branch marked DON'T MERGE.
2. Land a "foundation" commit (shared config/types) first.
3. Migrate **two** independent slices on top, each its own small PR/commit, each behavior-preserving.
4. Keep the old code compiling/running as a reference the whole time.

## 🔗 Mythos PR map
| PR | What it demonstrates |
|----|----------------------|
| `BE#29` | JS→TS umbrella branch, "DON'T MERGE". |
| `BE#35` | Foundation scaffolding ("Phase 0") for a 12-worker fan-out. |
| `BE#31`–`BE#47` | 13 parallel domain slices. |
| `FM#43` | React/Vite→Next.js umbrella; old SPA kept as reference; shared infra first. |
| `FM#44`/`FM#47`/`FM#48`/`FM#49` | Routing, auth, deployment slices on the new foundation. |

## See also
- [03 — Database design & migrations](03-database-migrations.md) — expand/contract.
- [13 — Design principles](13-design-principles.md) — why small reviewable changes win.
- [15 — Spec-driven development & governance](15-spec-governance.md)
