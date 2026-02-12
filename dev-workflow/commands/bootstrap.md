# Bootstrap Command

You are scaffolding the development methodology for a new project. This is a conversational process — you're helping someone set up their project with structured planning, DDD documentation, and a verification workflow.

## Process

### 1. Conversational Start

Ask the user about their project:

"What do you want to build? Just explain the idea to me."

Listen and ask follow-up questions to understand:
- What the project does
- Who it's for
- What problem it solves

Get enough to write a meaningful project overview (2-3 paragraphs). This will go in CLAUDE.md and is the primary orientation for any Claude instance working on the project.

### 2. Tech Stack

Ask: "Do you already have a tech stack in mind, or would you like me to suggest one?"

- **User has a stack** → Use it
- **User wants suggestions** → Recommend a stack based on the project description. Explain your reasoning. Let the user approve or modify.

### 3. Identify Packages

Every project is a monorepo. Ask: "What packages will this project have? For example: a backend API, a frontend app, a shared library?"

Help the user identify the initial set of packages. Each package is a top-level directory in the monorepo. Examples:
- `server/` — Backend API
- `client/` — Frontend app
- `shared/` — Shared types/utilities
- `infra/` — Infrastructure/deployment

It's fine to start with one package. More can be added later via `/dev:plan`.

### 4. Assign Ports and Hosts

The runtime environment provides four port/host pairs via environment variables (`$PORT1`/`$HOST1` through `$PORT4`/`$HOST4`). Each port is behind a reverse proxy at its corresponding HOST URL.

Ask: "Which packages need a port? For example, does the backend need PORT1 and the frontend need PORT2?"

