# TokenCheck — product brief

> Working name. Final brand naming is a post-V1 concern.

---

## What this is

TokenCheck is a Figma plugin that validates designs against an external **MUI theme file** (or any JSON / flat-token file) coming from the engineering codebase. The engineering tokens are treated as the **source of truth**; any Figma layer using a value that doesn't appear in that token set is flagged as a violation.

In effect: a linter for designs. Where a code linter checks code against a style guide, TokenCheck checks Figma against the actual code tokens — colours, typography, corner radii, and auto-layout spacing.

The MVP is the inverse of how most "design tokens" plugins work: instead of pushing tokens *from* Figma *into* code, it pulls tokens *from* code and checks Figma against them. This direction is what makes drift visible *before* engineering sees it.

## What this is NOT

- **Not a token authoring / publishing tool.** Designers don't define tokens here; engineers do, in code. The plugin is a read-only consumer of the engineering theme.
- **Not a generic design-system platform.** No multi-team dashboards, no governance UI, no figma-to-code export in V1.
- **Not a Figma → code sync tool.** The dataflow is one-way (code → Figma check). Reverse direction is V2 scope, not V1.
- **Not a server.** No backend, no auth, no hosted state. The plugin runs entirely in the Figma sandbox + UI iframe; parsed tokens live in `figma.clientStorage`.
- **Not a TypeScript parser.** Developers paste / upload JSON. The plugin does not parse `theme.ts` or evaluate `createTheme()` — see SPEC.md "Token file format" for the rationale.

---

## The problem

In teams with many designers, style drift between Figma and the codebase is inevitable. Someone hardcodes a colour, uses the wrong font size, or picks a spacing value that doesn't exist in the theme. This is currently caught (if at all) at engineering handoff — too late, by the wrong person, and with no automatic way to find every offender on a page.

Existing Figma linters check against *Figma styles or local variables*, which means they only catch drift if designers have first reproduced the engineering theme inside Figma. That reproduction step is the part teams skip — and once skipped, the linter loses its source of truth.

TokenCheck removes the reproduction step. The engineering theme JSON *is* the source of truth, loaded directly into the plugin.

---

## Target users

Ranked by priority:

1. **Design-system / frontend engineers maintaining the MUI theme.** They own the source-of-truth file. They want a way to enforce that designers stay within it. **Primary archetype.**
2. **Designers shipping screens for engineering handoff.** They run the plugin pre-handoff to catch their own drift before a developer flags it.
3. **Design-system leads / DesignOps.** They audit consistency across files / teams. Tertiary — they validate the plugin's value but they aren't who we build the UI for first.

---

## Success metrics

### V1 (ship target)

- **Functional:** every V1 scope item shipped; the plugin loads in Figma dev-mode and lints a real screen against a real MUI theme end-to-end.
- **Correctness:** zero false negatives on a hand-built fixture file (every intentionally-hardcoded value gets flagged). False-positive rate target <5% on real screens.
- **Performance:** lint a frame of ≤200 visible nodes in <500ms on a typical laptop. Lint of `figma.currentPage` should not block the UI for >2s on a reasonably-sized file.
- **Usability:** a first-time user can load a theme and run a lint in <60s without reading docs.

### Post-V1

- **Retention:** plugin re-opened by the same user within 7 days of first use (proxy for "useful enough to come back to").
- **Coverage:** non-MUI projects (flat CSS variable JSON) work without bug-reports.
- **Distribution:** published to Figma Community (vs private-team install).

Numbers are aspirational; refine after first real use.

---

## V1 scope (MVP cut-line)

**Held tight.** Anything not on this list is V2.

### IN scope (V1)

