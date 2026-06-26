# Dev Workflow Plugin

A structured development methodology for Claude Code projects. Provides DDD documentation, architecture decision records, phased feature plans, and orchestrated implementation with mandatory verification.

## The Methodology

Every project using this plugin follows the same workflow:

1. **Plan** — Describe what you want to build. `/plan` creates a phased implementation plan that includes DDD documentation updates, architecture decision records, and code changes.
2. **Implement** — `/implement` executes the plan phase by phase, delegating work to sub-agents, verifying with `./do check` after each phase, and creating focused commits.
3. **Verify** — `./do check` must pass before every commit. No exceptions.

### Domain-Driven Design

Projects maintain living DDD documentation:
- **Glossary** (`product/DDD/glossary.md`) — Shared vocabulary for the domain
- **Context Map** (`product/DDD/context-map.md`) — Bounded contexts and their relationships
- **Context Documents** (`product/DDD/contexts/`) — Detailed documentation per bounded context

DDD updates are included as phases within feature plans, not as separate manual steps.

### Architecture Decision Records

Significant technical decisions are recorded in `product/ADR/`. When `/plan` identifies an architectural decision, it includes an ADR phase in the plan. ADRs capture the context, decision, rationale, and consequences.

### Phased Plans

Plans live in `product/plans/todo/` (pending) and `product/plans/done/` (completed). Each plan breaks work into phases with specific file changes and verification steps.

## Commands

### `/bootstrap`

Scaffold the methodology for a new project. Conversational — asks what you want to build, helps choose a tech stack, then creates:
- `CLAUDE.md` — Project guide for Claude instances
- `./do` — Task runner with placeholder commands
- `product/DDD/` — Starter glossary and context map
- `product/ADR/` — Empty, ready for decisions
- `product/plans/` — Empty, ready for plans

### `/plan`

Create a feature plan. Process:
1. Asks clarifying questions about the feature
2. Explores the codebase (CLAUDE.md, DDD docs, ADRs, source files)
3. Presents technical decisions to the user for approval
4. Identifies DDD and ADR implications
5. For frontend features, decides the design direction with the user and captures a UI/UX Spec (design system, screens, states, responsive, accessibility); for Figma-sourced designs, resolves the Figma mapping (logical pieces ↔ nodes per breakpoint, plus an ignore list) that `/figma-refresh-plan` consumes
6. Writes a phased plan to `product/plans/todo/`
7. Reviews the plan with the user until approved

The plan document is the only output. No code changes, no DDD edits, no ADR creation.

### `/implement`

Execute a plan with sub-agent orchestration. The orchestrator:
1. Reads the plan and creates a task list
2. For each phase: launches an implementation sub-agent, then a verification sub-agent
3. If verification fails with code errors: launches a fix sub-agent, re-verifies, repeats
4. If verification fails with infrastructure errors: stops and asks the user
5. For frontend phases: after `./do check` passes, runs a visual review loop — a UI review sub-agent renders the app with Playwright and critiques it against the plan's UI/UX Spec, then a fix sub-agent resolves issues, until the UI passes (or after 3 cycles, records remaining issues)
6. Creates a focused commit per phase
7. Moves the completed plan to `product/plans/done/`

The orchestrator never implements, fixes, verifies, or reviews UI directly — it only delegates and commits.

Frontend phases are built in **frontend mode**: the implementation sub-agent invokes the `frontend-design` skill, implements every UI state from the spec, and applies design judgment to unspecified visual details. This requires the `frontend-design` skill and the Playwright MCP server to be available.

### `/figma-refresh-plan`

Keep a Figma-sourced design and the code in sync. The command:
1. Pulls the current design via the Figma MCP (Code Connect for component identity, design context, variables)
2. Maintains a per-piece **status ledger** (`product/design/<plan-slug>/status.json`) tracking which pieces — tokens, primitives, components, layouts, views per breakpoint — are implemented against the current design and which have drifted
3. Writes design files (`tokens.json` in W3C DTCG format, `context.md`, `source.md`)
4. Commits a dedicated design snapshot (git history is the diff substrate)

