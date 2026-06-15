# Module 13 — Design Principles: SOLID, DRY & Clean Code

> **Core question:** What makes code easy to change six months later?
> **Reference doc:** [../13-design-principles-solid-dry.md](../13-design-principles-solid-dry.md)
> **Prerequisites:** none, but it makes [06](06-error-handling.md), [07](07-api-design.md), and [12](12-frontend.md) click.

## Why this module
Most of a program's lifetime cost is in *changing* it, not writing it the first time. These
principles don't make code "work" (tests do that) — they make code **survivable** for the next
person, who is often you. They're heuristics, not laws.

### What you'll be able to do
- Apply each SOLID principle with a concrete code shape.
- Apply DRY without falling into *false* DRY.
- Use clean-code heuristics (guard clauses, small pure functions, immutability, naming).
- Recognize over-abstraction and stop before you cause it.

---

## Lesson 13.1 — Why principles exist (the cost of change)

**Goal:** adopt the right frame.

### Concept
Code is **read far more than written, and changed far more than read once.** Tangled, duplicated,
deeply nested code makes every change risky because you can't predict what an edit will break. These
principles are accumulated wisdom about keeping the cost of the *next* change low.

### ✅ Lesson 13.1 checklist
- [ ] 📖 Read & understood — I can explain why "easy to change" is the goal, not "clever."
- [ ] 💻 Applied — I named a change that was painful because of poor structure.
- [ ] 🔍 Found in Mythos — these are written into the Engineering Constitution and enforced in review.

---

## Lesson 13.2 — SOLID, part 1: S, O, L

**Goal:** apply the first three.

### Concept
- **S — Single Responsibility.** A module/function has **one reason to change.** If its name needs
  an "and," split it.
- **O — Open/Closed.** Open for extension, closed for modification. Add new behavior as a **new
  module**, not an `if (type === 'new')` branch bolted into existing code.
- **L — Liskov Substitution.** A subtype must be usable anywhere its base is. **Wrap external
  providers behind an interface** so swapping one changes only the adapter.

```ts
// O — adding an email type is a NEW module, not an edit to a switch
// emails/welcome.ts, emails/receipt.ts each: (data) => ({ subject, html })  ← pure
// L — swap email providers by changing only the adapter behind a stable interface
interface EmailProvider { send(to: string, subject: string, html: string): Promise<void>; }
class ResendProvider implements EmailProvider { /* ... */ }   // swap → only this file changes
```

### ✅ Lesson 13.2 checklist
- [ ] 📖 Read & understood — I can state S, O, L with an example each.
- [ ] 💻 Applied — I split a function with an "and" in its name (S) or added behavior via a new module (O).
- [ ] 🔍 Found in Mythos — `BE#35` email templates are pure fns; adding an email is a new module (Open/Closed).

---

## Lesson 13.3 — SOLID, part 2: I, D

**Goal:** apply the last two.

### Concept
- **I — Interface Segregation.** Depend on **narrow** interfaces; don't force callers to know about
  fields they don't use. Pass `Pick<User,'id'|'email'>`, not the whole `User`.
- **D — Dependency Inversion.** Depend on **abstractions, not concretions.** Call a factory
  (`initSupabase()`), don't `new Client()` inline — so the dependency can be swapped/mocked.

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — concrete dependency created inline; can't swap or mock
function getUser(id: string) { const db = new SupabaseClient(URL, KEY); /* ... */ }
// ✅ GOOD — depend on a factory/abstraction
function getUser(id: string, db = getSupabase()) { /* ... */ }   // injectable & mockable
```

### ✅ Lesson 13.3 checklist
- [ ] 📖 Read & understood — I can state I and D with examples.
- [ ] 💻 Applied — I narrowed a fat parameter (I) or replaced an inline `new` with a factory (D).
- [ ] 🔍 Found in Mythos — services call `initSupabase()`/`getStripeClient()`/`getResend()`, never `new Client()`; `BE#35` narrowed `Request.user`.

---

## Lesson 13.4 — DRY (and false DRY)

**Goal:** remove real duplication, avoid fake coupling.

