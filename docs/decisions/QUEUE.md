# ADR Queue — pending stack-selection decisions

**This is Phase 1 work.** Walk through each entry below with the human, finalise as a real numbered ADR using `_template.md`, then commit. The order is roughly dependency order — earlier choices constrain later ones.

Each entry below has:
- The decision to make
- A strawman recommendation (the previous-session Claude's or your own preliminary view)
- The alternatives that were considered and why they're not the recommendation
- What might change the recommendation

These are NOT decisions yet. They're strawmen for the new-session Claude + human to challenge or accept.

---

## How to work through this queue in Phase 1

1. **Read each entry below with the human.**
2. **For each: confirm the recommendation, push back, or pick an alternative.** Don't accept silently — make the human articulate why they agree.
3. **Write the real ADR file** at `docs/decisions/000X-<title-slug>.md` using `_template.md`. Status = "Accepted". Date = today.
4. **Commit each ADR as its own commit** (`docs(adr): accept ADR-0001 monorepo tool` etc).
5. **Mark this `QUEUE.md` entry as resolved** by deleting that section and adding a one-liner to `README.md`'s index table.

Once all pending entries are accepted: Phase 1 done, move to Phase 2 (repo bootstrap).

---

## Starter list of Phase 1 decisions to make

Add one section per decision below. Suggested decisions for a typical web-app project (skip any that don't apply):

### Monorepo tool

**Strawman:** plain pnpm workspaces (turbo or nx if build caching becomes a bottleneck).

**Alternatives + why not:**
- **turbo** — adds build caching across packages, but extra config + concept overhead.
- **nx** — heavy / opinionated; valuable for large teams, overkill for solo.
- **Yarn workspaces** — fine but pnpm is faster + stricter about phantom deps.

**Would change our mind:** more than ~5 packages with shared build steps that take >30s each.

---

### Frontend framework

`<fill in with the decision shape: strawman, alternatives, change-our-mind signals>`

---

### API surface

`<...>`

---

### Auth provider

`<...>`

---

### Database

`<...>`

---

### ORM

`<...>`

---

### Job orchestration

`<...>`

---

### Observability stack

`<...>`

---

### Hosting platform

`<...>`

---

### Styling approach

`<...>`

---

### Component library

`<...>`

---

### Testing approach

`<...>`

---

### Privacy compliance approach

`<...>`

---

`<Add project-specific decisions below as needed — payment provider, vector store, ML inference stack, etc.>`
