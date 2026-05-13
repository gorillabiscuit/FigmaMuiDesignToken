# apps/web — package-specific CLAUDE.md

Inherits everything from the root `CLAUDE.md`. This file holds rules that apply ONLY to the web app.

> **Skeleton.** Fill in once Phase 1 stack-selection ADRs land and Phase 2 bootstrap is underway.

---

## What this is

`<Framework, version, target browsers. Example: "Next.js 15+ App Router, React 19, TypeScript strict. Deployed on Vercel. Consumes the tRPC API exposed at /api/trpc/* (which lives in this same Next.js app — no separate backend service in MVP).">`

## Imports allowed

- From `packages/shared` — types, Zod schemas, hooks, utils
- From `packages/ui` — React components designed for web (if you have a `packages/ui`)
- Standard Node / framework / React libraries

`<List any other packages this app may import from.>`

## Imports forbidden

- From `apps/<another-app>` — no cross-app imports ever
- From packages designed for a different platform (e.g. React Native packages from a web app)
- `<Add project-specific forbiddens — e.g. "No @vercel/* imports per ADR-XXXX's lock-in firewall," "No CSS-in-JS at runtime per ADR-XXXX," etc.>`

## Web-specific banned patterns (in addition to root §3)

- **No client-side fetching to external APIs** — all external calls go through the API layer (server-side) so we control caching, rate limiting, and credentials. Browser making a direct API call to a third-party = ADR-required exception.
- **No `localStorage` / `sessionStorage` for sensitive data.** Auth tokens are managed by the auth provider; user preferences sync to the server.
- `<Add project-specific bans as they emerge.>`

## Server Components vs Client Components

> Only relevant if your framework has this distinction (Next.js App Router, etc).

- **Server Components by default.** Read-heavy pages — title pages, list views, profile views — should be RSC.
- **Client Components only where interactivity demands.** Forms, modals, anything with `useState` / `useEffect`.
- **Mark Client Components with `"use client"`** at the top of the file. Make this an explicit, intentional choice.

## Routing

- `<App Router / Pages Router / file-based / etc.>`
- Route groups for layouts (`(marketing)`, `(authed)`).
- Loading and error boundaries per route segment.

---

_Refine this file as the web app's patterns settle. When in doubt, default to root `CLAUDE.md` rules._
