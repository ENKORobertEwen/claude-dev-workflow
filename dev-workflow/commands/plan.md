# Plan Command

You are creating a feature plan. Plans document what will be built, how it will be implemented phase by phase, and what DDD/ADR work is needed. The plan is a document — you do NOT execute any changes.

The plan must be so detailed and decision-complete that a "code monkey" implementer can execute it mechanically without making any technical or architectural judgments.

## Input

The user describes what they want to build. This may be a feature request, a bug fix, a refactor, or any other change.

If the user provides a file path, read it for context but still go through the full planning process.

## Process

### 1. Understand the Feature

Ask clarifying questions about scope, requirements, and constraints. Dig into what the user really needs. Don't assume — ask.

- What problem does this solve?
- Who is affected?
- What are the boundaries of this change?
- Are there constraints (performance, compatibility, etc.)?
- What are the acceptance criteria? (Get explicit, testable criteria from the user)

Keep asking until you have a clear picture. It's fine to ask multiple rounds of questions.

### 2. Explore the Codebase

Read and understand the existing code before planning changes:

1. Read `CLAUDE.md` for project orientation
2. Read `product/DDD/glossary.md` for domain vocabulary
3. Read `product/DDD/context-map.md` for bounded context overview
4. Read relevant bounded context docs in `product/DDD/contexts/`
5. Read existing ADRs in `product/ADR/` that relate to the feature
6. Read the `./do` script to understand current commands and dependencies
7. Read the source files that will be affected
8. Check `product/plans/done/` for similar past work

### 3. Make Technical Decisions with the User

For every significant decision the plan involves, present it to the user for approval. Don't decide alone — propose options and explain trade-offs:

- "For X, I'd suggest approach A because [reason]. We could also do B [reason] or C [reason]. What do you think?"
- Architecture choices, library choices, data model decisions, API design — all should be presented
- The user is the overall authority. Every decision should be explicitly approved before it goes into the plan.

**The goal is zero ambiguity.** After this step, every technical choice should be resolved. The implementer should never have to decide between approaches, pick a library, choose a pattern, or interpret vague instructions.

Continue until all significant decisions are resolved.

### 4. Identify DDD Implications

If the feature introduces new domain concepts, the plan should include phases for:

- **Glossary updates** — New terms, updated definitions
- **New bounded context docs** — If a new context is needed, create `product/DDD/contexts/{name}.md`
- **Context map updates** — New contexts, new relationships, updated diagrams

These are documented as phases in the plan. They are NOT executed now.

### 5. Identify ADR Implications

If the feature involves an architectural decision (one that was discussed and decided with the user in step 3), the plan should include a phase for creating an ADR in `product/ADR/`.

The ADR captures the decision, rationale, and consequences. It is documented as a phase in the plan. It is NOT created now.

### 6. Identify `./do` Script Changes

The `./do` script is the project's single entry point for all development tasks. Check whether the plan requires changes to `./do`:

- **New commands** — Does the feature need a new `./do` subcommand?
- **Updated commands** — Do existing commands (build, lint, test, run, check) need modification?
- **New dependencies** — Does the feature introduce dependencies that `./do` should auto-install?
- **`./do check` updates** — Must `./do check` be updated to include new acceptance tests or verification steps?

If `./do` changes are needed, include them as an explicit phase in the plan. The `./do` script must always remain self-contained: it auto-installs dependencies, checks for prerequisites, and "just works" when run from a fresh checkout.

### 7. Define Acceptance Tests

This is a critical step. The plan follows **Acceptance Test Driven Development (ATDD)**:

1. Extract each acceptance criterion from step 1 into a concrete, testable scenario.
2. Each acceptance criterion gets a corresponding acceptance test.
3. Acceptance tests should test from the outside-in — through the application's public interfaces (API endpoints, CLI commands, UI interactions). Use real or in-process infrastructure where practical; mock only external services.

The acceptance tests are structured into the plan as follows:

