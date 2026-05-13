# ADR-0000: Architecture overview

**Status:** Accepted (strawman — challenge anything that looks wrong)
**Date:** 2026-05-13
**Supersedes:** —

This ADR is the macro shape of the system. Subsequent ADRs refine specific choices without re-litigating the whole picture. It is the contract `CLAUDE.md §1` and §2 point at.

---

## What we chose

TokenCheck is a **Figma plugin** with no backend, no server, no auth, no database, and no network calls (V1). The product is shaped entirely by Figma's plugin runtime model, which forces a two-process design:

1. **Plugin sandbox** — pure JS in Figma's plugin VM. Has access to the Figma scene graph (`figma.*`). Cannot use DOM, cannot fetch, cannot use most browser globals.
2. **UI iframe** — a sandboxed `<iframe>` with HTML/CSS/JS (React). Has DOM access but no access to `figma.*`. Communicates with the plugin sandbox via `postMessage`.

These two halves are bundled together by webpack and shipped as one plugin per Figma's `manifest.json`. There is no third runtime.

All non-Figma-specific logic — token parsing, unit conversion, colour distance, lint comparison — lives in `packages/shared` so it is independently testable in plain Node with Vitest and never depends on a running Figma instance. This is the moat boundary: keep the lint engine pure.

Concretely:

```
                    ┌──────────────────────────────────────────┐
                    │           Figma Desktop / Browser         │
                    │                                           │
   ┌────────────────┴───────────┐         ┌────────────────────┴────┐
   │  Plugin sandbox            │         │  UI iframe (React)      │
   │  apps/plugin/src/plugin/   │         │  apps/plugin/src/app/   │
   │                            │         │                          │
   │  controller.ts             │◀───────▶│  App.tsx                 │
   │  ─ traverse scene graph    │ post-   │  ─ SetupScreen           │
   │  ─ extract style values    │ Message │  ─ ResultsScreen         │
   │  ─ call lint engine        │         │  ─ ViolationItem         │
   │  ─ select+zoom on click    │         │                          │
   │                            │         │  figma.clientStorage     │
   │  figma.* (scene graph,     │         │  is accessed via         │
   │  clientStorage)            │         │  postMessage relay       │
   └────────────┬───────────────┘         └──────────────┬───────────┘
                │                                        │
                │  both import from                      │
                ▼                                        ▼
   ┌──────────────────────────────────────────────────────────────────┐
   │  packages/shared  (pure TS, platform-agnostic, no figma.* / DOM) │
   │                                                                   │
   │  ─ tokenParser.ts        MUI theme / flat CSS-var JSON → tokens   │
   │  ─ lintEngine.ts         compare extracted values to tokens       │
   │  ─ colorDistance.ts      CIE76 LAB distance                       │
   │  ─ unitConvert.ts        rem → px, spacing-base multiples         │
   │  ─ types.ts              ParsedToken, Violation, LintSummary      │
   │                                                                   │
   │  Tested with Vitest in plain Node. No Figma, no jsdom.            │
   └───────────────────────────────────────────────────────────────────┘

   ────────────────────────────  no network access (V1)  ─────────────
                            networkAccess.allowedDomains = ["none"]
   ───────────────────────────────────────────────────────────────────

   ▼ Phase 1B / 2 (DEFERRED, not built yet)

   ─ networkAccess for URL token fetching (GitHub raw / private repo)
   ─ Figma REST API + CI integration (headless lint on PRs)
   ─ Reverse direction: export Figma values as tokens
```

### Repo layout that this implies

```
apps/
  plugin/              ← the Figma plugin (one manifest → one bundle)
    manifest.json
    src/
      plugin/          ← sandbox-side: figma.* API consumer
      app/             ← UI iframe: React tree
    webpack.config.js
packages/
  shared/              ← pure TS lint engine + parsers (no figma.*, no DOM)
docs/
  decisions/           ← this file lives here
scripts/
.claude/
```

This deviates from the scaffold's current placeholder name `apps/web/`. **Proposed rename: `apps/web/` → `apps/plugin/`**, since there is no web app. The rename is queued for Phase 2 bootstrap and tracked in `docs/decisions/QUEUE.md`.

It also deviates from SPEC.md's flat `tokencheck/` directory. The monorepo split is chosen so the lint engine can be tested without Figma (CLAUDE.md §2 platform-agnostic invariant), and so a future CI surface (V2) can reuse `packages/shared` without rewriting.

---

## What we rejected