Map each server-capable package to a port/host pair. Record this mapping — it goes into the CLAUDE.md Runtime Environment section. Not all packages need ports (e.g., a shared library doesn't).

### 6. Create Directory Structure

Create the following files and directories:

```
{{package_a}}/              ← empty package directories from step 3
{{package_b}}/
product/
├── DDD/
│   ├── glossary.md
│   ├── context-map.md
│   └── contexts/
│       └── .gitkeep
├── ADR/
│   └── .gitkeep
└── plans/
    ├── todo/
    │   └── .gitkeep
    └── done/
        └── .gitkeep
```

Create a `.gitkeep` in each empty package directory so they are tracked in git. Empty directories get a `.gitkeep` file.

### 7. Generate `product/DDD/glossary.md`

Create the glossary with the methodology terms pre-filled. Add project-specific domain terms as the project develops.

````markdown
# Ubiquitous Language Glossary

This glossary defines the shared vocabulary used throughout the system. All team members and documentation should use these terms consistently.

---

## Methodology Terms

### Monorepo
All project code lives in a single repository. Every package, service, library, and configuration file is versioned together. There are no multi-repo splits — one repo, one `./do` script, one source of truth.

Key properties:
- **Single repository:** All code, documentation, and infrastructure in one place
- **Unified tooling:** The `./do` script is the single entry point for all tasks across all packages
- **Atomic changes:** Cross-package changes are a single commit, not coordinated across repos

### `./do` Script
The project's single entry point for all development tasks. Self-contained, auto-installing, and the only thing a developer needs to know after `git clone`.

Key properties:
- **One-stop shop:** `./do run` starts everything, `./do check` verifies everything
- **Self-contained:** Auto-installs dependencies, checks prerequisites, handles setup
- **Minimal host requirements:** Only requires bash and the project's primary runtime
- **Running documentation:** The script IS the setup guide — reading it tells you how the project works
- **Monorepo-aware:** Orchestrates across all packages in the repo

### Package
A distinct unit within the monorepo — a service, library, frontend app, or other self-contained module. Packages live in top-level directories and are orchestrated by the `./do` script.

### Runtime Environment
The application runs behind a reverse proxy. Four port/host pairs are available via environment variables (`$PORT1`/`$HOST1` through `$PORT4`/`$HOST4`). Each port listens locally and is exposed publicly via its HOST URL.

Key properties:
- **No localhost:** Users access the application via `$HOST1`–`$HOST4`, never `localhost`
- **Port variables:** Servers bind to `$PORT1`–`$PORT4`, never hardcoded ports
- **Reverse proxy:** Each PORT has a corresponding public HOST URL with TLS termination
- **Security implications:** CORS, cookies, CSP, OAuth redirects must all use the HOST URLs

---

## Core Domain Terms

<!-- Add project-specific domain terms here -->

---

## How to Add New Terms

When introducing a new domain concept:

1. **Choose the right category** — Group related terms together. Create a new category if none fits.
2. **Write a precise definition** — Avoid ambiguity. If a term means something specific in this project that differs from common usage, call that out explicitly.
3. **List key properties** — What are the important attributes or characteristics?
4. **Note relationships** — Reference other glossary terms where relevant.
5. **Keep it current** — Update definitions when the domain model evolves. Remove terms that are no longer used.
````

### 8. Generate `product/DDD/context-map.md`

Create a starter context map. Replace the example diagram and placeholders as the project develops.

````markdown
# Bounded Context Map

This document provides an overview of all bounded contexts in the system and their relationships.

---

## Context Overview

```
<!-- Replace with an ASCII diagram of bounded contexts -->
```

---

## Contexts by Strategic Classification

### Core Domain

| Context | Purpose | Strategic Value |
|---------|---------|-----------------|
| <!-- fill in --> | <!-- fill in --> | Highest - this is the product's core value |

### Supporting Domains

| Context | Purpose | Strategic Value |
|---------|---------|-----------------|
| <!-- fill in --> | <!-- fill in --> | <!-- fill in --> |

### Infrastructure Domains

| Context | Purpose | Strategic Value |
|---------|---------|-----------------|
| <!-- fill in --> | <!-- fill in --> | <!-- fill in --> |

---

## Context Relationships

<!-- Document relationships as they emerge -->

---

## File to Context Mapping

| File | Context |
|------|---------|
| <!-- fill in --> | <!-- fill in --> |

---

## Communication Patterns

Describe how contexts communicate:
- Synchronous / Asynchronous
- In-process / Over network
- Event-driven / Direct calls
````

### 9. Generate CLAUDE.md

Create `CLAUDE.md` in the project root. Fill in the placeholders from the conversation. All other sections are used as-is.

````markdown
# Claude Guide for {{PROJECT_NAME}}

This document helps Claude instances understand and work effectively with the {{PROJECT_NAME}} codebase.

## Project Overview

{{PROJECT_DESCRIPTION}}

**Tech stack**: {{TECH_STACK}}

## Documentation Structure

```
product/
├── DDD/                          # Domain-Driven Design documentation
│   ├── glossary.md               # Ubiquitous language
│   ├── context-map.md            # Bounded contexts overview
│   └── contexts/                 # Detailed context documentation
│       └── (context files)
│
├── ADR/                          # Architecture Decision Records
│   └── (decision records)
│
└── plans/
    ├── todo/                     # Plans awaiting implementation
    └── done/                     # Completed plans (historical reference)
```

## Before Starting Work

### 1. Understand the Domain

Read in this order:
1. `product/DDD/glossary.md` - Learn the vocabulary
2. `product/DDD/context-map.md` - Understand bounded contexts
3. Relevant context file in `product/DDD/contexts/` for the area you'll modify

### 2. Check Existing Decisions

Review `product/ADR/` for architecture decisions. Don't contradict established patterns without good reason and a new ADR.

### 3. Check for Pending Plans

Look in `product/plans/todo/` for any plans awaiting implementation.

## Monorepo Structure

This is a monorepo — all code, documentation, and infrastructure lives in this single repository. Packages are orchestrated by the `./do` script.

### Packages

| Package | Path | Purpose |
|---------|------|---------|
| {{PACKAGE_NAME}} | `{{PACKAGE_PATH}}/` | {{PACKAGE_PURPOSE}} |

### Source Tree

```
# TODO: Update as source files are created
```

## The `./do` Script

The `./do` script is the project's **single entry point** for all development tasks. It is self-contained: it checks for prerequisites, auto-installs dependencies where possible, and "just works" from a fresh checkout.

```bash
./do build     # Build the project
./do lint      # Run linter
./do test      # Run tests
./do run       # Run the project locally
./do check     # Full verification (must pass before every commit)
```

**Philosophy:** Anyone should be able to `git clone` this repo, run `./do run`, and have the project running. The script handles dependency installation, environment setup, and anything else needed. It only requires minimal host dependencies (bash, and the project's primary runtime like Node). If something needs `sudo`, the script asks the user instead of failing silently.

The `./do` script is maintained by `/dev:plan` — every feature plan considers whether `./do` needs updates.

## Runtime Environment

The application runs behind a reverse proxy. Four port/host pairs are available via environment variables:

| Variable | Purpose |
|----------|---------|
| `$PORT1` / `$HOST1` | {{PORT1_PURPOSE}} |
| `$PORT2` / `$HOST2` | {{PORT2_PURPOSE}} |
| `$PORT3` / `$HOST3` | {{PORT3_PURPOSE}} |
| `$PORT4` / `$HOST4` | {{PORT4_PURPOSE}} |

**Critical rules for all server-side and web code:**

- **Port binding**: Server processes MUST bind to `$PORT1`–`$PORT4`, never hardcoded ports.
- **No localhost**: The application is accessed via the public `$HOST1`–`$HOST4` URLs, never `localhost`. This affects:
  - CORS origins — must allow the HOST URLs
  - Cookie domains — must match the HOST domain
  - CSP headers — must reference HOST URLs
  - API base URLs — frontend must use the HOST URL of the API, not `localhost:PORT`
  - WebSocket endpoints — must use `wss://` with the HOST URL
  - OAuth/redirect URLs — must use HOST URLs for callbacks
- **`./do run`** must start all services on the correct PORT variables and configure them with the HOST URLs.

## Planning and Implementation

### Creating Plans

Use `/dev:plan` to create feature plans. Plans live in `product/plans/todo/` and include:
- Acceptance test scaffolds (always Phase 1)
- Phased implementation following ATDD red-green cycles
- DDD documentation updates (glossary, contexts, context map)
- Architecture decision records for significant decisions
- `./do` script updates as needed
- Verification steps for each phase

Plans are reviewed by four specialized sub-agent reviewers (feasibility, ATDD coverage, architecture, decision completeness) before being presented for approval.

### Implementing Plans

Use `/dev:implement` to execute plans. The implementation process:
1. Reads the plan and breaks it into phases
2. Delegates each phase to a sub-agent
3. Runs `./do check` after each phase
4. Creates a focused commit per phase
5. Moves completed plans to `product/plans/done/`

### Verification Rule

**`./do check` must pass before every commit.** No exceptions. No partial substitutes (`./do lint` alone, `./do build` alone, etc.).

### Acceptance Test Driven Development

Every feature follows ATDD:
1. Acceptance test scaffolds are created first (skipped)
2. Each phase: unskip test → verify it fails → implement code → verify it passes
3. `./do check` always runs all acceptance tests

## Pre-change Questions

Before making significant changes, ask:
1. Does this align with existing ADRs?
2. Which bounded context does this belong to?
3. Should this be a new plan document first?
4. Have I read the existing code I'm modifying?
5. Does this maintain the simplicity principle?
````

### 10. Create Starter `./do` Script

Create `./do` in the project root with this content. All command bodies are placeholders — `/dev:plan` will fill these in as part of the first feature plan.

Make it executable: `chmod +x ./do`

````bash
#!/usr/bin/env bash
set -euo pipefail

# ============================================================
# ./do — Development task runner
#
# This script is the single entry point for all development
# tasks. It is self-contained: it checks for prerequisites,
# auto-installs dependencies where possible, and "just works"
# when run from a fresh checkout.
#
# Philosophy:
#   - One-stop shop: ./do run starts everything
#   - Self-contained: auto-installs what it can
#   - Minimal host dependencies: only requires bash and basic tools
#   - Running documentation: the script IS the setup guide
#
# Usage: ./do <command>
#
# Commands:
#   build    Build the project
#   lint     Run linter
#   test     Run tests
#   run      Run the project locally
#   check    Full verification (must pass before every commit)
# ============================================================

# --- Color output helpers ---

_green()  { printf '\033[0;32m%s\033[0m\n' "$*"; }
_yellow() { printf '\033[0;33m%s\033[0m\n' "$*"; }
_red()    { printf '\033[0;31m%s\033[0m\n' "$*"; }
_bold()   { printf '\033[1m%s\033[0m\n' "$*"; }

# --- Dependency checks ---

require_cmd() {
  if ! command -v "$1" &>/dev/null; then
    _red "Required command not found: $1"
    if [ -n "${2:-}" ]; then
      _yellow "Install it: $2"
    fi
    exit 1
  fi
}

# --- Commands ---

build() {
  _bold "build"
  _yellow "Not implemented yet — ./do build will be configured by /dev:plan"
}

lint() {
  _bold "lint"
  _yellow "Not implemented yet — ./do lint will be configured by /dev:plan"
}

test_cmd() {
  _bold "test"
  _yellow "Not implemented yet — ./do test will be configured by /dev:plan"
}

run_cmd() {
  _bold "run"
  _yellow "Not implemented yet — ./do run will be configured by /dev:plan"
}

check() {
  _bold "check"
  _yellow "Not implemented yet — ./do check will be configured by /dev:plan"
}

case "${1:-}" in
  build)  build ;;
  lint)   lint ;;
  test)   test_cmd ;;
  run)    run_cmd ;;
  check)  check ;;
  *)
    _bold "Usage: ./do <command>"
    echo ""
    echo "Commands:"
    echo "  build    Build the project"
    echo "  lint     Run linter"
    echo "  test     Run tests"
    echo "  run      Run the project locally"
    echo "  check    Full verification (must pass before every commit)"
    exit 1
    ;;
