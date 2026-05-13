# DEPS.md — npm dependency justification

One line per dependency in any `package.json` in this monorepo. Dev dependencies and runtime dependencies both go here. Per `CLAUDE.md §5`, every new dep gets its own commit AND a line here, in the same commit.

## Why this exists

The day-1 dependency choices fade from memory in months. "Why do we have `lodash` if we have `radash`" or "what is `clsx` for?" become unanswerable conversations. Forcing a one-line justification per dep at install time keeps the answers live and forces "do we actually need this" thinking.

## Format

One row per dep:

```
- **<package-name>** *(@version-range)* — <one-line reason. Be specific. "utility library" doesn't count.>
```

Group by package within the monorepo (root, then per-app, per-package).

## Example

```
- **next** *(^15.0.0)* — the framework. ADR-0002.
- **@trpc/server** *(^11.0.0)* — internal API surface. ADR-0003. Pinned to v11+ for new app router compatibility.
- **zod** *(^3.22.0)* — runtime validation; backbone of tRPC schemas + form validation.
- **clsx** *(^2.1.0)* — conditional className composition for Tailwind. Lighter than classnames.
```

---

## Root (workspace)

- **@eslint/js** *(^9.18.0)* — base ESLint JS rules. Required by the flat config in `eslint.config.mjs`.
- **@types/node** *(^20.10.0)* — Node typings for scripts + config files.
- **eslint** *(^9.18.0)* — linter. CLAUDE.md §3 banned patterns enforced here.
- **eslint-config-prettier** *(^10.0.0)* — disables ESLint rules that conflict with Prettier.
- **globals** *(^15.14.0)* — env globals for ESLint (`node`, `browser`, etc).
- **husky** *(^9.1.0)* — git hook wiring for `.husky/pre-commit` + `.husky/pre-push`. CLAUDE.md §5 + §7.
- **lint-staged** *(^15.0.0)* — runs eslint + prettier on staged files in `pre-commit`.
- **prettier** *(^3.4.0)* — formatter.
- **typescript** *(^5.7.0)* — type-checker.
- **typescript-eslint** *(^8.20.0)* — TS plugin for ESLint flat config.
- **vitest** *(^3.0.0)* — test runner. CLAUDE.md §8.2.

---

## apps/`<your-app-name>`

`<Add per-app deps here as you install them.>`

---

## packages/shared

`<Add per-package deps here.>`
