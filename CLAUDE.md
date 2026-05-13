# CLAUDE.md — working contract

This file is the contract between the human maintainer and the AI coding agent on this repo. It exists because AI-assisted code accumulates invisible technical debt fast without explicit guardrails. Every rule has been chosen because it catches an *AI-specific* failure mode, not just as general good engineering.

Keep this file under 500 lines. Detail goes in `docs/decisions/` (ADRs), per-package `CLAUDE.md` files, or external skills.

---

## 1. Project identity

- **Service:** **TokenCheck** — a Figma plugin that lints designs against an external MUI theme JSON (or any flat-token JSON) coming from the engineering codebase. Engineering tokens are the source of truth; any Figma value that doesn't match a token is flagged.
- **Differentiator:** code-as-source-of-truth direction (engineering theme → Figma check), with MUI-aware normalisation (rem→px, spacing-base multiples, palette nesting). Most existing plugins go the other way (Figma → code) and lose source of truth the moment a designer skips the "rebuild the theme in Figma" step. See `PROJECT.md` "Defensible moats".
- **Phase:** V1 (MVP). Target: TBD; no time pressure; ship when ready. Small project — V1 / V2 buckets only, no formal phase ladder. See `docs/ROADMAP.md` for sequenced milestones.
- **Tracker:** TBD (likely GitHub Issues on this repo until volume justifies Linear). Until set, treat the IN-scope list in `PROJECT.md` as the de-facto backlog.
- **Read first:** `SPEC.md` (full technical spec — single source of truth for V1 behaviour), `PROJECT.md` (product scope + cut-line), `docs/decisions/0000-architecture-overview.md` (system shape).

### Repo structure

```
apps/
  plugin/     ← the Figma plugin (one manifest → one webpack bundle)
                src/plugin/   sandbox-side controller (figma.* consumer)
                src/app/      UI iframe (React)
packages/
  shared/     ← pure TS: token parser, lint engine, colour distance, types.
                No figma.*, no DOM, no Node-specifics. Tested with Vitest in plain Node.
docs/
  decisions/  ← ADRs
  runbooks/   ← per-vendor incident references (sparse — V1 has no vendors)
scripts/      ← project-level scripts (gates, hooks)
.claude/
  commands/   ← Claude Code slash commands (e.g. /pre-pr)
```

> **Note:** the scaffold currently ships `apps/web/` as a placeholder. There is no web app — that directory is queued for rename to `apps/plugin/` as part of Phase 2 bootstrap. Until renamed, treat references to `apps/plugin/` in this file as forward-looking.

Each `apps/*` and `packages/*` has its own `CLAUDE.md` for package-specific rules. Root `CLAUDE.md` (this file) holds global rules.

---

## 2. Architectural invariants

Things that MUST be true. Violations get reverted, not patched.

- **Cross-package import direction.** `apps/plugin` imports from `packages/shared`. No app imports from another app. `packages/shared` imports from nothing inside this repo.
- **`packages/shared` is platform-agnostic.** No `figma.*`, no DOM APIs (`document`, `window`, `localStorage`), no React, no Node-specific APIs unless wrapped behind a platform-detection layer. The lint engine and token parser MUST be testable in plain Node with Vitest — no jsdom, no Figma mock harness. This is the moat boundary (per ADR-0000).
- **The plugin sandbox does not touch the DOM.** Code under `apps/plugin/src/plugin/` runs in Figma's plugin VM, which has no `document` / `window`. Anything DOM-related belongs in `apps/plugin/src/app/` (the UI iframe) and communicates back via `postMessage`.
- **The UI iframe does not touch `figma.*`.** Code under `apps/plugin/src/app/` has no access to the Figma API. Anything that needs to read or mutate the scene graph happens in the sandbox and is mediated by typed `postMessage` payloads.
- **`networkAccess` stays `["none"]` until an ADR says otherwise.** V1 makes zero outbound requests. Any change to `manifest.json`'s `networkAccess` is a stop-and-ask (§4), a new ADR, and a privacy review — it's new external egress.
- **Token JSON is pasted/uploaded locally.** The plugin does NOT parse TypeScript and does NOT call `eval` / `Function` / dynamic `import()` on user input. The contract per SPEC.md is: user provides JSON; we parse JSON.
- **Persisted user state lives in `figma.clientStorage` only.** No cookies, no localStorage, no remote stores in V1. Anything beyond `clientStorage` is a new ingress / egress decision (§4).
- **The lint engine is a pure function of (Figma-extracted values, parsed tokens).** No global state, no side effects, no I/O. This is what makes the moat testable and what would let a future CI surface reuse the engine without a Figma runtime.

