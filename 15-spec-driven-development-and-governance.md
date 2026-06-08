# 15 — Spec-Driven Development & Governance

## The idea

Beyond writing code, a team needs a way to **decide what to build and how to build it**, and to
**remember those decisions** so they survive turnover and time. Two complementary practices:

- **Spec-driven development** — write the specification *before* the code: what the feature does,
  its acceptance criteria, the API contract, the data model, the open questions. The spec is the
  source of truth the implementation is checked against.
- **Governance** — the durable rules and records that keep a codebase coherent: a coding
  constitution, architecture decision records (ADRs), migration docs, commit/PR conventions, and a
  review process that enforces them.

## Why it exists

Code alone doesn't capture *intent* or *rationale* — six months later nobody remembers why the
wallet has two buckets or why downgrades don't claw back credits. Without written specs, features
drift from what was agreed; without governance, every contributor invents their own conventions and
the codebase fragments. Writing the decision down *before* and *around* the code is how a team
scales beyond what fits in one person's head.

## Patterns & trade-offs

- **Spec → plan → implement.** Author a spec (scope, acceptance criteria, in/out of scope), then a
  plan (files to touch, algorithm, open questions, definition-of-done), then implement against it.
  Catches design problems while they're cheap (a paragraph) instead of expensive (a rewrite).
- **Definition of Done (DoD).** An explicit checklist a change must satisfy — acceptance criteria
  met, tests added, docs updated — so "done" is objective, not a feeling.
- **In-scope / out-of-scope tables.** State plainly what a PR does *not* do and which follow-ups
  it defers, so reviewers don't expect the world and scope creep is visible.
- **Architecture Decision Records (ADRs).** Short immutable docs capturing a significant decision,
  its context, the options considered, and the consequences. Future readers learn *why*, not just
  *what*.
- **A coding constitution / style guide.** One authoritative document of non-negotiable rules
  (layering, error handling, naming, function length). It *supersedes* personal preference and is
  versioned and amended deliberately.
- **Conventional commits & meaningful history.** `feat(scope): …`, `fix(scope): …`,
  `refactor(scope)!: …` (the `!` flags breaking). Machine-readable history powers changelogs and
  communicates intent at a glance.
- **Review as enforcement.** A checklist-driven review maps each change to its acceptance criteria
  and flags constitution violations as *blocking*. The rules only matter if review enforces them.

Trade-off: process has overhead. Too much ceremony slows small changes; too little lets a growing
team fragment. Calibrate to team size and blast radius — heavier for schema/payments/auth.

## Pitfalls

- **Code without a spec** → builds the wrong thing efficiently.
- **Decisions only in people's heads / chat** → lost rationale, repeated debates, "why is this like
  this?" with no answer.
- **A constitution nobody enforces** → decorative rules, drift anyway.
- **Editing an applied decision record/migration doc** → rewrites history others relied on.
- **Vague "done"** → endless re-litigation of whether a feature is finished.
- **Scope creep** from unstated boundaries.

## Seen in Mythos

- **A real Engineering Constitution.** Mythos has a versioned constitution (v1.2.0, ratified
  2026-04-26) covering architecture (strict layering), language/type safety, API design, the error
  hierarchy, security, logging, testing, migrations, and SOLID/naming rules. It explicitly states
  *"Constitution supersedes all other conventions"* and lists **blocking** review comments
  (`console.*` in new code, bare `throw new Error()` in services, commented-out code, new dependency
  without rationale). This document is *why* the `BE#88` review reads the way it does — every flag
  cites a constitution rule.

- **Spec-kit, front and back — `BE#28`, `FM#42`.** Both repos initialized a "spec-kit"
  (`feat(claude): Initialize back-end spec-kit` `BE#28`; `feat(spec): Initialize front-end
  spec-kit` `FM#42`), with a documented flow `specify → clarify → plan → tasks → implement →
  checklist`.

- **Spec/plan-before-code in practice — `BE#78`.** The launch-endpoint PR *opened with only a
  planning doc* (`docs/myt-107-plan.md`) "covering all files to create, algorithm, open questions,
  and DoD checklist" — implementation followed in later commits. Textbook spec→plan→implement, with
  open questions flagged *before* writing code.

- **In/out-of-scope discipline — `BE#75`.** The signing-keys PR has an explicit two-column "In scope
  (this PR) / Out of scope (follow-ups)" table and names the separate ticket (MYT-403/KMS) that owns
  deferred work — visible scope boundaries.

- **ADRs referenced by features — `BE#75`.** Implementation is described as "aligned with Jira spec
  and **ADR-0004**," and the repo keeps `docs/adr/`. Decisions are recorded and then *cited* by the
  code that follows them.

- **Definition of Done as a review artifact — `BE#88`.** The review opens with a DoD table mapping
  each acceptance criterion to ✅/⚠️/🔄 status — "done" is checked against written criteria, not vibes.

- **Migration docs as immutable records — constitution, `BE#44`, `BE#75`, `BE#88`.** Every schema
  change requires `docs/migrations/NNNN-*.md` (description, affected tables, SQL up/down, prod-applied
  dates) and *must not be edited after being applied*. The `BE#88` review **blocked merge** for a
  missing migration doc — governance with teeth.

- **Conventional commits everywhere.** The PR titles in this very corpus are the convention in
  action: `feat(MYT101):`, `fix(jwt):`, `refactor(ts-migration)!:`, `chore(release):`,
  `ci(deploy):` — scoped, typed, breaking-marked.

- **Tooling the review process itself — `BE#89`, `FM#72`.** `feat: Add review skills for easier
  review process` — the team even invests in making governance *easier to apply*, so the process
  scales instead of rotting.

## See also

- [05 — Incremental migration](05-incremental-migration-strangler-fig.md)
- [10 — Testing strategy & quality gates](10-testing-strategy-and-quality-gates.md)
- [13 — Design principles: SOLID, DRY](13-design-principles-solid-dry.md)
