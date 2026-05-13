# Per-vendor runbooks

One page per third-party dependency. The point is not exhaustive vendor
documentation — it's enough to answer three questions in a hurry:

1. **What breaks if this vendor is down?** What's the user-visible impact?
2. **What's the manual fallback?** Can we keep operating in some degraded
   mode while the vendor recovers, or do we just have to wait?
3. **Where do we look to confirm it's the vendor and not us?** Status page,
   our own observability, log signatures.

Each runbook is intentionally short — under ~100 lines. If a runbook starts
growing into a multi-page guide, that's a sign the vendor is significant
enough to warrant its own ADR or on-call doc, not a longer runbook.

## Index

`<List one entry per vendor as they're added. Example:>`

- `<vendor-name>.md` — `<one-line description>` (per `ADR-XXXX`)

## Conventions

- **Status page** links go to the vendor's official status page. If a vendor
  doesn't have one, that's noted explicitly — outages have to be inferred from
  our own signals.
- **Cost signals** capture the free-tier ceiling and the first paid step.
  "Something to watch" not "exact pricing" — pricing pages are authoritative.
- **Key rotation** is the procedure for revoking + reissuing the vendor's
  primary credential after a leak. Rotation should be possible from the
  vendor's dashboard without code changes (env-var-only update).
- **No on-call paging** is wired up by default. These runbooks are for
  manual reference during incidents, not for automation.

## When to update

- A new vendor is added → add a runbook in the same PR as the dep.
- A vendor's status page URL changes → update the runbook.
- A vendor outage exposed a failure mode we hadn't documented → capture it
  under "What breaks" and link the post-mortem if there is one.

## Template for a new runbook

```markdown
# Runbook: `<Vendor>`

**What it is:** `<one-line role in our stack>`. Per [ADR-XXXX](../decisions/XXXX-slug.md).

## What breaks if it's down

- `<impact 1>`
- `<impact 2>`

## Manual fallback

`<degraded mode, or "there isn't one in Phase 1A">`

## Status page

`<URL>`

## How we tell it's the vendor and not us

- `<log signature or observability signal that points at vendor failure>`
- `<our own metric/dashboard that goes red when this vendor fails>`

## Key rotation

`<step-by-step from the vendor's dashboard>`

## Cost signals

- Free tier ceiling: `<...>`
- First paid step: `<...>`
- Something to watch: `<usage that climbs unexpectedly>`
```
