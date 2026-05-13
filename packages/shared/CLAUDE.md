# packages/shared — package-specific CLAUDE.md

Inherits everything from the root `CLAUDE.md`. This file holds rules that apply ONLY to `packages/shared`.

---

## What this is

The platform-agnostic package. Code here MUST run on every consumer platform — web (Node + browser), mobile (React Native, if applicable), Edge runtimes, build-time scripts. Common contents:

- Types (TypeScript interfaces, Zod schemas)
- Pure utility functions
- Cross-platform hooks (must work in both React-DOM and React-Native, or be flagged as web-only)

## Imports allowed

- Standard library
- Other `packages/*` packages that are themselves platform-agnostic
- Universally-supported npm packages (`zod`, `date-fns`, `clsx`, etc.)

## Imports forbidden

- **No DOM APIs.** `document`, `window`, `navigator`, `localStorage`, etc. ESLint enforces this via `no-restricted-globals` per `eslint.config.mjs`.
- **No React Native APIs.** `react-native`, `expo-*`, etc.
- **No Node-specific APIs** unless wrapped behind a platform-detection layer.
- **No HTTP framework imports.** No `next`, no `express`, no `trpc`. tRPC routers IMPORT this package, not the other way around.
- **No direct database / ORM imports.** No `pg`, no Prisma, no Drizzle. Data comes in as plain objects via function arguments.
- **No React / DOM / RN imports.** Pure logic only.
- **No file I/O** (`fs.readFile` etc) without an ADR. Data should be loaded by the caller and passed in.
- **No mutable global state.** Functions are pure. Caches at this layer are pure-function memoisation only (e.g., `lru-cache`).

## Why these rules

If `packages/shared` becomes coupled to a specific platform, it stops being shared. Every coupling forces the next consumer (a future mobile app, a future Edge function, a future test runner) to either work around the coupling or rewrite the code.

Treat this package as if you're writing a library you'll publish to npm — even though you won't.

## Testing requirements

- All non-trivial logic uses Approach B (sub-agent isolation) per CLAUDE.md §8.1.
- Tests live alongside the code (`*.test.ts` next to `*.ts`).
- Tests must run in a pure Node environment — no jsdom, no browser globals.
- Edge cases that MUST be tested: empty inputs, boundary values, error paths.

---

_Refine this file as the shared package's patterns settle. The platform-agnostic invariant is non-negotiable; everything else can evolve._
