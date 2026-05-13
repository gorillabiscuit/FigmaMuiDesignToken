# ADR-0002: V1 has zero network access

**Status:** Accepted
**Date:** 2026-05-13
**Supersedes:** —

---

## What we chose

`manifest.json` is shipped with:

```json
"networkAccess": {
  "allowedDomains": ["none"]
}
```

The plugin makes **zero** outbound HTTP requests in V1. All token input is pasted into a textarea or uploaded as a local `.json` file. All persisted state lives in `figma.clientStorage` (per-user, on-device). No analytics, no error reporting, no remote config, no auto-update of token sources.

Changing this is a CLAUDE.md §4 stop-and-ask gate. Adding any domain to `allowedDomains` requires a new ADR that supersedes this one and addresses:

- Which domain(s) and what data leaves the device
- The user-facing consent / disclosure
- The failure mode if the remote is unreachable
- Whether any data the remote returns is treated as trusted input (it shouldn't be — see §"What we rejected")

## What we rejected

- **Allow `https://raw.githubusercontent.com` for direct GitHub-hosted theme files.** Genuinely useful — a developer pastes a URL instead of JSON. Deferred to V2 because: (a) it's not in the SPEC's V1 scope, (b) private repos still need auth tokens, which is a much bigger surface, and (c) a pasted JSON works perfectly well as a starting point.
- **Allow analytics / error reporting (Sentry, PostHog, etc).** Useful for debugging real-world breakage, but inverts the privacy posture (designs and theme content become observable in aggregate). Defer until we have evidence the cost is worth it; if added, route through a privacy-reviewed proxy, never the vendor's SDK directly.
- **Allow a single dev-mode telemetry endpoint behind a feature flag.** Tempting and reasonable, but rejected because "behind a flag" tends to drift toward "on by default for some users". One bright line is easier to reason about: zero, until we re-decide.
- **`allowedDomains: ["*"]` during development.** No. The dev manifest must match the production manifest; otherwise we ship a posture we haven't tested.

## Why

- **Simplest privacy posture.** No data egresses the user's machine. We can publicly state "TokenCheck does not phone home" without footnotes. This is a real moat in a category where designers are increasingly wary of plugins that exfiltrate file contents.
- **Matches the SPEC.** SPEC.md "manifest.json" pins `networkAccess` to `["none"]` and notes the V2 caveat.
- **No vendor risk surface.** No supply-chain dependency on analytics SDKs, no rate-limit failure modes, no privacy-policy gotchas in V1.
- **The user always has the JSON in front of them.** This isn't a power loss — copying a theme into the plugin is a 5-second operation. The marginal value of fetching is automation, which is the V2 contract.

## What would change our mind

- **A real ergonomic pain point emerges.** Teams update their theme weekly and re-pasting becomes annoying. Then URL fetching (V2) is worth the privacy + auth complexity, gated behind a clear ADR.
- **CI surface (V2) ships.** Headless lint via Figma REST API necessarily involves network; that's a separate surface from the in-Figma plugin and would have its own ADR.
- **A critical client-side bug is impossible to diagnose without remote error reporting.** That would justify the smallest possible egress — a privacy-reviewed error endpoint, opt-in, no payload beyond stack + plugin version.

## Related

- [ADR-0000](./0000-architecture-overview.md) — macro architecture; lists this as a hard invariant
- CLAUDE.md §2 — "`networkAccess` stays `['none']` until an ADR says otherwise"
- CLAUDE.md §4 — new external ingress / egress is a stop-and-ask
- SPEC.md "manifest.json" section
- Reference: [Figma `networkAccess` docs](https://www.figma.com/plugin-docs/manifest/#networkaccess)
