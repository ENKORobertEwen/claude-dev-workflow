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
5. Writes a phased plan to `product/plans/todo/`
6. Reviews the plan with the user until approved

The plan document is the only output. No code changes, no DDD edits, no ADR creation.

### `/implement`

Execute a plan with sub-agent orchestration. The orchestrator:
1. Reads the plan and creates a task list
2. For each phase: launches an implementation sub-agent, then a verification sub-agent
3. If verification fails with code errors: launches a fix sub-agent, re-verifies, repeats
4. If verification fails with infrastructure errors: stops and asks the user
5. Creates a focused commit per phase
6. Moves the completed plan to `product/plans/done/`

The orchestrator never implements, fixes, or verifies directly — it only delegates and commits.

## Skills

### `notify-user`

Sends push notifications via [Bark](https://github.com/Finb/Bark) at terminal states:
- Implementation complete (all phases done, PR created)
- Verification blocked (fix cycles exhausted)
- Infrastructure failure (execution stopped)
- Plan ready for review (after sub-agent review completes)

Does NOT notify for routine progress or individual phase completions. Configure the key via `BARK_KEY` env var (default: `ykExqiREfYAbHToaf9cK5X`).

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
