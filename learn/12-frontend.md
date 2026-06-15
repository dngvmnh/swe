# Module 12 — Frontend Architecture & Rendering

> **Core question:** How do you build a UI that scales and renders fast?
> **Reference doc:** [../12-frontend-architecture-and-rendering.md](../12-frontend-architecture-and-rendering.md)
> **Prerequisites:** [13 — Design principles](13-design-principles.md); pairs with [08 — Security](08-security.md) (XSS) and [05 — Migration](05-incremental-migration.md).

## Why this module
A UI built as one big pile of bespoke screens becomes impossible to keep consistent or fast: every
button differs, the same widget is reimplemented five times, a color change means hunting hundreds
of files. Structure — reusable components, shared tokens, a clear rendering model — is what lets a UI
scale to many pages and contributors without chaos.

### What you'll be able to do
- Choose a rendering model (CSR/SSR/SSG) and handle hydration.
- Compose UIs from shared components with a single source of truth.
- Centralize design tokens and theming.
- Pick the right state tool, decouple cross-cutting concerns, and add error boundaries.

---

## Lesson 12.1 — Rendering models (CSR / SSR / SSG)

**Goal:** know where HTML is produced and the trade-offs.

### Concept
| Model | Where HTML is built | Pros | Cons |
|-------|--------------------|------|------|
| **CSR / SPA** | in the browser | simple hosting (static + CDN) | slower first paint, weaker SEO |
| **SSR** | on the server per request | fast first paint, good SEO | needs a running server (changes hosting — Module [11](11-devops-cicd.md)) |
| **SSG / hybrid** | pre-rendered at build, hydrated on client | fast + cacheable | build-time data only (until revalidated) |

Modern frameworks (Next.js App Router) **mix these per route.**

### ✅ Lesson 12.1 checklist
- [ ] 📖 Read & understood — I can compare CSR/SSR/SSG and their hosting implications.
- [ ] 💻 Applied — I chose a rendering model for a page and justified it.
- [ ] 🔍 Found in Mythos — `FM#43`/`FM#44`: Vite/React SPA → Next.js 16 App Router (SSR/hybrid).

---

## Lesson 12.2 — Hydration & mismatches

**Goal:** understand and avoid hydration bugs.

### Concept
With SSR, the server sends HTML and the client **"hydrates"** it by attaching JS. The server- and
client-rendered markup **must match**, or you get a **hydration mismatch** (flicker or errors). This
is why CSS-in-JS needs **SSR-aware style flushing**, and why environment-dependent output
(timestamps, random values) breaks hydration.

```tsx
// ✅ MUI SSR style flush prevents a hydration mismatch / flash of unstyled content
// emotion-cache-provider.tsx wraps the app so server and client emit matching styles
```

### Check yourself
1. Why does rendering `new Date().toLocaleString()` directly in a component risk a hydration
   mismatch? (Server and client may produce different strings → markup differs.)

### ✅ Lesson 12.2 checklist
- [ ] 📖 Read & understood — I can explain hydration and what causes a mismatch.
- [ ] 💻 Applied — I identified a hydration risk (env-dependent render) in some code.
- [ ] 🔍 Found in Mythos — `FM#43` added `emotion-cache-provider.tsx` for MUI SSR flush to "prevent hydration mismatch."

---

## Lesson 12.3 — Component composition & shared UI

**Goal:** build from reusable pieces with one source of truth.

### Concept
Build from small components; **promote anything used on more than one page into a shared library**
(a `Navbar`, a `Footer`) with a single source of truth. **Barrel exports** (`index.ts`) give
consumers one clean import path. Centralize shared/mock data to avoid **prop drilling**.

```ts
// src/shared/ui/index.ts — a barrel: one clean import path
export { Navbar } from "./navbar";
export { Footer } from "./footer";
// consumer:  import { Navbar, Footer } from "@/shared/ui";
```

### ✅ Lesson 12.3 checklist
- [ ] 📖 Read & understood — I can explain shared components, barrels, and prop drilling.
- [ ] 💻 Applied — I extracted a duplicated component into a shared, barrel-exported one.
- [ ] 🔍 Found in Mythos — `FM#67` shared `Navbar`+`Footer` under `src/shared/ui/`, barrel-exported, replacing per-page navbars.

---

## Lesson 12.4 — Design tokens & theming

**Goal:** make the look consistent and themeable from one place.

### Concept
Centralize colors, spacing, typography as **named tokens** instead of hardcoded values, so the look
is consistent and themeable (light/dark) **from one place.**

