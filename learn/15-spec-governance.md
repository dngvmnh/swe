# Module 15 — Spec-Driven Development & Governance

> **Core question:** How does a team decide and remember *how* it builds?
> **Reference doc:** [../15-spec-driven-development-and-governance.md](../15-spec-driven-development-and-governance.md)
> **Prerequisites:** helps to have seen [05](05-incremental-migration.md), [10](10-testing.md), [13](13-design-principles.md) in action.

## Why this module
Code alone doesn't capture *intent* or *rationale* — six months later nobody remembers why the wallet
has two buckets or why downgrades don't claw back credits. Without specs, features drift; without
governance, every contributor invents their own conventions and the codebase fragments. This module
is about writing decisions down *before* and *around* the code.

### What you'll be able to do
- Run a spec → plan → implement flow and define "done" objectively.
- Scope a PR with in/out-of-scope boundaries.
- Record decisions as ADRs and codify rules in a constitution.
- Use conventional commits and make review the enforcement mechanism.

---

## Lesson 15.1 — Spec → plan → implement

**Goal:** catch design problems while they're cheap.

### Concept
- **Spec-driven development:** write the specification *before* the code — what the feature does, its
  acceptance criteria, the API contract, the data model, the open questions. The spec is the **source
  of truth** the implementation is checked against.
- Flow: **Spec → Plan → Implement.** Author a spec (scope, acceptance criteria), then a plan (files to
  touch, algorithm, open questions, definition-of-done), then implement against it. Catches design
  problems while they're cheap (a paragraph) instead of expensive (a rewrite).

> Mythos `BE#78` *opened with only a planning doc* (`docs/myt-107-plan.md`) covering files to create,
> algorithm, open questions, and a DoD checklist — implementation followed in later commits.

### ✅ Lesson 15.1 checklist
- [ ] 📖 Read & understood — I can explain spec → plan → implement and why it's cheaper.
- [ ] 💻 Applied — I wrote a one-page spec/plan before coding a feature.
- [ ] 🔍 Found in Mythos — `BE#28`/`FM#42` spec-kits (`specify → clarify → plan → tasks → implement → checklist`); `BE#78` plan-first.

---

## Lesson 15.2 — Definition of Done & scope boundaries

**Goal:** make "done" objective and scope visible.

### Concept
- **Definition of Done (DoD).** An explicit checklist a change must satisfy — acceptance criteria
  met, tests added, docs updated — so "done" is **objective, not a feeling.**
- **In-scope / out-of-scope tables.** State plainly what a PR does *not* do and which follow-ups it
  defers, so reviewers don't expect the world and **scope creep is visible.**

```md
## In scope (this PR)        | ## Out of scope (follow-ups)
- per-listing signing keys    | - KMS-backed key storage (MYT-403)
- JWKS endpoint               | - key rotation policy
```

### ✅ Lesson 15.2 checklist
- [ ] 📖 Read & understood — I can write a DoD checklist and an in/out-of-scope table.
- [ ] 💻 Applied — I added a DoD + scope table to a PR/task.
- [ ] 🔍 Found in Mythos — `BE#75` in/out-of-scope table naming MYT-403; `BE#88` review opens with a DoD table (✅/⚠️/🔄).

---

## Lesson 15.3 — Architecture Decision Records (ADRs)

**Goal:** preserve the *why* behind decisions.

### Concept
An **ADR** is a short, **immutable** doc capturing a significant decision, its context, the options
considered, and the consequences. Future readers learn *why*, not just *what* — and you stop
re-litigating settled debates.

```md
# ADR-0004: Per-listing RSA signing keys
## Context   — creators must verify launch tokens without our secret
## Decision  — generate per-listing RSA-2048 keys; expose public via JWKS
## Options considered — shared HMAC (rejected: secret sharing); shared RSA (rejected: blast radius)
## Consequences — private keys must be encrypted at rest; rotation is a follow-up (MYT-403)
```

### ✅ Lesson 15.3 checklist
- [ ] 📖 Read & understood — I can explain what an ADR captures and why it's immutable.
- [ ] 💻 Applied — I wrote an ADR for a real decision.
- [ ] 🔍 Found in Mythos — `BE#75` is "aligned with … ADR-0004"; the repo keeps `docs/adr/`.

---

## Lesson 15.4 — The coding constitution

**Goal:** make rules supersede personal preference.