---

## 3. Banned patterns

The agent must not do any of these. If instructed to, stop and flag.

- **No `any` type.** Use `unknown` and narrow, or define a type. `// @ts-ignore` and `// @ts-expect-error` similarly banned without a one-line comment explaining why.
- **No `==` / `!=`.** `===` / `!==` only.
- **No `var`.** `const` by default; `let` if reassigned.
- **No console.log in committed code.** Use the project logger (introduce one when adding observability). `console.log` is a code-review-blocker.
- **No silent catch.** `try { ... } catch {}` and `try { ... } catch (e) { /* ignored */ }` both banned. Either log it, re-throw it, or surface it via the error-handling layer — never swallow.
- **No `as` type casts** without a one-line comment explaining why the type system can't verify it.
- **No hallucinated imports.** If the symbol doesn't exist in a real npm package at the version we have installed, don't write code calling it. TypeScript and ESLint catch most cases but verify.
- **No PII / secrets in source, tests, or fixtures.** Use env vars and `.env.example` placeholders.
- **No third-party scripts loaded into the web app at runtime** without an ADR and a stop-and-ask. (Performance, privacy, supply-chain risk.)
- **No `setTimeout` / `setInterval` for retry logic.** Use a typed retry library (e.g. `p-retry`) or write an explicit retry abstraction in `packages/shared`.
- **No `document` / `window` in `packages/shared`.** Platform-agnostic means platform-agnostic. Use `useEffect` or platform-specific code in the consuming app.
- **No `eval`. No `new Function(...)`. No dynamic `require()`.**
- **No production code paths gated by `NODE_ENV`.** Use feature flags or environment-driven config — not `if (NODE_ENV === 'production')` scattered through code.

---

## 4. Stop-and-ask list

Before doing any of these, pause and confirm with the human in the chat:

- **Adding a new dependency to any `package.json`.** Each new dep gets its own commit + a line in `DEPS.md` justifying it.
- **Rewriting a file when only a small change was requested.**
- **Changing the public shape of any API procedure** (path, input schema, output schema).
- **Changing the response semantics of an existing endpoint** in a way that could break a client, even if the shape looks identical. Announce as `BREAKING CHANGE vs <ticket>` and wait for confirmation.
- **Adding a new top-level directory** (`apps/foo`, `packages/bar`, anything new at the root).
- **Schema migrations that touch existing tables** — drop/rename/alter columns, change or add indexes on populated tables, change enum members, add constraints with retroactive effect. New tables and purely additive nullable columns are usually safe. Anything else needs a rollback plan, a sandbox dry-run, and confirmation.
- **Touching auth, JWT handling, session management, or privacy-control code.**
- **Changing consent, deletion, or privacy-control flows** — cookie/consent banner UX, account deletion routes, region / age-verification columns, analytics opt-in/out wiring, any new cookie or persisted user preference, DSAR (Article 15/17/20) handling. ADRs on privacy compliance are law here; wrong defaults are legally significant.
- **Introducing a new cross-cutting pattern** (a shared base class, a middleware, a global state slice).
- **Deleting or renaming anything in `docs/decisions/`.**
- **Calling an external API from new code paths** — confirm rate limits, caching, and error handling before adding.
- **Adding new external ingress** — webhooks, OAuth callbacks, public POST/PUT routes outside the protected API surface. New ingress is new attack surface. Confirm: auth model (signing secret / HMAC / OAuth state), idempotency, rate-limit posture, and the failure mode if the endpoint is hit during an incident.
- **Processing user data without an explicit user action** — nightly recompute jobs over user histories, behavioural-signal aggregation, ML training inputs, batch updates triggered by upstream webhooks. Read-on-request from a logged-in user is fine; background work over user data needs confirmation because the user isn't there to consent in the moment.
- *Add project-specific stop-and-ask items here* — e.g. "touching the recommendation engine boundary," "modifying the payment intent flow," "anything that changes pricing display."
- **Adding any kind of analytics / tracking pixel / third-party script.**

