# Module 03 — Database Design & Schema Migrations

> **Core question:** How do you evolve a schema without downtime or data loss?
> **Reference doc:** [../03-database-design-and-schema-migrations.md](../03-database-design-and-schema-migrations.md)
> **Prerequisites:** [01 — Idempotency](01-idempotency.md), [02 — Concurrency](02-concurrency.md).

## Why this module
The schema in production already holds real data, and live code expects a specific shape. You can't
just "edit the schema." This module teaches you to evolve it in a controlled, ordered, reversible
way that every environment applies identically — often *while the system is live*.

### What you'll be able to do
- Treat migrations like commits: versioned, ordered, immutable.
- Tell additive from destructive changes and apply expand/contract for zero downtime.
- Adopt migrations on a database that predates your tooling (baseline).
- Design append-only ledgers and avoid the classic ordering/`IF NOT EXISTS`/float-money traps.

---

## Lesson 3.1 — Schema as a contract, migration as history

**Goal:** frame migrations correctly.

### Concept
A **schema** is the contract for how data is shaped. A **migration** is a versioned, ordered change
to that contract — a script that moves the schema (and sometimes the data) from one known state to
the next. **Migrations are to a database what commits are to code:** a replayable history any
environment can fast-forward through to reach an identical state.

### ✅ Lesson 3.1 checklist
- [ ] 📖 Read & understood — I can explain migrations as a replayable, ordered history.
- [ ] 💻 Applied — I described the current migration mechanism of a project I work on.
- [ ] 🔍 Found in Mythos — every schema change ships a numbered `docs/migrations/NNNN-*.md`.

---

## Lesson 3.2 — Forward-only, ordered, immutable

**Goal:** learn the cardinal rule of migrations.

### Concept
- Each migration has a **monotonic id** (often a timestamp).
- **Once a migration is applied to a shared environment, you NEVER edit it** — you write a *new*
  one. Editing applied history makes environments diverge. This is the cardinal sin.
- Ship **up/down** (apply/revert) so you can roll back.

```sql
-- 20260601120000_add_referral_code.up.sql
ALTER TABLE users ADD COLUMN referral_code text;

-- 20260601120000_add_referral_code.down.sql
ALTER TABLE users DROP COLUMN referral_code;
```

### ❌ Bad → ✅ Good
- ❌ Notice a typo in a migration that already ran in staging → edit the file and re-run. *Staging
  and prod now disagree about history.*
- ✅ Write a **new** migration that fixes the typo forward.

### ✅ Lesson 3.2 checklist
- [ ] 📖 Read & understood — I can state "never edit an applied migration; write a new one."
- [ ] 💻 Applied — I wrote a matching up/down pair.
- [ ] 🔍 Found in Mythos — `BE#88` review *blocked merge* for a missing migration doc; docs are immutable records.

---

## Lesson 3.3 — Additive vs destructive & expand/contract

**Goal:** change a schema with zero downtime.

### Concept
- **Additive** (new table, *nullable* column, new index): safe in one shot — old code ignores what
  it doesn't know about.
- **Destructive** (drop / rename / retype a column): needs a **two-phase expand/contract** deploy,
  because old and new code run *simultaneously* during a rollout.

**Renaming `email` → `email_address` without downtime:**
```
Phase 1 (EXPAND):  add new column email_address; write to BOTH; backfill old rows.
                   deploy code that reads new, falls back to old. (both shapes tolerated)
Phase 2 (CONTRACT): once all code writes/reads the new column, drop `email`.
```
During the overlap, **code (and schema) must tolerate both worlds** — the same principle as
incremental migration (Module [05](05-incremental-migration.md)).

### Check yourself
1. Why can't you rename a column in a single migration with zero downtime? (Because during the deploy,
   old and new app versions run at once; one of them will reference a column that doesn't exist.)
2. Which of these is safe in one shot: add nullable column / drop column / add index? (Add nullable
   column and add index are additive; drop is destructive.)

### ✅ Lesson 3.3 checklist
- [ ] 📖 Read & understood — I can lay out expand/contract for a rename.
- [ ] 💻 Applied — I classified 3 schema changes as additive or destructive.
- [ ] 🔍 Found in Mythos — constitution + `BE#82`: additive changes ship without phased rollout; `BE#82` made a migration re-runnable with `IF NOT EXISTS`.

---

## Lesson 3.4 — Baseline migrations on a live DB

**Goal:** adopt migration tooling on a database that already exists.

### Concept
When a database predates your migration tooling, you **capture the current state as migration #1**
and mark it *already applied* rather than re-running it (which would fail — the tables exist).

```bash
# capture the existing 20 tables/ENUMs/FKs/indexes as the baseline migration, then:
supabase migration repair --status applied <baseline_id>   # "this is already in prod"
# do NOT `supabase db push` — the schema already exists
```

### ✅ Lesson 3.4 checklist
- [ ] 📖 Read & understood — I can explain why you mark a baseline "applied" instead of running it.
- [ ] 💻 Applied — n/a unless you have a legacy DB; describe the steps you'd take.
- [ ] 🔍 Found in Mythos — `BE#44` captured all 20 existing tables as the baseline + `migration repair --status applied`.

