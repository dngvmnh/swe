# Module 11 — DevOps, CI/CD & Deployment

> **Core question:** How does code get from a laptop to production?
> **Reference doc:** [../11-devops-cicd-and-deployment.md](../11-devops-cicd-and-deployment.md)
> **Prerequisites:** [10 — Testing & gates](10-testing.md), [03 — Migrations](03-database-migrations.md).

## Why this module
Manual, snowflake deployments are slow and error-prone: "works on my machine," forgotten steps,
config drift, scary big-bang releases. This module teaches you to make releasing **fast, repeatable,
and boring**.

### What you'll be able to do
- Explain CI and CD and what each pillar buys you.
- Achieve dev/prod parity with local stacks and config-as-environment.
- Pick a hosting model that matches your rendering model.
- Put migrations in the pipeline and choose a deploy/rollback strategy.

---

## Lesson 11.1 — CI vs CD

**Goal:** separate the two pillars.

### Concept
- **CI (Continuous Integration):** every change is automatically built and tested on merge, so
  integration problems surface in **minutes, not at release time.**
- **CD (Continuous Delivery/Deployment):** every change that passes CI **can be (or is) released**
  through an automated pipeline, so deploys are **routine and boring** instead of risky and rare.

Underneath both: **environment parity**, **infrastructure as code**, **reproducible builds**.

### ✅ Lesson 11.1 checklist
- [ ] 📖 Read & understood — I can distinguish CI from CD and name what each buys.
- [ ] 💻 Applied — I described the CI/CD pipeline (or lack of one) in a project I know.
- [ ] 🔍 Found in Mythos — `chore(release)`/`ci(deploy)` PRs are first-class, reviewed events.

---

## Lesson 11.2 — Dev/prod parity via local stacks

**Goal:** make local behave like prod.

### Concept
Run the **real services locally** (containers or a CLI that mirrors the managed service) so local
behavior matches prod. **The closer the parity, the fewer "only breaks in prod" surprises.**

```bash
# a local stack that mirrors managed Postgres, so behavior matches production
make start && make migrate && npm run dev
```

### ✅ Lesson 11.2 checklist
- [ ] 📖 Read & understood — I can explain dev/prod parity and why it reduces surprises.
- [ ] 💻 Applied — I ran a local stack that mirrors a managed dependency.
- [ ] 🔍 Found in Mythos — `BE#67` Docker Compose local stack → `BE#76` migrated local dev to the Supabase CLI.

---

## Lesson 11.3 — Hosting model must match rendering model

**Goal:** choose infrastructure for how the app renders.

### Concept
- **Static site + CDN** (S3/CloudFront): cheap and simple for **purely client-rendered SPAs**.
- **Containerized orchestration** (ECS/Kubernetes): needed once you have **SSR, server routes, or
  background processes.**
- **Containerization** packages the app + runtime into an image so it runs identically anywhere
  (trade-off: build/registry/orchestration overhead).

> **The canonical move:** the rendering model changed (SPA → SSR/server routes), so the hosting model
> must change too (static CDN → containerized). Migrating between them is **its own project.**

### Check yourself
1. You add server-side rendering and server API routes to a CDN-hosted SPA. What has to change?
   (The hosting model — you now need a running server / containers, not just static files.)

### ✅ Lesson 11.3 checklist
- [ ] 📖 Read & understood — I can match a rendering model to a hosting model.
- [ ] 💻 Applied — I identified which hosting model an app I know should use and why.
- [ ] 🔍 Found in Mythos — `FM#49` `ci(deploy): Migrate from S3/CloudFront to ECS` alongside the Next.js (SSR) migration.

---

## Lesson 11.4 — Config & secrets per environment

**Goal:** drive behavior from the environment, not hardcoded values.

