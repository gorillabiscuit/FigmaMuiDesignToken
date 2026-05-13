# TokenCheck — roadmap

**Status:** living document
**Last updated:** 2026-05-13

This is the sequenced milestone view of [V1 scope in PROJECT.md](../PROJECT.md). It captures the build order; PROJECT.md remains the cut-line contract.

The project is intentionally small. There are no formal phases — just **V1** (what we ship) and **V2** (what we deliberately don't). The milestones below are V1.

---

## Sequenced milestones

Each milestone ships as it lands. M3 is the vertical-slice de-risker — first end-to-end loop through the plugin.

### M1 — Bootstrap

- Rename `apps/web/` → `apps/plugin/`; update root `pnpm-workspace.yaml`, the package's `CLAUDE.md`, and `package.json` `name`.
- Add `apps/plugin/manifest.json` per SPEC.md (`networkAccess: ["none"]`, `documentAccess: "dynamic-page"`).
- Install + commit (one commit per dep, with `DEPS.md` lines): `webpack`, `webpack-cli`, `ts-loader`, `html-webpack-plugin`, `html-webpack-inline-source-plugin` (or equivalent), `@figma/plugin-typings`, `react`, `react-dom`, `@types/react`, `@types/react-dom`.
- `apps/plugin/webpack.config.js` with two entries (`code`, `ui`) and the single-HTML-file UI output.
- `apps/plugin/src/plugin/controller.ts` — empty handler that logs "loaded" on plugin open.
- `apps/plugin/src/app/index.tsx` — hello-world iframe.
- Verify: `pnpm preflight` passes; `pnpm --filter=plugin build`; manual Figma → Plugins → Development → Import plugin from manifest → plugin opens, UI renders.

**Outcome:** the plugin runs in Figma; no lint logic yet.

### M2 — Lint engine + parsers (pure, in `packages/shared`)

- `types.ts` — `ParsedToken`, `Violation`, `LintSummary` per SPEC.md.
- `parsers/` — MUI adapter and flat-CSS adapter behind the [ADR-0003](decisions/0003-token-parser-interface.md) interface. Auto-detect by `canParse`.
- `unitConvert.ts` — `rem → px` with configurable base; spacing-base-multiple validator.
- `colorDistance.ts` — sRGB → LAB conversion + CIE76 distance. No library; pure function.
- `lintEngine.ts` — pure function `(extractedValues, parsedTokens) → Violation[]`. Knows about category, not about Figma.
- Vitest tests: Approach B sub-agent isolation for the lint engine (it's moat code). Approach A for parsers and unit-convert. Fixtures: a real MUI theme JSON, a real flat-CSS JSON, and a hand-built drift fixture.

**Outcome:** the engine is correct, fully unit-tested in plain Node, and can be lifted out into a V2 CI surface without modification.

### M3 — First vertical slice: colours end-to-end

- Plugin controller traverses `figma.currentPage` (visible nodes only, skip external library components).
- Extracts solid fills + strokes (skip image / gradient fills). Sends a typed `EXTRACT_RESULT` message to UI.
- UI: setup screen accepts pasted JSON, calls `tokenParser` from `packages/shared`, shows "loaded N tokens". "Lint" button triggers extraction → engine → violation list.
- Results screen: violation rows; click selects + zooms via `figma.viewport.scrollAndZoomIntoView`.
- Persist parsed tokens in `figma.clientStorage`.
- Manual test: load a real MUI theme, lint a hand-built screen with deliberate colour drift, every violation appears and click-to-zoom works.

**Outcome:** the loop works for colours. This is the de-risker — every later milestone is "add a category to a working loop".

### M4 — Typography linting

- Extract `fontName`, `fontSize`, `fontWeight` (derived from `fontName.style`), `lineHeight` from every `TEXT` node.
- Handle mixed text styles via `getRangeFontSize` / `getRangeFontName` (SPEC.md).
- Engine flags per-property; UI groups violations under the layer.
- Filter tab "Typography" works in the results screen.

**Outcome:** typography drift is caught and surfaced.

### M5 — Radius + spacing linting

- Extract `cornerRadius` (handle `figma.mixed` for per-corner values).
- Extract `itemSpacing`, `paddingTop/Right/Bottom/Left` from auto-layout frames only.
- Engine checks: radius against `shape.borderRadius` (+ any other radius tokens); spacing against multiples of the configured spacing base unit.
- Filter tabs "Radius" and "Spacing" work.

**Outcome:** all V1 categories covered.

### M6 — UI polish + persistence

- Base-font-size + spacing-base inputs on setup screen, persisted alongside tokens.
- "Re-lint" + "Change tokens" actions.
- Clean state ("all clean — every value matches your tokens").
- Counts: "Loaded N tokens (X colours, Y typography, Z spacing, W radii)".
- Summary bar on results: "N issues across M layers", breakdown by category.
- Closest-token suggestion text per row.
- Empty state for "no nodes scanned".
- Accessibility: keyboard navigation through the violation list (real keyboard users in design teams exist).

**Outcome:** the UI matches SPEC.md's "Plugin UI" section.

### M7 — Ship readiness

- Run on a real MUI project (your own `theme.ts` → JSON → real screen). Iterate on false positives.
- Performance check: lint ≤200-node frame in <500ms; lint full page should not freeze UI for >2s (use chunked traversal if needed).
- README install instructions (clone, `pnpm install`, `pnpm --filter=plugin build`, Figma dev import).
- Tag `v0.1.0`.

**Outcome:** shippable as a private / dev-mode plugin. Figma Community publication is a separate post-V1 step (see PROJECT.md open decisions).

---

## Parallelism + dependency notes

- **M2 has no Figma dependency** — it can be built and finished in plain Node before M1 ships, if useful. M3 depends on both M1 and M2.
- **M4 and M5 can run in parallel after M3 lands** — they extend the same loop with different categories.
- **M6 polish work can run in parallel with M4 / M5** — UI polish doesn't depend on which categories are wired up.

---

## Explicitly deferred to V2 (per PROJECT.md cut-line)

- Fetching tokens from a URL — requires `networkAccess` change → new ADR superseding [ADR-0002](decisions/0002-network-access-none.md).
- Exporting Figma values as tokens (reverse direction).
- Shadow / effect linting.
- Severity levels (error vs warning).
- Ignore rules (mark specific layers as intentionally non-standard).
- CI integration via Figma REST API.
- Additional design-system adapters (Tailwind, Style Dictionary, Chakra) — interface ready per [ADR-0003](decisions/0003-token-parser-interface.md), adapters added on demand.

---

## Maintenance

- Refine this file as scope is learned, but PROJECT.md remains the contract.
- When a milestone completes, mark it ✅ and link the relevant commit / tag.
- When a deferred item moves into V1, update both this file AND PROJECT.md (per CLAUDE.md §4 stop-and-ask).