- **Phase 1 is always: Create acceptance test scaffolds.** All tests are written as skipped/pending stubs. Each stub has a clear name describing the acceptance criterion it validates. After this phase, `./do check` passes (skipped tests don't fail).
- **Each subsequent implementation phase** follows this cycle:
  1. Unskip and implement the acceptance test(s) for this phase
  2. Verify the test fails (red)
  3. Implement the production code to make the test pass
  4. Verify `./do check` passes (green)

The acceptance test passing is the definition of "phase complete."

### 8. Determine Next Plan Number

Scan both directories for the highest existing plan number:
- `product/plans/todo/`
- `product/plans/done/`

The new plan gets the next number (e.g., if highest is 004, new plan is 005).

### 9. Write the Plan

Create the plan file at `product/plans/todo/XXX-PLAN-FEATURE-NAME.md`.

Reference the plan template at `templates/plan.md.template` for the expected format. The plan must include:

- **Overview** — What and why
- **Acceptance Criteria** — Complete list with test mapping
- **Phase 1: Acceptance Test Scaffolds** — Always first: create all tests as skipped stubs
- **Subsequent Phases** — Each with: acceptance test implementation (red), production code (green), `./do check` verification
- **DDD phases** — Glossary updates, context docs, context map updates (as applicable)
- **ADR phases** — Architecture decision records (as applicable)
- **`./do` script phases** — Any changes to the task runner (as applicable)
- **Files Summary** — Complete table of all files touched
- **End-to-End Verification** — Including `./do check`
- **Update Considerations** — Config, storage, dependencies, migration, backwards compatibility

Order phases logically. DDD/ADR phases typically come before or alongside the code phases they document. The acceptance test scaffold phase is always first.

**Every instruction in the plan must be specific and unambiguous.** Don't write "implement appropriate error handling" — write "return HTTP 422 with body `{ "error": "invalid_email", "message": "Email must contain @" }` when the email field fails validation." The implementer should be able to work mechanically.

The plan should reflect all the decisions made with the user in step 3.

### 10. Review Plan via Sub-Agents

Before presenting the plan to the user, run an automated review using four specialized sub-agents in parallel:

#### Reviewer 1: Feasibility

Reads the plan against the actual codebase. Checks:
- Are file paths correct and do referenced files exist?
- Are dependencies available or properly specified for installation?
- Is the phase ordering correct (no phase depends on work from a later phase)?
- Are there missing steps (migrations, config changes, package installs)?
- Will `./do check` actually pass after each phase?

#### Reviewer 2: ATDD Coverage

Checks:
- Does every acceptance criterion have a corresponding acceptance test?
- Are all acceptance tests created as skipped stubs in Phase 1?
- Does each implementation phase follow the red-green cycle (implement test → verify failure → implement code → verify pass)?
- Are acceptance tests testing from the outside-in through public interfaces?
- Is `./do check` updated to include the acceptance tests?

#### Reviewer 3: Architecture & DDD

Checks:
- Are glossary updates needed that weren't identified?
- Does the plan violate any existing ADRs?
- Are bounded context boundaries respected?
- Should a new ADR be created for any decisions?
- Are the `./do` script changes consistent with its philosophy (self-contained, auto-installing)?

#### Reviewer 4: Decision Completeness

This is the most critical reviewer. Checks:
- Can an implementer execute every phase mechanically, without making any technical or architectural judgment calls?
- Are there vague instructions? ("choose an appropriate X", "handle as needed", "implement error handling")
- Are all library/package choices explicit with versions?
- Are all data formats, API contracts, and schemas fully specified?
- Are error handling strategies explicit for each case?
- Are naming conventions specified (file names, function names, variable names)?
- Would two different implementers produce substantially the same result from this plan?

**Process:**
1. Launch all four reviewers in parallel as sub-agents. Provide each with the full plan text and the relevant codebase context.
2. Collect all findings.
3. Synthesize: resolve each finding by updating the plan.
4. Run one final holistic review sub-agent with fresh eyes on the updated plan. This reviewer checks the whole plan end-to-end, including that the fixes from step 3 didn't introduce new issues.
5. Apply any final fixes.

### 11. Final Review with User

Present the complete, reviewed plan to the user. Include a summary of what the review process found and fixed. Iterate until they approve:

- Walk through each phase briefly
- Highlight key decisions and their rationale
- Highlight the acceptance test strategy
- Note any `./do` script changes
- Ask if anything needs to change

The plan is ready when the user approves it.

### 12. Commit and Push

Once the user approves the plan:

1. Stage the plan file: `git add product/plans/todo/XXX-PLAN-FEATURE-NAME.md`
2. Commit: `Plan: XXX — Feature Name`
3. Push to main: `git push origin main`

The plan is now on main and ready for `/dev:implement` to pick up.

## Critical Rules

1. **The `/dev:plan` command writes the plan document, commits, and pushes.** It does NOT execute any code changes — no code, no DDD edits, no ADR creation. All of that happens when `/dev:implement` executes the plan.

2. **Every significant decision goes through the user.** Don't make architectural choices unilaterally. Present options, explain trade-offs, get approval.

3. **DDD and ADR work are plan phases, not separate commands.** If a feature needs glossary updates or a new ADR, those appear as phases within the plan alongside the code changes.

4. **Acceptance tests come first.** Phase 1 is always acceptance test scaffolds. Every subsequent phase follows the red-green ATDD cycle.

5. **The `./do` script is always maintained.** Every plan must consider whether `./do` needs updates and include them as explicit phases.

6. **The plan must be decision-complete.** An implementer must be able to execute every phase mechanically. No ambiguity, no judgment calls, no unresolved choices.

7. **The plan is reviewed before the user sees it.** Four specialized sub-agent reviewers check feasibility, ATDD coverage, architecture, and decision completeness. Findings are fixed before presenting to the user.

8. **Plans reference the template format.** See `templates/plan.md.template` for the expected structure.

9. **Plans go in `product/plans/todo/`.** After implementation via `/dev:implement`, they are moved to `product/plans/done/`.

10. **Always commit and push.** The approved plan is committed to main and pushed immediately.
