# ADR-0003: Token parser interface — pluggable, MUI-first

**Status:** Accepted
**Date:** 2026-05-13
**Supersedes:** —

---

## What we chose

Token parsing is structured as **an interface in `packages/shared` with one adapter per supported input format**. V1 ships two adapters: MUI theme JSON, and flat CSS-custom-property JSON. The lint engine consumes the interface's output — a flat list of `ParsedToken` — and knows nothing about the input format.

```ts
// packages/shared/src/parsers/index.ts
export interface TokenParser {
  /** Cheap predicate — does this shape look like our format? */
  canParse(input: unknown): boolean;
  /** Throws if shape is invalid; returns the flat token list otherwise. */
  parse(input: unknown, config: ParserConfig): ParsedToken[];
}

export interface ParserConfig {
  baseFontSizePx: number;   // for rem → px conversion (default 16)
  spacingBaseUnitPx: number; // for spacing multiples (default 8)
}
```

Adapter selection at runtime: try each registered parser's `canParse` in order; first match wins. If none match, surface an error to the UI with a "this doesn't look like an MUI theme or a flat token JSON" message.

V1 adapters (both in `packages/shared/src/parsers/`):

- `muiTheme.ts` — recognises an object with a `palette` key. Recursively flattens to dot-paths (`palette.primary.main`, `typography.body1.fontSize`, `shape.borderRadius`, `spacing`). Applies unit conversion per `ParserConfig`. Treats MUI's `spacing` as a base unit (any positive integer multiple is a valid spacing token).
- `flatCssVars.ts` — recognises a flat object of `--*: <value>` pairs. Categorises by name prefix heuristics (`--color-*` → colour, `--font-*` → typography, etc); falls back to value-shape inference (hex / px / number) if the prefix is ambiguous.

The output `ParsedToken` shape is defined exactly as in SPEC.md "Types":

```ts
interface ParsedToken {
  path: string;
  value: string;
  category: 'color' | 'typography' | 'spacing' | 'radius';
  originalValue: string;
}
```

The lint engine receives `ParsedToken[]` and compares Figma-extracted values against it. The engine does not know whether the source was MUI or flat CSS.

## What we rejected

- **Hard-code MUI shape into the lint engine.** Simpler in V1, but the moat (PROJECT.md) is partly *pluggability* — Tailwind / Style Dictionary / Chakra are the next adapters and we don't want a rewrite to add them. Cost of the interface is one file; cost of avoiding it is a rewrite.
- **Make the interface async / return a `Result<…>` type.** Parsing is pure, synchronous, and small enough that throw-on-invalid is fine. Adding async / `Result` is speculative complexity for V1.
- **Adopt Style Dictionary's data model as the canonical interface.** Style Dictionary is well-designed but its model is heavier (transforms, formats, references) than what we need. We can write a Style Dictionary adapter that emits `ParsedToken[]` later; we don't need to adopt its model wholesale.
- **Zod schemas for each input format.** Tempting — runtime validation is nice. Rejected for V1 because (a) it pulls in zod as a runtime dep, (b) the shapes we accept are loose (MUI themes vary across versions and projects), and (c) we already need a fallback heuristic for category inference. A targeted set of throw-with-helpful-message checks beats a schema that's too strict to ever pass.
- **Auto-detect more than two formats in V1.** SPEC.md names MUI + flat CSS vars; we don't go beyond. Tailwind / Style Dictionary / Chakra arrive when there's evidence of demand, each as its own commit.

## Why

- **Moat alignment.** PROJECT.md identifies pluggable parsing as a moat. This ADR is what makes that real — adding a new adapter is a contained change, not a refactor.
- **Test boundary alignment.** The interface gives a clean test target: feed an adapter a fixture, assert the output `ParsedToken[]`. Sub-agent isolation (CLAUDE.md §8.1, Approach B) is straightforward because the contract is small.
- **Engine purity.** The lint engine reads `ParsedToken[]` and Figma-extracted values and decides matches. It has no branches for MUI specifics. This is the property that makes V2 CI surface (running the engine on extracted JSON from a Figma REST snapshot) trivially possible.
- **Conservative surface.** One interface, two adapters in V1, no zod / no async / no DI framework. Easy to read, easy to extend.

## What would change our mind

- **The two-adapter set fails on a real-world MUI theme.** Common cause: developers extend MUI with custom palette branches the recursive flattener doesn't anticipate. Fix is per-case (update the MUI adapter) — interface holds.
- **Adapters need to share large amounts of code.** If MUI and Tailwind adapters both want, say, the same colour normaliser, lift it into `packages/shared/src/parsers/util/`. Doesn't change the interface.
- **A user-supplied parser becomes a feature.** ("Paste your own JS that turns my custom format into `ParsedToken[]`".) That's a major security surface (running user JS in the plugin sandbox) and would need a dedicated ADR.

## Related

- [ADR-0000](./0000-architecture-overview.md) — macro architecture; identifies the lint engine as the moat boundary
- [PROJECT.md](../../PROJECT.md) — moats section lists pluggable parsing
- [SPEC.md](../../SPEC.md) — "Token file format", "Flattening logic", "Unit conversion", "Types"
- Future: Tailwind / Style Dictionary / Chakra adapters arrive as new files under `packages/shared/src/parsers/`, no new ADR per adapter unless one introduces a runtime-significant dep or risk
