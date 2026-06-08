# 11 — DevOps, CI/CD & Deployment

## The idea

DevOps is the practice of making the path from "code on a laptop" to "running in production"
**fast, repeatable, and safe**. Its two pillars:

- **CI (Continuous Integration)** — every change is automatically built and tested on merge, so
  integration problems surface in minutes, not at release time.
- **CD (Continuous Delivery/Deployment)** — every change that passes CI can be (or is
  automatically) released through an automated pipeline, so deploys are routine and boring instead
  of risky and rare.

Underneath both: **environment parity** (dev looks like prod), **infrastructure as code**, and
**reproducible builds**.

## Why it exists

Manual, snowflake deployments are slow and error-prone: "works on my machine," forgotten steps,
config drift between environments, scary big-bang releases nobody wants to do. Automating the
pipeline makes releasing a non-event, shrinks the blast radius of each change, and gives you a
fast feedback loop. The closer dev/staging/prod are, the fewer "only breaks in prod" surprises.

## Patterns & trade-offs

- **Dev/prod parity via local stacks.** Run the real services locally (containers or a CLI that
  mirrors the managed service) so local behavior matches prod. The closer the parity, the fewer
  environment-specific bugs.
- **Containerization.** Package the app with its runtime into an image so it runs identically
  anywhere. Trade-off: image build/registry overhead and orchestration complexity.
- **Orchestrated, containerized hosting vs static/CDN hosting.** A static-site + CDN model
  (S3/CloudFront) is cheap and simple for purely client-rendered SPAs; a containerized model
  (ECS/Kubernetes) is needed once you have server-side rendering, server routes, or background
  processes. Migrating from one to the other is itself a project.
- **Pipeline gates.** CI runs build + type-check + lint + tests on every PR; only green merges
  (see [doc 10](10-testing-strategy-and-quality-gates.md)).
- **Config & secrets per environment.** Behavior comes from environment variables, not hardcoded
  values; secrets are injected at deploy time, never committed (see
  [doc 08](08-security-engineering.md)).
- **Migrations in the pipeline.** Schema changes are applied through the same controlled,
  versioned mechanism as code, in order, with a baseline for pre-existing schema (see
  [doc 03](03-database-design-and-schema-migrations.md)).
- **Deploy strategies.** Rolling / blue-green / canary trade rollout speed against the ability to
  cut over and roll back instantly.

## Pitfalls

- **Config drift** between environments → "works in staging, breaks in prod."
- **Snowflake servers** configured by hand → unreproducible and undocumented.
- **Choosing the wrong hosting model** for the rendering model (static CDN for an app that now
  needs SSR/server routes).
- **Secrets baked into images or committed** to the repo.
- **Manual migration steps** outside the pipeline → forgotten or run out of order.
- **No fast rollback** → a bad deploy means downtime instead of a one-click revert.

## Seen in Mythos

- **Local stack, evolved for parity — `BE#67` → `BE#76`.** First a **Docker Compose** local
  development stack with Supabase-compatible services (`BE#67`), then a deliberate migration *to
  the Supabase CLI* for local dev (`BE#76`). The goal both times: make local mirror the managed
  Postgres so behavior matches production. The CLI also became the migration mechanism (`make
  start && make migrate && npm run dev` appears in `BE#85`/`BE#86` test instructions).

- **Migrations as pipeline artifacts — `BE#44`.** The Supabase CLI baseline migration + the
  `migration repair --status applied` workflow is infrastructure-as-code for the schema: every
  environment fast-forwards through the same ordered migration history.

- **Hosting model migration to match rendering — `FM#49`.** `ci(deploy): Migrate from
  S3/CloudFront to ECS containerized deployment`, landing as part of the Next.js migration. This is
  the canonical "the rendering model changed (SPA → SSR/server routes), so the hosting model must
  change too (static CDN → containerized orchestration)." It's filed under `ci(deploy)` precisely
  because it's a pipeline/infra change, not a feature.

- **Release as an explicit, tracked step — `BE#13`/`BE#14`/`BE#22`/`BE#27`, `FM#28`/`FM#30`/
  `FM#37`/`FM#38`, `BE#52`/`BE#73`.** A steady cadence of `chore(release)`/`chore(deploy)` PRs
  ("Deploy Backend to live," "Fix build error at FE," "Release to production"). Deploys are
  first-class, reviewed events with their own PRs — and several (`FM#30`, `FM#37`,
  `BE#19` "changes to be live") are *build-error fixes surfaced at deploy time*, the exact feedback
  CI exists to move earlier.

- **Config over hardcoding — `FM#43`.** The Next.js migration replaced `import.meta.env` (Vite)
  with `process.env`-based API config (`src/shared/config/api.ts`) — environment-driven behavior, a
  prerequisite for clean multi-environment deploys.

## See also

- [03 — Database design & migrations](03-database-design-and-schema-migrations.md)
- [10 — Testing strategy & quality gates](10-testing-strategy-and-quality-gates.md)
- [05 — Incremental migration](05-incremental-migration-strangler-fig.md) (the Next.js + ECS move)