### Concept
Behavior comes from **environment variables**, not hardcoded values; **secrets are injected at
deploy time, never committed** (Module [08](08-security.md)).

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — build-tool-specific, hardcoded base URL
const API = "https://api.prod.example.com";
const API = import.meta.env.VITE_API;     // ties you to one build tool
// ✅ GOOD — environment-driven, portable across environments
const API = process.env.API_BASE_URL;
```

### ✅ Lesson 11.4 checklist
- [ ] 📖 Read & understood — I can explain config-as-environment and deploy-time secret injection.
- [ ] 💻 Applied — I replaced a hardcoded value with an env-driven config.
- [ ] 🔍 Found in Mythos — `FM#43` replaced Vite `import.meta.env` with `process.env`-based API config.

---

## Lesson 11.5 — Migrations in the pipeline

**Goal:** apply schema changes the same controlled way as code.

### Concept
Schema changes go through the **same controlled, versioned mechanism** as code — in order, with a
**baseline** for pre-existing schema (Module [03](03-database-migrations.md)). Every environment
fast-forwards through the same ordered history. **Don't run migrations by hand outside the pipeline**
— they get forgotten or run out of order.

### ✅ Lesson 11.5 checklist
- [ ] 📖 Read & understood — I can explain migrations as ordered pipeline artifacts.
- [ ] 💻 Applied — I described how migrations run in a deploy I know.
- [ ] 🔍 Found in Mythos — `BE#44` baseline + `migration repair --status applied` = infrastructure-as-code for the schema.

---

## Lesson 11.6 — Deploy strategies & rollback

**Goal:** trade rollout speed against instant recovery.

### Concept
| Strategy | How | Trade-off |
|----------|-----|-----------|
| Rolling | replace instances gradually | simple; mixed versions briefly run together |
| Blue-green | stand up new env, switch traffic | instant cutover & rollback; double the infra briefly |
| Canary | send a small % of traffic to the new version | safest; most orchestration |

**Always have a fast rollback** — a bad deploy should be a one-click revert, not downtime.

### ✅ Lesson 11.6 checklist
- [ ] 📖 Read & understood — I can compare rolling/blue-green/canary and explain fast rollback.
- [ ] 💻 Applied — I described a rollback plan for a deploy.
- [ ] 🔍 Found in Mythos — release PRs (`BE#13/14/22/27`, `FM#28/30/37/38`) treat deploys as tracked, revertible events.

---

## 🎯 Module 11 mastery checklist
- [ ] I can explain CI vs CD and what each buys.
- [ ] I can set up dev/prod parity with a local stack mirroring the managed service.
- [ ] I can match a hosting model to a rendering model.
- [ ] I can drive config from the environment and inject secrets at deploy time.
- [ ] I can run migrations as ordered pipeline artifacts.
- [ ] I can compare deploy strategies and ensure fast rollback.

## 🛠️ Mini-project
1. Take a small app and add a CI workflow (GitHub Actions) that builds + type-checks + lints + tests
   on every PR and **blocks merge** on failure.
2. Add a local stack (Docker Compose or a managed-service CLI) so `make start && migrate && dev`
   reproduces prod-like behavior.
3. Move every hardcoded URL/secret into env config with an `.env.example`.
4. Write a one-paragraph deploy + rollback runbook (which strategy, how to revert).

## 🔗 Mythos PR map
| PR | What it demonstrates |
|----|----------------------|
| `BE#67` → `BE#76` | Local stack (Docker Compose) → Supabase CLI for dev/prod parity. |
| `FM#49` | Hosting migration S3/CloudFront → ECS to match SSR. |
| `BE#44` | Migrations as pipeline artifacts (baseline + repair). |
| `FM#43` | Config over hardcoding (`process.env`). |
| `BE#13/14/22/27`, `FM#28/30/37/38` | Releases/deploys as first-class tracked events. |

## See also
- [03 — Database design & migrations](03-database-migrations.md)
- [10 — Testing strategy & quality gates](10-testing.md)
- [05 — Incremental migration](05-incremental-migration.md) — the Next.js + ECS move.