### Concept
**DRY:** each piece of knowledge has **one authoritative home.** Duplication is a bug waiting to
happen — a fix applied to one copy silently misses the other. **But beware *false* DRY:** don't
couple two things that merely *look* alike today but change for different reasons.

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — uploadFileToS3 defined identically in TWO files; they will diverge
// listing-service.ts:  function uploadFileToS3(...) { ...A... }
// profile-service.ts:  function uploadFileToS3(...) { ...A... }
// ✅ GOOD — one shared util; both import it
// utils/s3.ts:  export function uploadFileToS3(...) { ... }
```

> Rule of thumb: logic used in **≥2 places** belongs in a shared `utils/`. But if two snippets are
> equal *by coincidence* (e.g. two validation rules that happen to be `len > 3`), leave them separate.

### ✅ Lesson 13.4 checklist
- [ ] 📖 Read & understood — I can explain DRY and how false DRY causes harmful coupling.
- [ ] 💻 Applied — I deduplicated real duplication into a shared module.
- [ ] 🔍 Found in Mythos — `FM#20` deduped `uploadFileToS3` from two files into one shared util.

---

## Lesson 13.5 — Clean-code heuristics

**Goal:** write code that reads top-to-bottom.

### Concept
- **Guard clauses.** Check preconditions at the top, return/throw immediately; keep nesting shallow
  (≤2 levels). (See [Module 06](06-error-handling.md), Lesson 6.4.)
- **Small functions.** A function over a threshold (e.g. ~50 lines) is doing too much — treat length
  as a **hard refactor signal** and extract named helpers.
- **Pure functions.** Data transforms, formatting, and template rendering should be **side-effect-
  free** — trivially testable and reusable.
- **Immutability.** Prefer `const`; never mutate function arguments; return new objects.
- **Named constants.** No magic strings/numbers — give them names (`LAUNCH_TOKEN_EXPIRY_SECONDS`).
- **Naming conventions.** Verbs for functions, `is*`/`has*` for booleans — self-documenting code.

```ts
// ❌ BAD — an 82-line function doing fetch + validate + charge + ledger + payout
async function meterSession(...) { /* ...82 lines... */ }
// ✅ GOOD — split by responsibility; each helper does one thing
async function meterSession(...) {
  const charge = await resolveIdempotentCharge(...);   // one thing
  return callMeteredChargeRpc(charge);                 // one thing
}
```

### ✅ Lesson 13.5 checklist
- [ ] 📖 Read & understood — I can list the clean-code heuristics and the function-length signal.
- [ ] 💻 Applied — I extracted a long function into named single-purpose helpers.
- [ ] 🔍 Found in Mythos — `BE#88` split `meterSession` (~82 lines) into `resolveIdempotentCharge()` + `callMeteredChargeRpc()`; constants live in `src/constants/`.

---

## Lesson 13.6 — The over-abstraction pitfall

**Goal:** know when to *stop* applying principles.

### Concept
Principles are **heuristics, not laws.** Over-applied, they cause:
- **Over-abstraction** → indirection nobody needs, premature generality.
- **"DRY" that couples unrelated code** (false DRY).
- A factory/interface for something that has exactly one implementation and always will.

> Balance: apply a principle when it reduces the cost of a *likely* future change — not to satisfy
> the principle.

### ✅ Lesson 13.6 checklist
- [ ] 📖 Read & understood — I can give an example of harmful over-abstraction.
- [ ] 💻 Applied — I removed (or chose not to add) an unnecessary abstraction.
- [ ] 🔍 Found in Mythos — the reference doc flags over-abstraction/false-DRY as explicit pitfalls.

---

## 🎯 Module 13 mastery checklist
- [ ] I can apply each SOLID principle with a concrete code shape.
- [ ] I can apply DRY and explain when *not* to (false DRY).
- [ ] I can write guard clauses, small pure functions, and immutable code.
- [ ] I can replace magic values with named constants and use intention-revealing names.
- [ ] I can recognize over-abstraction and stop before causing it.

## 🛠️ Mini-project
Refactor a messy function (yours or one you write to be bad on purpose):
1. It should start as one 60+ line function with deep nesting, magic numbers, an inline `new Client()`,
   and a duplicated helper.
2. Apply, in order: guard clauses → extract single-purpose helpers (SRP, ≤50 lines) → named constants
   → factory/injection (DIP) → dedupe the helper (DRY) → narrow a fat parameter (ISP).
3. Write a one-paragraph note on one place you *chose not* to abstract, and why.

## 🔗 Mythos PR map
| PR | What it demonstrates |
|----|----------------------|
| `FM#20` | DRY: deduped `uploadFileToS3`. |
| `BE#88` | SRP + function length: split `meterSession`; guard clauses; named constants. |
| `BE#35` | DIP/Liskov (factories, adapters), ISP (narrow `Request.user`), Open/Closed (email templates). |
| `BE#31`–`BE#47` | Narrow typed contracts per boundary (ISP). |

## See also
- [06 — Error handling & typed errors](06-error-handling.md)
- [07 — API design & layered architecture](07-api-design.md)
- [15 — Spec-driven development & governance](15-spec-governance.md)