---

## 5. Commit conventions

- **Conventional commits:** `feat(scope): ...`, `fix(scope): ...`, `chore: ...`, `docs: ...`, `refactor: ...`, `test: ...`. Scope is usually the package or feature area.
- **One logical change per commit.** If you're mixing concerns, split.
- **Dependencies get their own commit.** Never bundle `package.json` changes into a feature commit.
- **No `Co-Authored-By: Claude`** or any AI co-author trailer. Commit as the human maintainer; no AI authorship claims anywhere in commit metadata.
- **No "Generated with Claude Code" footers.**

The pre-push hook (`scripts/scan-ai-attribution.sh`, wired in `.husky/pre-push`) automatically aborts pushes with AI-attribution trailers.

### Review-Note trailers

When the agent is uncertain about something in a commit that warrants post-hoc human review, add a trailer:

```
feat(area): short summary (TICKET-15)

Implements the X for the Y flow.

Review-Note: integration — the upstream API rate-limit is undocumented;
re-verify behaviour once we have production traffic data.
```

Categories: `test-independence`, `logic`, `sanity`, `integration`, `security`, `perf`, freeform.

**Mirror the Review-Note in the PR description.** Trailer is for `git log --grep`; PR description is for the live reviewer.

Not every commit gets one. Only when genuine uncertainty exists.

---

## 6. Diff-reading protocol

When the agent presents a diff to the human, it MUST categorise the changes so the human can read with focus. Tags:

| Tag | Meaning | Human's duty |
|---|---|---|
| **Scaffolding** | Setup boilerplate, standard patterns, package markers | Skim |
| **Decision** | Architectural choice (naming, layout, pattern, library picked) | Read carefully; push back if wrong |
| **Logic** | Business / domain computation | Read and verify; ask "what if…" |
| **External** | Calls into third-party APIs or vendor SDKs | Read and trace the data flow |
| **Security** | Auth, scoping, PII handling, error-message sanitisation, rate limits | Read line by line |

Scaffolding is the only category that may be skimmed. Everything else, the human confirms understanding before commit.

### 6.1 Tagging-escalation rule

**When in doubt, classify upward.** Never tag a Security or External change as Scaffolding. If a chunk *might* be security-relevant or *might* touch an external system, it gets the higher classification by default.

### 6.2 Acknowledgement requirement for Security / External

The human must explicitly acknowledge they've read each Security or External chunk before commit. "Acknowledged" / "OK" / "👍" works; silence does not. This forces a real read rather than a passive skim.

---

## 7. Pre-PR review

Pre-PR review is something the **agent does on demand**, not a checklist for the human to type commands through.

### Trigger

When the human says one of: `ship it`, `ready to PR`, `open the PR`, `submit the PR for TICKET-X`, `let's submit`, `pre-PR check`, or invokes `/pre-pr` — the agent runs the full review inline. No scripts for the human to launch, no commands for the human to copy-paste.

### What the agent does, in order