esac
````

### 11. Initialize Git Repository

Initialize a git repository and create the initial commit:

1. Run `git init`
2. Stage all created files with `git add`
3. Create an initial commit with a message like: `Bootstrap project with dev-workflow methodology`

### 12. Summary

Report what was created:

```
Created:
- CLAUDE.md — Project guide for Claude instances
- ./do — Development task runner
- {{package_a}}/ — Package directory
- {{package_b}}/ — Package directory
- product/DDD/glossary.md — Domain glossary (with methodology terms)
- product/DDD/context-map.md — Bounded context map (starter)
- product/DDD/contexts/.gitkeep
- product/ADR/.gitkeep
- product/plans/todo/.gitkeep
- product/plans/done/.gitkeep
- Initialized git repository with initial commit
```

Then explain the workflow:

"Your project is set up as a monorepo. Here's how to use it:

1. **`/dev:plan`** — Create a feature plan. Describe what you want to build, and I'll create a phased plan with acceptance tests, DDD docs, ADRs, and implementation steps.
2. **`/dev:implement`** — Execute a plan. I'll work through each phase, verify with `./do check`, and commit.

Start by describing your first feature and I'll help you plan it."

## Rules

1. **This command is for new, empty projects.** Don't run it on an existing project with established structure.
2. **Every project is a monorepo.** All code lives in one repository. Identify packages early (step 3) and create their directories.
3. **Be conversational.** Ask questions, understand the project, help the user think through their idea.
4. **Don't over-scaffold.** Create the structure and starters, not the implementation. The project grows through `/dev:plan` and `/dev:implement`.
5. **Empty directories get `.gitkeep`.** So they can be tracked in git.
6. **File formats are embedded above.** Do not look for external template files — the exact content for each generated file is defined in steps 7–10.
7. **Always create a git repo and initial commit.** The project should be version-controlled from the start.