### Concept
A **coding constitution / style guide** is one authoritative document of **non-negotiable** rules
(layering, error handling, naming, function length). It **supersedes personal preference**, is
versioned, and is amended deliberately.

> Mythos has a versioned constitution (v1.2.0, ratified 2026-04-26) covering architecture, type
> safety, API design, the error hierarchy, security, logging, testing, migrations, and SOLID/naming.
> It explicitly states *"Constitution supersedes all other conventions,"* and lists **blocking**
> review comments: `console.*` in new code, bare `throw new Error()` in services, commented-out
> code, a new dependency without rationale.

### ✅ Lesson 15.4 checklist
- [ ] 📖 Read & understood — I can explain what belongs in a constitution and why it's versioned.
- [ ] 💻 Applied — I drafted 5 non-negotiable rules for a project.
- [ ] 🔍 Found in Mythos — the Engineering Constitution is *why* the `BE#88` review reads the way it does.

---

## Lesson 15.5 — Conventional commits & meaningful history

**Goal:** make history machine-readable and intentional.

### Concept
**Conventional commits:** `feat(scope): …`, `fix(scope): …`, `refactor(scope)!: …` (the `!` flags a
**breaking** change). Machine-readable history powers changelogs and communicates intent at a glance.

```
feat(MYT101): add dual-bucket wallet
fix(jwt): reject present-but-invalid token in optional auth
refactor(ts-migration)!: migrate auth domain to typed contracts
chore(release): deploy backend to live
ci(deploy): migrate from S3/CloudFront to ECS
```

### ✅ Lesson 15.5 checklist
- [ ] 📖 Read & understood — I can write conventional commits and mark breaking changes.
- [ ] 💻 Applied — I committed using `type(scope): subject`.
- [ ] 🔍 Found in Mythos — the entire PR corpus uses `feat()/fix()/refactor()!/chore()/ci()`.

---

## Lesson 15.6 — Review as enforcement

**Goal:** make the rules actually matter.

### Concept
A **checklist-driven review** maps each change to its acceptance criteria and flags constitution
violations as **blocking**. *The rules only matter if review enforces them.* Investing in **tooling
the review process itself** keeps governance scaling instead of rotting.

> Mythos `BE#88` review **blocked merge** for a missing migration doc — governance with teeth.
> `BE#89`/`FM#72` added "review skills for easier review process" — tooling the process so it scales.

### ✅ Lesson 15.6 checklist
- [ ] 📖 Read & understood — I can explain why review is the enforcement point.
- [ ] 💻 Applied — I reviewed a change against a checklist and flagged a blocking issue.
- [ ] 🔍 Found in Mythos — `BE#88` blocked on a missing migration doc; `BE#89`/`FM#72` review tooling.

---

## 🎯 Module 15 mastery checklist
- [ ] I can run a spec → plan → implement flow before writing code.
- [ ] I can define "done" objectively and bound scope with in/out tables.
- [ ] I can write an ADR that captures context, options, and consequences.
- [ ] I can draft constitution rules that supersede personal preference.
- [ ] I can write conventional commits and mark breaking changes.
- [ ] I can run a checklist-driven review that blocks on violations.

## 🛠️ Mini-project
For a feature you build in another module's mini-project:
1. Write a one-page **spec** (scope, acceptance criteria) and a **plan** (files, algorithm, open
   questions, DoD) *before* coding.
2. Add an **in/out-of-scope** table and name one deferred follow-up.
3. Write one **ADR** for a real decision you made.
4. Draft a 5-rule **mini-constitution** and review your own PR against it, flagging at least one
   blocking issue.
5. Use **conventional commits** throughout.

## 🔗 Mythos PR map
| PR | What it demonstrates |
|----|----------------------|
| `BE#28`, `FM#42` | Spec-kits initialized (specify→clarify→plan→tasks→implement→checklist). |
| `BE#78` | Plan-first PR (`docs/myt-107-plan.md` before code). |
| `BE#75` | In/out-of-scope table + ADR-0004 reference. |
| `BE#88` | DoD review table; blocked merge for a missing migration doc. |
| `BE#89`, `FM#72` | Review skills/tooling to make governance scale. |
| (all PRs) | Conventional commit titles throughout the corpus. |

## See also
- [05 — Incremental migration](05-incremental-migration.md)
- [10 — Testing strategy & quality gates](10-testing.md)
- [13 — Design principles: SOLID, DRY](13-design-principles.md)