- **Parse MUI theme JSON** — auto-detect `palette` key, recursively flatten leaves to dot-paths.
- **Parse flat CSS-custom-property JSON** — `{"--color-primary": "#…"}` shape.
- **Unit conversion** — rem → px using a configurable base font size (default 16px); spacing multiples derived from a configurable base unit (default 8).
- **Lint colours** — fills + strokes (solid only; skip image fills / gradients).
- **Lint typography** — fontFamily, fontSize (px), fontWeight, lineHeight (px). Handle mixed text styles within a single TextNode.
- **Lint corner radius.**
- **Lint spacing** — auto-layout `itemSpacing` + `padding{Top,Right,Bottom,Left}`, checked against multiples of the spacing base unit.
- **Closest-token suggestion** for colours — CIE76 LAB distance.
- **Violation list UI** — grouped by category, filter tabs, click-to-select-and-zoom.
- **Persistence** — parsed tokens stored in `figma.clientStorage` between sessions.
- **Clean state** — explicit "all clean" view when no violations found.

### OUT of V1 — explicit deferrals

- **Fetching tokens from a URL** (GitHub raw / API endpoint) → V2. Requires `networkAccess` change, superseding [ADR-0002](docs/decisions/0002-network-access-none.md).
- **Export Figma values as tokens** (reverse direction) → V2.
- **Shadow / effect linting** → V2.
- **Severity levels (error vs warning)** → V2.
- **Ignore rules** — marking specific layers as intentionally non-standard → V2.
- **CI integration** — headless linting via Figma REST API → V2.
- **Additional design-system adapters** (Tailwind, Style Dictionary, Chakra) → V2. Interface ready per [ADR-0003](docs/decisions/0003-token-parser-interface.md).
- **Opacity / alpha linting** — too many valid non-token use cases → Out of scope entirely.
- **Layout-constraint linting** — not token-related → Out.
- **Linting external library components** — only local work is linted → Out.

---

## Revenue model

Open. The MVP is a free utility. Plausible monetisation paths if it gains traction:

1. **Team / Org tier** — multi-file lint reports, shared ignore-rule registries, Slack/Linear integrations.
2. **CI add-on** — headless lint via Figma REST API, surfacing violations in a PR status check.
3. **Premium token-source connectors** — Style Dictionary, Tailwind, Chakra, design-system platforms (Specify, Knapsack).

Architectural implications: keep the token parser pluggable (an interface, not hard-coded MUI), keep the lint engine pure (testable without Figma), keep the Figma plugin as the *first* surface but not the *only* one.

---

## Defensible moats

In rough order of strength:

1. **MUI-aware normalisation.** Generic linters don't know that `typography.body1.fontSize: "1rem"` should be compared against a Figma fontSize of 16. Getting MUI's structural conventions right (rem conversion, spacing-as-base-unit-multiples, palette nesting) is non-trivial and project-specific.
2. **Code-as-source-of-truth direction.** Most plugins push Figma → code. The opposite direction catches a class of drift the other can't, and requires zero designer setup ("paste this JSON" vs "rebuild your theme inside Figma").
3. **Pluggable parser surface.** Once the lint engine is pure and the parser is an interface, supporting Tailwind / Style Dictionary / Chakra / flat CSS vars is incremental work, not a rewrite.

---

## Open product decisions (deferred to V2)

- **Brand naming.** "TokenCheck" is the working name from the spec.
- **Distribution model.** Figma Community (public) vs private-team install? Affects whether we add usage telemetry, error reporting, support process.
- **Non-MUI design system support.** Tailwind config, Chakra theme, Style Dictionary output. Likely first 1B feature after URL fetching.
- **URL-fetching auth model.** GitHub raw URLs work without auth; private repos need a token. How do we accept and store one? (Stop-and-ask gate per CLAUDE.md §4.)
- **Telemetry.** Currently `networkAccess: none`. Any analytics requires a manifest change + ADR.
- **Quick-fix actions.** Should the plugin offer "apply the closest token" as a one-click fix, or stay read-only? Read-only is the V1 default.

---

_Last updated 2026-05-13. Treat this document as the contract for what's in MVP and what isn't — anything outside the IN-scope list above requires explicit negotiation per CLAUDE.md §4 (stop-and-ask). Full technical spec lives in `SPEC.md`._
