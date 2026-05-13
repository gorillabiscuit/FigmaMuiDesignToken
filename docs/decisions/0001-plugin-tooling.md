# ADR-0001: Plugin tooling — webpack + React + TypeScript

**Status:** Accepted
**Date:** 2026-05-13
**Supersedes:** —

---

## What we chose

The Figma plugin is built with:

- **TypeScript (strict)** — same compiler settings as the root `tsconfig.base.json`. Both the sandbox half (`apps/plugin/src/plugin/`) and the UI half (`apps/plugin/src/app/`) are TS.
- **webpack** for bundling — two entry points, two outputs:
  - `src/plugin/controller.ts` → `dist/code.js` (loaded by Figma as `manifest.main`)
  - `src/app/index.tsx` + `src/app/index.html` → `dist/ui.html` (loaded by Figma as `manifest.ui`, inlined HTML with the JS embedded via `html-webpack-inline-source-plugin` or equivalent)
- **React 18** for the UI iframe. ReactDOM client API. No state-management library; `useState` / `useReducer` are sufficient for the violation-list UI.
- **No CSS framework.** Vanilla CSS per Figma's plugin UI conventions (SPEC.md "Visual style" section: 11px body text, Figma palette, no gradients).
- **`@figma/plugin-typings`** as a devDependency for `figma.*` types in the sandbox half.

The directory shape is the SPEC.md layout, mapped onto the monorepo per ADR-0000:

```
apps/plugin/
  manifest.json
  webpack.config.js
  tsconfig.json          ← extends ../../tsconfig.base.json
  src/
    plugin/
      controller.ts
      lintingFunctions.ts
    app/
      index.tsx
      index.html
      components/
      styles/
```

Each new dependency lands in its own commit with a line in `DEPS.md` per CLAUDE.md §5.

## What we rejected

- **Vite.** Faster dev loop and lighter config than webpack. Rejected because the Figma-plugin ecosystem (templates, reference plugins like `design-lint`, community knowledge) is webpack-centric, and the single-HTML-file constraint (`ui.html` must inline its JS) is a webpack-plugin path well-trodden but a Vite path requiring custom config. Revisit if HMR pain dominates.
- **esbuild / tsup directly (no webpack).** Possible but `ui.html` inlining + asset handling is bespoke work. Webpack is paying for itself by being the path everyone else uses.
- **Preact.** ~3kB vs ~45kB gzipped for React. Real win on bundle size, but adds a "you have to remember it's not quite React" footgun for contributors and the size difference is not load-bearing inside Figma. Revisit if bundle size becomes a problem.
- **No React (vanilla DOM / template strings).** Possible — the UI is small. Rejected because the violation list + filter tabs + screen transitions benefit from component composition, and we'd reinvent diffing.
- **Tailwind / shadcn / Mantine.** Too heavy for an 11px-body Figma plugin UI. They optimise for product apps, not for matching Figma's chrome. Vanilla CSS is the right ceiling.
- **CSS-in-JS (styled-components, emotion).** Adds runtime, conflicts with the small-bundle goal, no clear win at this scale.

## Why

- **Matches the SPEC.** SPEC.md "Architecture" and "Build setup" both name TS + React + webpack. This ADR formalises that as the project's tooling rather than re-litigating it.
- **Matches reference plugins.** `design-lint` (the MIT reference cited in the SPEC) uses TS + React + webpack. Lower friction to crib config decisions from a known-good repo.
- **Keeps `packages/shared` framework-free.** React and webpack are confined to `apps/plugin/`. The lint engine in `packages/shared` is plain TS, testable in Node, no bundler step needed for unit tests.
- **No CSS framework matches Figma's house style.** Figma plugins look out of place when they bring their own design system; matching the platform chrome is the cheaper UX win.
- **Bundle target is conservative.** Figma plugins have a practical size ceiling. React + ReactDOM + lint engine should fit comfortably; we'll measure on first bundle and revisit if needed (a "change our mind" signal below).

## What would change our mind

- **Bundle pushes past Figma's comfortable size for a plugin.** Swap React → Preact (compat layer is one webpack alias). If that's still too big, revisit React → vanilla.
- **HMR / iteration speed becomes a real bottleneck.** Webpack's dev experience for Figma plugins is fine but not great. If we hit it daily, evaluate Vite + `vite-plugin-singlefile` for the UI half.
- **A second app surface ships** (e.g. a marketing site, a hosted CI worker). At that point each app picks its own tooling; this ADR remains scoped to `apps/plugin/`.

## Related

- [ADR-0000](./0000-architecture-overview.md) — macro architecture this fits into
- [ADR-0002](./0002-network-access-none.md) — network posture (relevant because webpack config must not introduce runtime fetches)
- [ADR-0003](./0003-token-parser-interface.md) — parser interface (the half that's framework-free)
- SPEC.md "Architecture" and "Build setup" sections
- Reference: [design-lint](https://github.com/destefanis/design-lint) — webpack + React + TS Figma plugin (MIT)
- Reference: [Figma plugin quickstart](https://www.figma.com/plugin-docs/plugin-quickstart-guide/)