1. **Run the gates.** `pnpm typecheck && pnpm lint && pnpm test` (or `pnpm preflight` which chains them). Report PASS/FAIL summary. If anything fails, stop and fix before continuing.
2. **Walk the diff.** `git diff origin/main...HEAD`. Tag every chunk Scaffolding / Decision / Logic / External / Security per §6 (apply §6.1 escalation + §6.2 acknowledgement).
3. **Pessimistic meta-check.** Spawn a fresh sub-agent (Explore type, no view of this conversation). Brief: *"Read CLAUDE.md, the relevant ADRs, and the branch diff (`git diff origin/main...HEAD`). Look for: rule violations (banned patterns, missing ADRs, missing DEPS.md entries), category/naming/scope mismatches, brittleness, things that pass the gates but a senior reviewer would flag. Report as Definitely-issue / Likely-issue / Maybe-issue / Looks-clean — file:line + quoted snippet per finding. Bias toward suspicion; assume at least three issues missed."* Resolution rule: every Definitely and Likely is fixed in the diff or has a Review-Note trailer. Maybes get a one-line decision.
4. **Acceptance-criteria mapping.** Read the ticket. For each AC bullet, point at file:line or "not satisfied".
5. **Commit hygiene + scope check.** One logical change per commit (per §5)? New deps in their own commits? Anything in the diff outside the ticket's scope?
6. **Rebase check.** `git rev-list --left-right --count origin/main...HEAD`. Flag if behind.
7. **PR description draft.** Three blocks (Summary / Test plan / Notes for reviewer). The Test plan **must** include a "Manual verification commands" subsection listing the exact actions a reviewer should run to exercise this PR's surface end-to-end. Pure-refactor PRs skip the subsection but say so explicitly.
8. **Output one consolidated handoff** inline (sections 1–7 above as one markdown document).
9. **Surface what only the human can do.** Three items, listed concretely: read the chunks tagged Decision/Logic/External/Security; run the manual verification commands; sign off with `ship it` or push back.

### What the human does

Three things, all irreducible:

1. **Read the chunks the agent flagged Decision / Logic / External / Security.** Eyes-on. The agent can't tell when its own categorisation was off — the human can.
2. **Run the manual verification commands** from the handoff. Skip only if the agent declared "no human-visible surface" (pure refactor) — and confirm that's true.
3. **Say `ship it`** (or `fix X first`).

### What happens on `ship it`

Agent does, in order:

