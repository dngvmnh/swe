# 03 — Database Design & Schema Migrations

## The idea

A database **schema** is the contract for how data is shaped. A **migration** is a versioned,
ordered change to that contract — a script that moves the schema (and sometimes the data) from
one known state to the next. Migrations are to a database what commits are to code: a replayable
history that any environment can fast-forward through to reach an identical state.

## Why it exists

The schema in production already holds real data, and the application running against it expects
a specific shape. You cannot just "edit the schema" — you must evolve it in a controlled,
reversible, ordered way that every environment (local, staging, prod) applies identically, often
*while the system is live*. Migrations make schema change auditable, repeatable, and safe to roll
back.

## Patterns & trade-offs

- **Forward-only, ordered, immutable migrations.** Each migration has a monotonic id (often a
  timestamp). Once a migration is applied to a shared environment, you **never edit it** — you
  write a new one. Editing applied history is the cardinal sin; environments diverge.
- **Up / down (apply / revert).** A complete migration ships both the change and its inverse so
  you can roll back.
- **Additive vs destructive changes.**
  - *Additive* (new table, nullable column, new index): safe to deploy in one shot — old code
    ignores what it doesn't know about.
  - *Destructive* (drop / rename / retype a column): requires a **two-phase / expand-contract**
    deploy. Phase 1 ships code that tolerates both old and new shapes; Phase 2 applies the DDL.
    This is how you change schema with zero downtime.
- **Baseline migration.** When a database predates your migration tooling, you capture the
  current state as migration #1 and mark it "already applied" rather than re-running it.
- **Normalization vs denormalization.** Normalize to remove duplication and update anomalies;
  denormalize deliberately for read performance, accepting the consistency cost.
- **Append-only / event-sourced tables.** Some data should never be updated in place — you only
  insert. A financial ledger is the archetype: you never edit a posted entry; you post a
  compensating one. This gives you a perfect audit trail and natural idempotency hooks.
- **Index the hot paths.** Index foreign keys and any column you filter or join on frequently;
  every index speeds reads but slows writes and costs storage.

## Pitfalls

- **Editing an applied migration.** Breaks every environment that already ran the old version.
- **`CREATE ... IF NOT EXISTS` hiding a needed change.** If the object already exists in a wrong
  state, the "corrected" migration silently no-ops and the wrong state survives. (Idempotent DDL
  is good for re-runs but dangerous when it masks a real correction.)
- **Migration ordering bugs.** A later-numbered migration that *depends on* an earlier one breaks
  if the timestamps imply the wrong order, or if an earlier migration was authored after a later
  one but with an earlier timestamp.
- **Relying on the application for referential integrity.** Declare foreign keys explicitly; app
  code forgets, the database doesn't.
- **PL/pgSQL column ambiguity.** In a function whose `RETURNS TABLE(...)` output variable shares
  a name with a real column, references become ambiguous (`42702`). Qualify with table aliases.
- **Float money.** Store money as integer minor units (cents/credits), never floats.

## Seen in Mythos

- **Baseline migration + `migration repair` — `BE#44`.** The Mythos DB predated the Supabase CLI
  migration system, so this PR captured all 20 existing production tables, ENUMs, FK constraints,
  and indexes as the baseline migration, with explicit instructions **not** to run
  `supabase db push` (the schema already exists) but to `supabase migration repair --status
  applied <id>`. This is the canonical "adopt migrations on a live DB" move.

- **Append-only ledger + dual-bucket wallet — `BE#70`.** The wallet model is two balance buckets
  (`subscription`, `topup`) plus an **append-only** `wallet_ledger` that records every credit and
  debit. Nothing in the ledger is mutated; the running balances are derived/maintained alongside
  immutable entries. This is event-ledger design applied to money: full audit trail, and the
  ledger's unique columns double as idempotency keys (see doc 01).

- **The migration-ordering disaster — `BE#87`.** A masterclass in three migration pitfalls at
  once: (1) migration `20260514` ran before MYT-106's `20260528`, creating `launch_sessions` with
  columns the spec had eliminated; (2) MYT-106 used `CREATE TABLE IF NOT EXISTS`, so the corrected
  schema silently no-op'd and the stale `NOT NULL` columns (`entitlement_id`, `creator_id`,
  `credits`) survived, causing `23502` constraint violations on every launch; (3) two PL/pgSQL
  functions had `42702` ambiguous-column errors because `RETURNS TABLE` output names shadowed real
  columns — fixed by table-qualifying with aliases. Read this PR to internalize *why* ordering and
  `IF NOT EXISTS` deserve respect.

- **Migration documentation as governance — `BE#75`, `BE#44`, and the constitution.** Every schema
  change ships a numbered doc in `docs/migrations/NNNN-*.md` containing description, affected
  tables, SQL up/down, and prod-applied dates. `BE#75` even shipped the migration as a *manual*
  doc that operators must run per-environment before deploying. (The same rule is why `BE#88`'s
  review demanded a missing migration doc be added.)

- **Index correctness for a real invariant — `BE#88`.** The original unique index `(reference_id,
  kind)` wrongly rejected a valid charge that split across both buckets (two ledger rows, same
  kind). It was replaced with `(reference_id, kind, bucket)` — a reminder that an index encodes a
  business rule, and the wrong index encodes the wrong rule.

- **Additive-change discipline — constitution + `BE#82`.** New nullable columns / tables / indexes
  ship without phased rollout; `BE#82` added `IF NOT EXISTS` to a signing-keys migration to keep
  it safely re-runnable.

## See also

- [01 — Idempotency](01-idempotency-and-exactly-once-semantics.md)
- [02 — Concurrency & locking](02-concurrency-locking-and-race-conditions.md)
- [11 — DevOps, CI/CD & deployment](11-devops-cicd-and-deployment.md) (how migrations reach prod)