---

## Lesson 3.5 — Modeling: normalization, append-only ledgers, indexing

**Goal:** make good data-shape decisions.

### Concept
- **Normalize vs denormalize.** Normalize to remove duplication and update anomalies; denormalize
  *deliberately* for read performance, accepting the consistency cost.
- **Append-only / event-sourced tables.** Some data should never be updated in place — you only
  insert. A **financial ledger** is the archetype: never edit a posted entry; post a *compensating*
  one. This gives a perfect audit trail and natural idempotency hooks.
- **Index the hot paths.** Index foreign keys and any column you filter/join on frequently. Every
  index speeds reads but **slows writes and costs storage** — and **encodes a business rule**:
```sql
-- ❌ WRONG rule: rejects a valid charge that legitimately splits across two buckets
CREATE UNIQUE INDEX uq ON wallet_ledger (reference_id, kind);
-- ✅ RIGHT rule: a charge may produce two rows (same kind) in different buckets
CREATE UNIQUE INDEX uq ON wallet_ledger (reference_id, kind, bucket);
```
- **Store money as integer minor units (cents/credits), never floats** (Module [09](09-payments.md)).
- **Declare foreign keys explicitly** — app code forgets referential integrity; the database doesn't.

### ✅ Lesson 3.5 checklist
- [ ] 📖 Read & understood — I can explain append-only ledgers and "an index encodes a business rule."
- [ ] 💻 Applied — I designed an append-only table for some event in my domain.
- [ ] 🔍 Found in Mythos — `BE#70` append-only `wallet_ledger`; `BE#88` fixed the unique index `(reference_id, kind)` → `(…, kind, bucket)`.

---

## Lesson 3.6 — The pitfalls that actually bite

**Goal:** internalize the failure modes.

### Concept
- **Editing an applied migration** → environments diverge (Lesson 3.2).
- **`CREATE … IF NOT EXISTS` masking a real correction.** If the object already exists in a *wrong*
  state, the corrected migration silently no-ops and the wrong state survives. Idempotent DDL is
  great for re-runs but dangerous when it hides a needed change.
- **Migration ordering bugs.** A later migration that depends on an earlier one breaks if timestamps
  imply the wrong order.
- **PL/pgSQL column ambiguity (`42702`).** In a function whose `RETURNS TABLE(...)` output variable
  shares a name with a real column, references become ambiguous. **Fix:** qualify with table aliases.
- **Float money.** Rounding drift, un-auditable totals.

> **The `BE#87` disaster — three pitfalls at once.** (1) migration `20260514` ran before MYT-106's
> `20260528`, creating `launch_sessions` with columns the spec had eliminated; (2) MYT-106 used
> `CREATE TABLE IF NOT EXISTS`, so the corrected schema silently no-op'd and stale `NOT NULL`
> columns survived → `23502` violations on every launch; (3) two functions had `42702` ambiguous
> columns. Read that PR to feel *why* ordering and `IF NOT EXISTS` deserve respect.

### ✅ Lesson 3.6 checklist
- [ ] 📖 Read & understood — I can explain how `IF NOT EXISTS` can mask a needed correction.
- [ ] 💻 Applied — I reviewed a migration for ordering/`IF NOT EXISTS`/float-money risk.
- [ ] 🔍 Found in Mythos — `BE#87` (the ordering disaster) and its `42702` alias fix.

---

## 🎯 Module 03 mastery checklist
- [ ] I can explain why migrations must be ordered, forward-only, and immutable once applied.
- [ ] I can perform an expand/contract rename with zero downtime.
- [ ] I can adopt migrations on a pre-existing database via a baseline + repair.
- [ ] I can design an append-only ledger and justify an index as a business rule.
- [ ] I can spot ordering, `IF NOT EXISTS`-masking, `42702`, and float-money hazards in review.

## 🛠️ Mini-project
1. Create a small schema with a migration tool (Supabase CLI, Prisma, Knex, Flyway — your choice).
2. Ship three migrations: add a table (additive), add a nullable column + index (additive), then
   **rename** a column using expand/contract across two migrations.
3. Deliberately try to "fix" an applied migration by editing it; observe the tool's drift detection.
4. Add an append-only `events` table and write a compensating-entry function instead of an `UPDATE`.

## 🔗 Mythos PR map
| PR | What it demonstrates |
|----|----------------------|
| `BE#44` | Baseline migration on a live DB + `migration repair --status applied`. |
| `BE#70` | Append-only `wallet_ledger` + dual-bucket wallet. |
| `BE#87` | The ordering / `IF NOT EXISTS` / `42702` triple-disaster. |
| `BE#88` | An index encodes a rule: `(reference_id, kind)` → `(…, kind, bucket)`. |
| `BE#75` | Migration shipped as a manual doc operators run per-environment. |

## See also
- [02 — Concurrency](02-concurrency.md) — ordering as a race in time.
- [09 — Payments & billing](09-payments.md) — the ledger model.
- [11 — DevOps, CI/CD](11-devops-cicd.md) — how migrations reach prod.
