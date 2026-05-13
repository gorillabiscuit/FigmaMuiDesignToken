# ADR Queue — pending decisions

This queue is **empty**.

TokenCheck is a Figma plugin with no backend, no API, no auth, and no database. The template's standard stack-selection decisions (DB / ORM / API surface / auth / observability / job orchestration / hosting platform) are **vacated by the architecture** chosen in ADR-0000 and intentionally not filled in.

The decisions that *do* apply to a Figma plugin are either resolved or N/A:

| Topic | Status | Where |
|---|---|---|
| Macro architecture | Accepted | [ADR-0000](./0000-architecture-overview.md) |
| Plugin tooling (bundler + UI framework + language) | Accepted | [ADR-0001](./0001-plugin-tooling.md) |
| Network egress posture | Accepted | [ADR-0002](./0002-network-access-none.md) |
| Token parser interface | Accepted | [ADR-0003](./0003-token-parser-interface.md) |
| Monorepo tool | Resolved | pnpm workspaces — already scaffolded; no separate ADR needed |
| Testing | Resolved | Vitest (root config) + manual Figma dev-import for the plugin runtime |
| Styling | Resolved | Vanilla CSS following Figma's plugin UI conventions (11px body, Figma palette). No CSS-in-JS, no Tailwind. Revisit if UI grows. |
| Hosting / distribution | Deferred to post-V1 | Figma Community vs private install — see `PROJECT.md` open decisions |
| Observability / error reporting | Deferred to post-V1 | No backend to observe; client-side error reporting requires `networkAccess` change → new ADR |
| Database / ORM | N/A | No persistent server-side state |
| API surface | N/A | No server |
| Auth | N/A | No accounts |
| Job orchestration | N/A | No background work |
| Privacy compliance | N/A in V1 | No user data leaves the user's device; ADR-0002 keeps it that way |

If a real architectural decision arrives, add a section here with a strawman, then resolve it into a numbered ADR using `_template.md`.