5. Writes the **UI plan** from the ledger to `product/plans/todo/` — phases for every open piece (`to-implement` / `to-review`) in dependency order. First run: all open pieces; later runs: only what changed. This is the single source of UI plans (first build and rework alike); the only no-plan case is nothing open.

**Order: `/plan` → `/figma-refresh-plan` → `/implement`.** `/plan` resolves the Figma mapping + design-system spec (naming convention, responsive rules) and plans the non-UI technical groundwork — it does **not** write per-piece UI phases. `/figma-refresh-plan` then pulls (using that mapping), maintains the ledger, and writes the UI plan. It is **not** an auto-step inside `/implement`, and never a post-implement cleanup — run it before `/implement`, and again only when the Figma design changes. Requires the Figma MCP and a Figma plan with Code Connect (Org/Enterprise + Dev seat) for reliable component identity.

**Adopting an in-progress project:** if you already have UI built (e.g. on an older flow), run `/plan` once as a base-straightening pass — it resolves the mapping, audits the existing code across all five levels (design system, primitives, components, layouts, views), classifies each piece (matches / refactor / rebuild / missing), and records the "matches" list. `/figma-refresh-plan` then seeds the ledger from that list (so already-correct work isn't re-flagged) and writes the UI remediation plan for the rest. After that one-time pass the project is on the normal lifecycle.

### `/figma-accept`

Record **human acceptance** of implemented UI pieces. Building a piece and passing the automated visual review is not the same as a tester signing off, so acceptance is tracked as a separate dimension in the ledger (`acceptedHash` / `acceptedBy` / `acceptedAt`). A piece is `accepted` only when a tester signed off on exactly what's built; any later design change that triggers a rebuild re-opens acceptance automatically.

Two channels: a CLI command (`/figma-accept <piece>`, `--all`, or `--from-pr <n>` to sync a PR's acceptance checklist) and the acceptance checklist that `/implement` puts in the PR body. Acceptance is **soft** — it never blocks merges or plan completion; unsigned pieces simply stay visible as `awaiting-acceptance` until accepted.

## Skills

### `verification-required`

Enforces `./do check` before any commit:
- `./do check` is mandatory, no exceptions
- Infrastructure failures = full stop, ask user for help
- Code failures = fix and re-verify until passing
- No partial verification (`./do lint` alone is not enough)

### `plan-before-code`

Suggests `/plan` before any implementation work:
- Always offers planning first, regardless of change size
- The user decides whether to plan — they can decline
- Planning includes DDD, ADR, and implementation phases

## The `./do` Script

Every project uses a `./do` script as its task runner:

| Command | Purpose |
|---------|---------|
| `./do build` | Build the project |
| `./do lint` | Run linter |
| `./do test` | Run tests |
| `./do run` | Run the project locally |
| `./do check` | Full verification — must pass before every commit |

`./do check` is the critical command. It defines whatever verification the project needs for full confidence. It is NOT simply "lint + build + test" — projects define their own pipeline (which may include deployment, integration tests, or other steps).

## Quick Start

1. Install the plugin via the marketplace
2. Run `/bootstrap` in a new project directory
3. Describe your first feature and run `/plan`
4. Fill in `./do` commands as your project develops
5. Run `/implement` to execute your plan

## Templates

The `templates/` directory contains reference formats for all generated files:

| Template | Used By |
|----------|---------|
| `CLAUDE.md.template` | `/bootstrap` |
| `glossary.md.template` | `/bootstrap` |
| `context-map.md.template` | `/bootstrap` |
| `context.md.template` | `/implement` (when creating bounded context docs from plan phases) |
| `adr.md.template` | `/implement` (when creating ADRs from plan phases) |
| `plan.md.template` | `/plan` (plan document format) |
| `do.template` | `/bootstrap` |
