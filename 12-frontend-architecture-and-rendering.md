# 12 — Frontend Architecture & Rendering

## The idea

Frontend architecture is how you organize a user interface so it stays fast, consistent, and
changeable as it grows. The big structural decisions are about **rendering** (where HTML is
produced — in the browser, on the server, or both), **component composition** (building UIs from
small reusable pieces), **state & data flow** (where data lives and how it moves), and **a design
system** (shared tokens and components so the UI is coherent and not re-invented per screen).

## Why it exists

A UI built as one big pile of bespoke screens becomes impossible to keep consistent or fast: every
button looks slightly different, the same widget is reimplemented five times, and a color change
means hunting through hundreds of files. Structure — reusable components, shared tokens, a clear
rendering model — is what lets a UI scale to many pages and many contributors without chaos.

## Patterns & trade-offs

- **Rendering models.**
  - *CSR (client-side rendering / SPA)*: the browser downloads JS and builds the page. Simple
    hosting (static + CDN), but slower first paint and weaker SEO.
  - *SSR (server-side rendering)*: HTML is rendered on the server per request — faster first paint,
    better SEO, but needs a running server (changes your hosting model — see
    [doc 11](11-devops-cicd-and-deployment.md)).
  - *SSG / hybrid*: pre-render static pages at build time, hydrate interactivity on the client.
    Modern frameworks (Next.js App Router) mix these per route.
- **Hydration.** With SSR, the server sends HTML and the client then "hydrates" it by attaching JS.
  The server- and client-rendered markup must match, or you get a **hydration mismatch** (flicker
  or errors) — a major reason CSS-in-JS needs SSR-aware style flushing.
- **Component composition & shared UI.** Build from small components; promote anything used on more
  than one page into a shared library (a `Navbar`, a `Footer`) with a single source of truth.
  **Barrel exports** (`index.ts`) give consumers one clean import path.
- **Design tokens.** Centralize colors, spacing, typography as named tokens instead of hardcoded
  values, so the look is consistent and themeable (light/dark) from one place.
- **State management & data fetching.** Local UI state vs server cache (React Query/TanStack) vs
  global app state — pick the narrowest tool. Decouple cross-cutting concerns (e.g. a global 401
  handler) via an event bus rather than reaching into `window.location` from data code.
- **Responsive design.** One layout that adapts across breakpoints rather than separate mobile/
  desktop code paths.
- **Error boundaries.** A component that catches render-time errors in its subtree and shows a
  fallback, so one broken widget doesn't blank the whole app.

## Pitfalls

- **Hardcoded design values** scattered everywhere → inconsistent UI, painful theme changes.
- **Duplicated components/utilities** → divergent behavior (see [doc 13](13-design-principles-solid-dry.md)).
- **Hydration mismatches** from non-SSR-aware styling or environment-dependent render output.
- **Timezone bugs** — rendering server UTC timestamps in the browser without converting to local.
- **No error boundary** → a single thrown error unmounts the whole tree (white screen).
- **`dangerouslySetInnerHTML` with user data** → XSS (see [doc 08](08-security-engineering.md)).
- **Coupling data-layer code to navigation/`window`** → hard to test, breaks under SSR.

## Seen in Mythos

- **SPA → SSR framework migration — `FM#43`, `FM#44`.** The frontend moved from a Vite/React SPA
  with TanStack Router to **Next.js 16 (App Router)**, installed *alongside* the old SPA (Strangler
  Fig — see [doc 05](05-incremental-migration-strangler-fig.md)). `FM#44` migrated routing patterns
  to `next/navigation`.

- **Solving hydration explicitly — `FM#43`.** The PR added
  `src/shared/ui/emotion-cache-provider.tsx` for **MUI SSR style flush** — its stated purpose is to
  "prevent hydration mismatch" and the "flash of unstyled content." A concrete instance of the
  SSR/hydration pitfall and its standard fix.

- **Decoupling cross-cutting concerns — `FM#43`.** An **auth event bus** (`shared/api/auth-events.ts`)
  "decouples the 401 handler from `window.location`," and `process.env`-based API config replaces
  Vite's `import.meta.env`. Both are about not coupling data/infra code to the browser environment —
  which also makes it SSR-safe.

- **Shared UI with a single source of truth — `FM#67` (MYT-57).** A shared `Navbar` + `Footer` under
  `src/shared/ui/`, replacing the legacy per-page MUI navbar app-wide, exported via a **barrel**
  (`index.ts`), with mock data centralized to avoid **prop drilling**. New code standardizes on
  Tailwind v4 + shadcn/ui + Radix — a deliberate, consistent component layer.

- **Design tokens over hardcoded values — `FM#23`, `FM#63`/`FM#65`.** `refactor: Revamp and replace
  hard-coded colours` (`FM#23`) centralizes color; `FM#63`/`FM#65` align "forced-dark tokens with
  Figma" — a design-token system feeding light/dark theming from one source.

- **Error boundary — `FM#63`/`FM#65`.** Adds a **root error boundary** so a render error shows a
  fallback instead of a blank app.

- **Responsive layout — `FM#35`, `FM#41`.** Restructured responsive landing-page layout (`FM#35`)
  and mobile metric-tile alignment (`FM#41`) — one adaptive layout across breakpoints.

- **Timezone correctness — `FM#24`.** `fix(timezone): Fix timezone mismatch` — the classic
  render-UTC-as-local bug.

- **Role-aware composition — `FM#39`.** A dual dashboard rendering different views for creator vs
  user roles — composition driven by authorization state.

## See also

- [05 — Incremental migration](05-incremental-migration-strangler-fig.md)
- [08 — Security engineering](08-security-engineering.md) (XSS in the render path)
- [13 — Design principles: SOLID, DRY](13-design-principles-solid-dry.md)