### ❌ Bad → ✅ Good
```tsx
// ❌ BAD — hardcoded hex scattered across files; a theme change means hunting all of them
<div style={{ color: "#1a1a1a", background: "#fafafa" }} />
// ✅ GOOD — named tokens; change once, theme everywhere
<div className="text-foreground bg-background" />     // tokens defined in one source
```

### ✅ Lesson 12.4 checklist
- [ ] 📖 Read & understood — I can explain design tokens and why they enable theming.
- [ ] 💻 Applied — I replaced hardcoded colors with tokens.
- [ ] 🔍 Found in Mythos — `FM#23` centralized colors; `FM#63`/`FM#65` aligned forced-dark tokens with Figma.

---

## Lesson 12.5 — State, data fetching & decoupling

**Goal:** pick the narrowest state tool and keep data code portable.

### Concept
- **Pick the narrowest tool:** local UI state vs server cache (React Query/TanStack) vs global app
  state.
- **Decouple cross-cutting concerns.** Don't reach into `window.location` from data code; use an
  **event bus** instead — which also makes it SSR-safe.

### ❌ Bad → ✅ Good
```ts
// ❌ BAD — data layer hard-coupled to the browser; breaks under SSR, hard to test
if (response.status === 401) window.location.href = "/login";
// ✅ GOOD — emit an event; a UI-layer listener decides what to do
if (response.status === 401) authEvents.emit("unauthorized");
```

### ✅ Lesson 12.5 checklist
- [ ] 📖 Read & understood — I can choose a state tool and explain decoupling via an event bus.
- [ ] 💻 Applied — I decoupled a `window`-coupled concern behind an event/callback.
- [ ] 🔍 Found in Mythos — `FM#43` auth event bus "decouples the 401 handler from `window.location`."

---

## Lesson 12.6 — Error boundaries, responsive, timezones

**Goal:** handle the everyday robustness gaps.

### Concept
- **Error boundaries.** A component that catches render-time errors in its subtree and shows a
  fallback, so **one broken widget doesn't blank the whole app** (white screen).
- **Responsive design.** One layout that adapts across breakpoints, not separate mobile/desktop code.
- **Timezone correctness.** Convert server UTC timestamps to local before rendering — the classic
  "render-UTC-as-local" bug.

### ✅ Lesson 12.6 checklist
- [ ] 📖 Read & understood — I can explain error boundaries, responsive layout, and the timezone bug.
- [ ] 💻 Applied — I added a root error boundary or fixed a timezone render.
- [ ] 🔍 Found in Mythos — `FM#63`/`FM#65` root error boundary; `FM#35`/`FM#41` responsive layout; `FM#24` timezone fix.

---

## 🎯 Module 12 mastery checklist
- [ ] I can choose CSR/SSR/SSG per route and explain hosting implications.
- [ ] I can explain hydration and avoid mismatches (SSR-aware styles, no env-dependent render).
- [ ] I can compose shared components with barrels and avoid prop drilling.
- [ ] I can centralize design tokens for consistent, themeable UI.
- [ ] I can pick the right state tool and decouple cross-cutting concerns from `window`.
- [ ] I can add an error boundary, build responsive layouts, and handle timezones.

## 🛠️ Mini-project
Build a small 3-page app (e.g. Next.js):
1. One page SSR, one SSG, one CSR — note the trade-offs you feel.
2. A shared `Navbar`/`Footer` in a `shared/ui` barrel; use design tokens for all colors (add a
   light/dark toggle).
3. A root error boundary with a fallback; deliberately throw in one widget and confirm the rest survives.
4. A data layer that emits an `unauthorized` event instead of touching `window.location`.
5. Render a UTC timestamp converted to the viewer's local time.

## 🔗 Mythos PR map
| PR | What it demonstrates |
|----|----------------------|
| `FM#43`/`FM#44` | SPA → Next.js (App Router); hydration fix; auth event bus; env config. |
| `FM#67` | Shared `Navbar`/`Footer`, barrel export, no prop drilling. |
| `FM#23`, `FM#63`/`FM#65` | Design tokens + theming + root error boundary. |
| `FM#35`/`FM#41` | Responsive layout. |
| `FM#24` | Timezone fix. |
| `FM#39` | Role-aware dual dashboard (composition by authz). |

## See also
- [05 — Incremental migration](05-incremental-migration.md)
- [08 — Security engineering](08-security.md) — XSS in the render path.
- [13 — Design principles: SOLID, DRY](13-design-principles.md)
