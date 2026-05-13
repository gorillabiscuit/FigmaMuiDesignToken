# claude-project-starter

A pnpm-monorepo starter that bakes in the working contract, doc layout, and
tooling that's already proven across multiple projects. Designed so a new
project can be productive with Claude Code from the first session — the
guardrails are pre-wired, the conventions are pre-decided, and the
project-specific scaffolding is clearly marked.

**What this gives you on day one:**

- `CLAUDE.md` — the working contract between you and the AI agent. Banned
  patterns, stop-and-ask gates, commit conventions, pre-PR process, testing
  approach (Approach A/B/C with sub-agent isolation for moat-relevant code).
- ADR pattern (`docs/decisions/`) + lightweight runbook pattern
  (`docs/runbooks/`).
- `LEARNED.md`, `DEPS.md`, `ROADMAP.md`, `PROJECT.md` skeletons — each
  with the "why this exists / when to update" prose intact.
- AI-attribution pre-push scanner (`scripts/scan-ai-attribution.sh`) wired
  into `.husky/pre-push` so AI-co-author trailers can never reach a PR.
- `/pre-pr` Claude Code slash command that runs the full §7 pre-PR review
  inline (gates, diff walk, sub-agent meta-check, AC mapping, commit
  hygiene, rebase status, PR-description draft).
- TypeScript strict baseline (`noUncheckedIndexedAccess`,
  `exactOptionalPropertyTypes`, the works).
- ESLint flat config + Prettier + lint-staged + husky pre-commit /
  pre-push.
- Vitest config that picks up `*.test.ts` in `apps/` and `packages/`.

**What this does NOT give you:**

- A specific framework (Next.js, Remix, etc) — add per project.
- A specific database / ORM / auth provider — pick per project and ADR
  the choice.
- Application code, schema, routes — empty `apps/web/` and
  `packages/shared/` placeholders ship with `CLAUDE.md` overlays and
  nothing else.

---

## How to use it

```bash
# 1. Clone or template-clone into a new project directory
gh repo create my-new-project --template gorillabiscuit/claude-project-starter --private --clone
# OR:
git clone git@github.com:gorillabiscuit/claude-project-starter.git my-new-project
cd my-new-project
rm -rf .git && git init

# 2. Edit project identity
#    - CLAUDE.md  §1 "Project identity" — name, phase, tracker
#    - PROJECT.md — replace skeleton with your product brief
#    - package.json `name` field — your project name
#    - docs/decisions/0000-architecture-overview.md — your macro shape

# 3. Install deps and run prepare (sets up husky hooks)
pnpm install

# 4. Verify gates work on the empty scaffolding
pnpm preflight    # typecheck + lint + test (will exit clean — nothing to check yet)

# 5. Open with Claude Code and start with KICKOFF.md
```

---

## Layout

```
.
├── CLAUDE.md              The working contract. Read end-to-end before any work.
├── KICKOFF.md             What the first session should do; remove once it has.
├── PROJECT.md             Product brief skeleton. Fill before writing code.
├── DEPS.md                Per-dependency justification, one line each.
├── LEARNED.md             Sharp-edges journal — append when something costs >15 min.
├── README.md              You are here.
│
├── docs/
│   ├── ROADMAP.md         Sequenced milestone view; complements PROJECT.md.
│   ├── decisions/
│   │   ├── README.md      ADR index + format reference.
│   │   ├── _template.md   Empty ADR — copy this when adding one.
│   │   ├── QUEUE.md       Strawman decisions awaiting human review.
│   │   └── 0000-architecture-overview.md  Macro shape (draft).
│   └── runbooks/
│       └── README.md      Per-vendor incident reference pattern.
│
├── .claude/
│   └── commands/
│       └── pre-pr.md      `/pre-pr` slash command — full §7 inline.
│
├── .husky/
│   ├── pre-commit         lint-staged
│   └── pre-push           AI-attribution scan + preflight
│
├── scripts/
│   └── scan-ai-attribution.sh  Pre-push hook tool.
│
├── apps/
│   └── web/
│       └── CLAUDE.md      Per-package rules overlay (skeleton).
│
├── packages/
│   └── shared/
│       └── CLAUDE.md      Platform-agnostic package overlay (skeleton).
│
├── eslint.config.mjs      Flat config. Banned-pattern rules enforced.
├── tsconfig.base.json     Strict TypeScript baseline.
├── vitest.config.ts       Workspace test runner.
├── .prettierrc.json
├── .lintstagedrc.json
├── .gitignore
├── package.json           Root scripts: preflight / typecheck / lint / test / format.
└── pnpm-workspace.yaml
```

---

## Customisation map

When you adapt this for a new project, here's where the boilerplate ends and
the project-specific content begins:

| File | What to keep | What to replace |
|---|---|---|
| `CLAUDE.md` | §2–§12 (conventions, banned patterns, pre-PR, testing, etc) | §1 "Project identity" — replace with your name, phase, tracker, repo structure |
| `apps/web/CLAUDE.md` | Pattern + "imports allowed/forbidden" structure | Specific package imports for your stack |
| `packages/shared/CLAUDE.md` | Platform-agnostic rule + banned patterns | Project-specific anti-patterns if any |
| `PROJECT.md` | Section headings | Everything below the headings |
| `docs/decisions/0000-architecture-overview.md` | The ADR-0000 structure | The ASCII diagram + every choice |
| `docs/decisions/QUEUE.md` | The intro prose explaining the queue | Empty until you have pending stack-selection ADRs |
| `package.json` | Scripts block + devDependencies | `name` field |
| `eslint.config.mjs` | All rule blocks | The `packages/shared` rule may need to point at your platform-agnostic package, if any |

Everything else is generic and can stay verbatim.

---

## Why these conventions?

A separate doc describes the rationale for each rule in CLAUDE.md (e.g. why
`==` is banned, why ADRs follow this specific format, why tests use
Approach B for moat code). That history isn't here yet — for now,
`CLAUDE.md` itself has the reasoning inline as comments where it matters.
