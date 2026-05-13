# ADR-0000: Architecture overview

**Status:** Draft. To be reviewed and finalised in Phase 1 by the Claude Code session that picks up after kickoff.
**Date:** `<DATE>`
**Supersedes:** —

This ADR is unusual — it's not a single decision; it's the macro shape of the system, written upfront so subsequent ADRs refine specific choices without re-litigating the whole picture. Treat it as a strawman: write the first draft from the product brief; the new session should challenge anything that looks wrong.

---

## What we chose

`<One paragraph describing the macro shape. Example: "A Next.js monorepo with a tRPC API, Postgres for data, Inngest for scheduled jobs, and a cleanly-separated <core-module> package that can be re-exposed as a public product later.">`

Concretely:

```
<ASCII diagram of the system. Show the request flow, the package boundaries,
the data stores, the third-party integrations, and the boundaries between
"phase 1A" and "deferred to phase 2+" components.>
```

---

## What we rejected

- **`<Alternative architecture 1>`** — `<why this isn't right for the product brief>`
- **`<Alternative architecture 2>`** — `<...>`
- **`<Alternative architecture 3>`** — `<...>`

---

## Why

`<The reasoning. Address: which architectural invariants from CLAUDE.md §2 this satisfies; which Phase 1A scope items it enables; which Phase 1B/2 features it leaves room for; what trade-offs were accepted.>`

---

## What would change our mind

- `<concrete signal that would invalidate this shape>`
- `<another signal>`

---

## Related

- `PROJECT.md` — product brief that drove this shape
- `docs/decisions/QUEUE.md` — pending stack-selection ADRs that refine this overview
- Future ADRs 0001–0099 — each picks a specific library / pattern that fits inside this shape