1. Stages and commits any final changes (one logical change per commit per §5).
2. `git push`. The pre-push hook automatically runs the AI-attribution scan and aborts on any `Co-Authored-By: Claude` / Anthropic / "Generated with Claude Code" trailer.
3. Outputs the GitHub PR-creation URL inline (or runs `gh pr create` if the user explicitly OK'd that, with the description from step 7 of the handoff).

### Why this design

The agent does everything programmatic. The human does only what's irreducibly human: read the diff, exercise the surface, decide. No scripts to launch. No commands to type beyond the natural-language trigger and `ship it`.

The only invisible automation is the **pre-push hook** (`scripts/scan-ai-attribution.sh` wired via `.husky/pre-push`). One-time install per clone (run by `pnpm install` if `prepare` script is set up — see README).

---

## 8. Testing conventions

- **Split sessions for tests on non-trivial logic.** Implementation in one conversation turn; tests written in a fresh turn (or by a sub-agent) with only the spec / contract visible. Prevents rubber-stamp tests that encode the agent's own misunderstanding.
- **One behaviour per test.** No multi-assert tests testing several things.
- **Name tests for behaviour, not method.** `test_returns_recommendation_list_for_valid_user_id` not `test_get_recommendations`.
- **No test is "green" until the human has run it locally at least once.** CI passing is necessary but not sufficient.

### 8.1 Test-isolation approach

| Category | Approach | What it means |
|---|---|---|
| **Scaffolding** (config, types, package boilerplate) | None | Nothing to rubber-stamp |
| **Simple Logic** (mechanical, narrow contract) | **A** — in-turn discipline | Agent writes impl, commits, then writes tests against ONLY the contract / docstring / ADR — not the implementation file |
| **Complex Logic** (anything moat-relevant, anything with subtle correctness invariants) | **B** — sub-agent isolation | Agent writes + commits implementation. Then spawns a fresh sub-agent with **only** the contract, ticket, and CLAUDE.md — sub-agent writes tests in a genuinely separate context. Agent reviews output, commits. |
| **Pure-logic function with fully-known contract** | **C** — TDD | Tests written **before** implementation; human reviews tests; implementation written to make tests pass. |

Approach B is mandatory for anything in your project's *moat* package — the code that, if it became defensibly valuable, you'd want to productise without rewriting. Rubber-stamp tests there will hide real bugs.

### 8.2 Tools

- **Vitest** for unit + integration. Fast, ESM-native, jest-compatible API.
- **Playwright** for end-to-end (web).
- **MSW** (Mock Service Worker) for mocking external APIs in tests. Don't hit real APIs from CI.
- **React Testing Library** for component tests on web.

---

## 9. Commands (canonical)

Run exactly these (pnpm):

```bash
# Install everything
pnpm install

# Dev — root invokes per-app
pnpm dev                                  # all apps in parallel
pnpm --filter=<your-app-name> dev         # just one

# Type check (recursive across workspace packages)
pnpm typecheck

# Lint (single root flat config covers the whole monorepo)
pnpm lint
pnpm lint:fix

# Format (Prettier)
pnpm format                               # write
pnpm format:check                         # check only

# Tests (single root vitest config covers all packages)
pnpm test                                 # all tests
pnpm test <path>                          # narrow to a path

# All commit-stage gates at once
pnpm preflight                            # typecheck + lint + test
```

**Pre-push gates** run automatically via `.husky/pre-push` and don't normally need invoking by hand:

```bash
bash scripts/scan-ai-attribution.sh       # AI-attribution scan
pnpm preflight                            # full preflight
```

**One-time install per clone:**

```bash
pnpm install                              # runs the `prepare` script, which invokes `husky` to wire .husky/_/ → .husky/ hook files
```

If any command fails on a clean checkout of `main`, that's a bug — file it.

**Pre-PR review** (gates, diff walk, meta-check, PR description) is not a command in this section — it's something the agent does inline when the human says `ship it` / `ready to PR` / etc, or invokes `/pre-pr`. See §7.

### 9.5 PR description template

PR descriptions are not commit-message dumps. Three short blocks:

```
## Summary

<1–3 bullets, plain English, what changed and why>

## Test plan

- [ ] <reproduction step 1>
- [ ] <reproduction step 2>
- [ ] <edge case>

### Manual verification commands

<exact commands or UI actions a reviewer should run to exercise this PR's
surface end-to-end. If pure refactor with no surface, say so explicitly>

## Notes for reviewer

<acceptance-criteria mappings; any Review-Note trailers from commits;
anything subtle worth flagging>
```

No emoji. No AI-attribution.

---

## 10. Documentation layout

| Location | Purpose | Audience |
|---|---|---|
| `CLAUDE.md` (this file) | Contract between human and AI; conventions and rules | Team + AI |
| `apps/*/CLAUDE.md` | Package-specific rules; inherits this file | Team + AI |
| `packages/*/CLAUDE.md` | Package-specific rules; inherits this file | Team + AI |
| `docs/decisions/` | ADRs (architecture decision records), numbered, chronological | Team |
| `docs/runbooks/` | One-page-per-vendor incident references — what breaks, manual fallback, status pages, key rotation | Team |
| `DEPS.md` | One-line justification per npm dependency | Team |
| `PROJECT.md` | Product brief; what we're building, scope | Team |
| `LEARNED.md` | Sharp edges, gotchas, non-obvious behaviour — append when something costs >15 min to diagnose; the "what to do next time" file | Team + AI |
| `README.md` | How to set up + run | Anyone |
| `.personal/` *(gitignored)* | Human's private notes, scratch | Human only |

ADRs are lightweight — one page max. Format: *Status / Date / What we chose / What we rejected / Why / What would change our mind / Related.*

---

## 11. When the human asks for behaviour not in this file

If the human requests something that isn't covered here (and isn't obviously ad-hoc), propose adding a rule to this file before executing. The default is: new conventions get written down before being followed.

When the human rejects a rule, the reasoning goes in an ADR so future-the-human (and future-the-agent) know why.

---

## 12. When something feels wrong

If the agent is about to produce something it has low confidence in — architectural choice, non-obvious algorithm, security-adjacent code — it stops and surfaces the uncertainty to the human instead of generating plausible-looking but unverified code. "I don't know" is a valid answer and always preferable to a confident hallucination.

The same applies to the human. If a rule in this file is producing bad outcomes, propose changing it; don't quietly skip it.
