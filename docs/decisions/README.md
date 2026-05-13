# Architecture Decision Records (ADRs)

Architectural decisions for this project. Each ADR captures one decision with the alternatives we rejected and the signals that would make us reconsider. Lightweight — one page each. Format defined in `_template.md`.

## Format

Five sections per ADR: **Status / Date / What we chose / What we rejected / Why / What would change our mind / Related.**

## When to write one

Per `CLAUDE.md §10`, ADRs cover architectural decisions that:

- Have lasting impact on the codebase shape
- Have rejected alternatives worth recording (so we don't re-debate)
- A future contributor (or future-you) might question

Not every code-level choice needs an ADR. Pick a library? Probably not unless it's load-bearing. Pick a state-management approach across the whole frontend? Yes, ADR.

## When to update one

When the world changes:

- A library we picked has changed enough that the rejected alternatives now look better — re-decide via a new ADR that supersedes the old one
- A "what would change our mind" signal fires — same: new ADR, supersedes the old
- A typo or factual error — edit in place, don't create a new ADR

## Index

| # | Title | Status | Date |
|---|---|---|---|
| 0000 | Architecture overview | Draft | `<date>` |

(More entries land as Phase 1 progresses. See `QUEUE.md` for pending decisions.)

## Numbering

ADRs are numbered chronologically as they're decided. Don't renumber. If an ADR is superseded, the superseding ADR gets a new number and references the old one — the old one gets a `Superseded by ADR-YYYY` line in its Status field.