- **Standalone web app (no Figma plugin).** Designers will not context-switch out of Figma to lint a file, and the lint requires read access to the Figma scene graph — only the plugin API exposes that for unpublished work. Rejected.
- **Figma REST API + CI-only (no in-Figma plugin).** Would work for *published* files, but most drift exists in unpublished branches the REST API cannot reach. Also makes feedback loop slow (commit → CI → comment). Deferred to V2 as a *supplement* to the plugin, not a replacement.
- **Flat single-package layout per SPEC.md `tokencheck/src/{plugin,app,shared}/`.** Simpler but couples the lint engine to the Figma bundle — every test run would need the plugin's webpack pipeline. We keep `packages/shared` as a separately-tested package so the moat code is fast to test and reusable from a future CI surface.
- **No React (vanilla JS UI).** Possible — the UI is small. But the violation list + filter tabs + screen transitions benefit from component composition, and React + ReactDOM adds ~45kB gzipped which is acceptable for a Figma plugin. Revisit if bundle size becomes a problem (Preact swap is one-line).
- **Parsing `theme.ts` directly in the plugin.** TypeScript parsing in-plugin is heavy (ts-morph, swc, or similar). The cost-vs-value is poor: developers can `JSON.stringify(theme)` once. Per SPEC.md, paste-JSON is the contract.
- **Style Dictionary as the canonical input format.** Style Dictionary is fine but adds an authoring step for MUI projects (which already have a structured theme). MUI JSON in, Style Dictionary support deferred to 1B.
- **Storing tokens in a published Figma library / variables.** That's the inverse of this plugin's whole thesis — it would re-introduce the "reproduce the theme inside Figma" step that makes existing plugins lose their source of truth.
- **A separate `apps/ui/` for the iframe.** Considered splitting plugin sandbox and UI iframe into two `apps/*`. Rejected because they ship together as one Figma manifest and webpack already produces both bundles from one config — splitting adds workspace overhead with no real isolation benefit.

---

## Why

- **Two-process model is mandated by Figma.** Not a choice; written down so future contributors don't waste time looking for a single-process alternative.
- **Pure-logic `packages/shared` satisfies CLAUDE.md §2** ("platform-agnostic — no DOM, no React Native, no Node-specifics"). Token parsing and CIE76 distance are pure functions over plain data. This is the right home and it gives us moat-quality tests via Approach B (CLAUDE.md §8.1).
- **`networkAccess: ["none"]` for V1** is the simplest privacy posture. No data leaves the user's machine. Theme JSON is pasted or uploaded locally. Revisiting this requires an ADR (URL fetching = new external ingress = §4 stop-and-ask).
- **No backend / no auth / no DB** matches the V1 scope — the plugin operates entirely on the active Figma file and a per-user `clientStorage` slot. Most of the CLAUDE.md stop-and-ask list (auth, DSAR, schema migrations) is therefore vacuously satisfied. The remaining live concerns are: networkAccess changes, manifest changes, anything that touches user data persistence semantics.
- **Webpack for the bundle** matches the standard Figma plugin pattern (the SPEC's reference, `design-lint`, also uses it). Vite for Figma plugins is workable but the ecosystem around webpack + Figma is more mature. Revisit at Phase 1B if HMR pain dominates.
- **React for the UI** is the same trade-off existing Figma plugins make (`design-lint` uses React). Bundle cost is acceptable; component composition is worth it for the violation list and filter tabs.

---

## What would change our mind

- **Figma ships a first-party lint hook.** If Figma exposes "validate against external token JSON" as a platform feature, this plugin's reason to exist evaporates — revisit.
- **CI surface becomes the primary use case.** If teams want headless lint on every PR more than they want in-Figma lint, the architecture flips: `packages/shared` stays, the plugin becomes optional, and a Node CLI / GitHub Action becomes the primary surface. The pure-logic split here is designed to make that flip cheap.
- **Bundle size becomes a problem in Figma.** Plugins have a soft size limit; if React + ReactDOM + lint engine pushes past it, swap to Preact (compat shim) before considering vanilla JS.
- **Non-MUI design systems dominate inbound demand.** If most users arrive with Tailwind / Style Dictionary / Chakra rather than MUI, the parser interface needs to be the primary product surface and MUI becomes one adapter of many. Doable inside this architecture — `tokenParser.ts` becomes `tokenParsers/{mui,tailwind,…}.ts` behind a shared `Parser` interface.
- **A second app surface ships** (e.g. a marketing/landing site, a hosted lint dashboard). Then the rename `apps/web/` → `apps/plugin/` becomes load-bearing rather than cosmetic, and a new `apps/web/` may be re-introduced for the marketing site.

---

## Related

- `SPEC.md` — full technical spec (sole source of truth for V1 behaviour).
- `PROJECT.md` — product brief that drove this shape (cut-line for what's in / out of V1).
- `docs/decisions/QUEUE.md` — remaining Phase 1 decisions. Most stack-selection ADRs (DB, ORM, auth, API surface, observability) are **vacated by the no-backend architecture** and should be deleted from the queue, not filled in.
- Future ADR-0001 — accept the rename `apps/web/` → `apps/plugin/` and create per-package `CLAUDE.md` for the new path.
- Future ADR-0002 — accept webpack + React + TS for the plugin bundle (formalises the existing dependencies as their own commit per CLAUDE.md §5).
- Future ADR-0003 — accept the token-parser interface (MUI adapter first, others later).
- Reference: [Figma Plugin API](https://www.figma.com/plugin-docs/api/figma/), [design-lint](https://github.com/destefanis/design-lint) (MIT — pattern to follow), [MUI default theme](https://mui.com/material-ui/customization/default-theme/).
