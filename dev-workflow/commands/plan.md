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

### 7. Consider Runtime Environment

The application runs behind a reverse proxy. Four port/host pairs are available via environment variables:

| Variable | Purpose |
|----------|---------|
| `$PORT1` / `$HOST1` | First available port and its public URL |
| `$PORT2` / `$HOST2` | Second available port and its public URL |
| `$PORT3` / `$HOST3` | Third available port and its public URL |
| `$PORT4` / `$HOST4` | Fourth available port and its public URL |

**Critical implications for any server-side or web feature:**

- **Port binding**: Server processes MUST bind to `$PORT1`–`$PORT4`, never hardcoded ports. The plan must specify which package uses which port variable.
- **No localhost**: The application is accessed via the public `$HOST1`–`$HOST4` URLs, never `localhost`. This affects:
  - **Framework allowed hosts**: Most frameworks block requests from unrecognized hostnames. Vite has `server.allowedHosts`, Django has `ALLOWED_HOSTS`, Next.js has `hostname`, webpack-dev-server has `allowedHosts`. These MUST include the HOST variable values or be set to allow all. **This is the most common source of "app runs but can't be accessed" bugs.**
  - **CORS origins**: Must allow `$HOST1`–`$HOST4`, not `localhost`
  - **Cookie domains**: Must be set for the HOST domain, not `localhost`
  - **CSP headers**: Must reference the HOST URLs
  - **API base URLs**: Frontend must use `$HOST2` (or whichever host the API is on), not `localhost:PORT`
  - **WebSocket endpoints**: Must use the HOST URL with `wss://`
  - **OAuth/redirect URLs**: Must use HOST URLs for callbacks
- **`./do run`** must start all services on the correct PORT variables and configure them with the correct HOST URLs.

If the plan involves any server, API, or web feature, include explicit port/host assignments and ensure all cross-origin configuration uses the HOST variables.

### 8. Consider UI/UX Design (frontend features)

If the feature has a user-facing frontend, design quality is a first-class concern — not something to leave to the implementer's defaults. Decide the design WITH the user, the same way you decide architecture in step 3, and capture the result in the plan's UI/UX Spec (see step 11 format).

**The user does NOT need to bring a design** — but they may, and the plan supports either path equally. Ask which source the design comes from:

- **A) User-supplied via Figma** — The user provides a Figma file/URL. Your job here is to resolve the **mapping**, not to pull pixels. Figma's node tree carries no semantics (which frames are the same screen at different breakpoints, which are flow/annotation frames, which layers are primitives), so resolve it once, with the user:
  1. Use the Figma MCP to inspect structure: `get_code_connect_map` (authoritative Figma→code identity for mapped components), `get_metadata` (pages/frames with names + sizes — size hints the breakpoint), `get_variable_defs` (tokens), and `get_screenshot` to understand visually.
  2. **Propose** a mapping table — logical screens × breakpoint (desktop/tablet/mobile), primitives/components (from Code Connect where mapped), and an explicit **ignore list** for flow/annotation frames — marking anything uncertain (e.g. "these buttons aren't Figma components — treat as primitives?").
  3. The user confirms/corrects. Record the confirmed mapping + the Figma URL in the UI/UX Spec.

  Do **not** pull the design or enumerate per-pixel frontend phases here. Pulling the design, the per-piece status ledger, and the (re)work plan are produced by `/dev:figma-refresh-plan`, which consumes this mapping. Best for high-fidelity / brand-critical UI and multi-screen consistency.
- **B) User-supplied, other references** — Screenshots, an existing app look, or a description the user gives. Capture these in the UI/UX Spec references.
- **C) Claude-generated** — The user has no design. You *propose* a direction using the `frontend-design` skill, exactly like the architecture decisions in step 3, and the user approves or adjusts. Best for greenfield/speed; the visual-review loop in `/dev:implement` then refines it.

These can be mixed (e.g. Claude proposes the overall direction, the user supplies Figma for the key screens). The point is that *a* direction is decided and written into the UI/UX Spec — not that the user delivers a polished spec. **Never block planning because the user hasn't provided a design.**

Resolve, with the user (proposing defaults wherever they have no preference):

- **Design direction** — Overall visual style and tone. Invoke the `frontend-design` skill (if available) to help shape an intentional aesthetic rather than templated defaults. Reference any existing design language in the app so new UI stays consistent.
- **Reference designs** — Figma files, screenshots, or existing screens to match. If the user has a Figma URL, note it; the implementer can pull design context from it.
- **Design system & tokens** — Component library and/or token set (colors, typography, spacing, radii). Pick explicitly; don't leave the implementer to invent one. For Figma-sourced designs the tokens come from Figma as discrete named styles, so also capture:
  - **Naming convention** — the pattern Figma uses (e.g. text `{role}-{size}-{weight}` → `body-lg-regular`; spacing `space-{step}`), so families and their size steps are machine-recognizable (`body-*-regular` is one family; lg/md/sm are steps).
  - **Responsive behavior** per axis/family — does typography and spacing scale **fluid** or **stepped** across breakpoints? (Strategy is per-project; ask and record it.) Define the rule concretely:
    - *Stepped:* which named token applies at each breakpoint (e.g. `body-regular` → desktop `body-lg-regular`, tablet `body-md-regular`, mobile `body-sm-regular`).
    - *Fluid:* which family steps define min/max and over which viewport range — `figma-refresh-plan` derives a `clamp()` token from this.
  - **Breakpoint values** (e.g. 480/768/1024) and **container max-widths** — part of the design system, not implicit.
- **Screens & components** — The list of screens/views and the key components each needs.
- **States** — For every screen/component, the states that must be implemented: loading, empty, error, success, disabled. Missing states are the most common UI-quality gap.
- **Responsive behavior** — Target breakpoints and how the layout adapts (e.g. desktop ≥1024px, mobile ≤480px).
- **Accessibility** — Keyboard navigation, visible focus states, contrast, labelled controls. Note any specific requirements.

The goal is that visual taste is fixed in the plan. The implementer should apply judgment only on small unspecified visual details — never on the overall design direction.

**Order frontend phases bottom-up.** The **first** frontend phase always establishes the **design system** (colors, spacing, text styles, radii) as the project's theme / token layer — nothing else can reference tokens until they exist. The **second** builds the **primitives** and exposes them on a browseable preview (Storybook or a dev route) for visual checking. Then: components → layouts → views. **Components, layouts, and views are all responsive** — map and build each across its breakpoints (desktop/tablet/mobile), not just views. This is the same dependency order `/dev:figma-refresh-plan` uses for rework plans.

**Re-check all five levels on every planning pass.** A project is never "done" at the lower levels. Every time you plan a frontend feature, sweep all five — design system, primitives, components, layouts, views — for additions and changes. The design system and primitives change rarely, but new components and pages are constantly added or modified, and a new component may require a new primitive or token. Do not assume lower levels are frozen; explicitly confirm whether this feature touches each level and include phases accordingly. (`/dev:figma-refresh-plan` enforces the same sweep automatically via the ledger, but only for pieces present in the mapping — so keep the mapping current here when new pieces appear.)

#### Adopting an existing / in-progress project

If the project already has UI built (e.g. on an older flow, mid design
implementation) and you are aligning it to this methodology, run this step as a
one-time **base-straightening** pass — not a greenfield plan. The bestand must be
reconciled against the design, or the ledger baseline will be wrong and the plan
will ignore work already done.

1. **Resolve the mapping + design system as usual** (above): Figma mapping,
   naming convention, responsive scaling rules, breakpoints.
2. **Audit the existing code against the design, across all five levels:**
   - *Design system:* Is there a theme/token layer, or are values hardcoded? Do
     existing tokens match the naming convention and responsive rules?
   - *Primitives:* Do reusable primitives exist? Is there a browseable preview?
   - *Components / layouts / views:* Which exist, and do they match the current
     design at each breakpoint?
3. **Classify each piece**: `matches` (already correct) / `refactor` (e.g.
   hardcoded → token, restructure) / `rebuild` / `missing`. Record this
   classification — it drives both the plan phases and the ledger seed.
4. **Write the remediation as plan phases**, bottom-up: normalize the design
   system first (extract hardcoded values to tokens), then primitives + preview,
   then components/layouts/views. `matches` pieces get no phase.
5. **Seed the ledger:** the pieces classified `matches` are handed to
   `/dev:figma-refresh-plan` to seed as `implemented` (with their current hash);
   everything else stays `to-implement`. Record the `matches` list in the plan so
   the refresh can seed from it (see that command's "Adoption seeding").

After this one-time pass the project is on the normal lifecycle.

If the feature has no frontend, skip this step and omit the UI/UX Spec from the plan.

### 9. Define Acceptance Tests

This is a critical step. The plan follows **Acceptance Test Driven Development (ATDD)**:

1. Extract each acceptance criterion from step 1 into a concrete, testable scenario.
2. Each acceptance criterion gets a corresponding acceptance test.
3. Acceptance tests should test from the outside-in — through the application's public interfaces (API endpoints, CLI commands, UI interactions). Use real or in-process infrastructure where practical; mock only external services.

**Frontend features:** acceptance criteria may cover UX, not just behavior — that a screen renders its loading/empty/error/success states, that the layout is responsive, that controls are keyboard-accessible. Where these are practical to assert in code (e.g. an empty-state element appears, a control has an accessible label), write them as acceptance tests. Where they are inherently visual (does it look right, is the hierarchy clear), they are verified by the visual review loop in `/dev:implement` rather than by an automated test — note these criteria in the UI/UX Spec instead of forcing a brittle pixel test.

The acceptance tests are structured into the plan as follows:

- **Phase 1 is always: Create acceptance test scaffolds.** All tests are written as skipped/pending stubs. Each stub has a clear name describing the acceptance criterion it validates. After this phase, `./do check` passes (skipped tests don't fail).
- **Each subsequent implementation phase** follows this cycle:
  1. Unskip and implement the acceptance test(s) for this phase
  2. Verify the test fails (red)
  3. Implement the production code to make the test pass
  4. Verify `./do check` passes (green)

The acceptance test passing is the definition of "phase complete."

### 10. Determine Next Plan Number

Scan both directories for the highest existing plan number:
- `product/plans/todo/`
- `product/plans/done/`

The new plan gets the next number (e.g., if highest is 004, new plan is 005).

### 11. Write the Plan

Create the plan file at `product/plans/todo/XXX-PLAN-FEATURE-NAME.md`.

Use the following format. Order phases logically. DDD/ADR phases typically come before or alongside the code phases they document. The acceptance test scaffold phase is always first.

**Every instruction in the plan must be specific and unambiguous.** Don't write "implement appropriate error handling" — write "return HTTP 422 with body `{ "error": "invalid_email", "message": "Email must contain @" }` when the email field fails validation." The implementer should be able to work mechanically.

The plan should reflect all the decisions made with the user in step 3.

#### Plan Format

````markdown
# {{Feature Name}} Plan

## Overview

Describe what this feature does and why it's needed. What problem does it solve? What value does it deliver?

## Acceptance Criteria

| # | Criterion | Acceptance Test |
|---|-----------|-----------------|
| 1 | {{User-facing behavior}} | `{{test file}}`: `{{test name}}` |
| 2 | {{User-facing behavior}} | `{{test file}}`: `{{test name}}` |

---

<!-- Include this section ONLY for features with a user-facing frontend. Omit it entirely otherwise. -->

## UI/UX Spec

### Design Direction

{{Overall visual style and tone, decided with the user in step 8. Reference the existing design language if any.}}

### References

{{Figma URLs, screenshots, or existing screens to match. "None" if not applicable.}}

<!-- Include this subsection ONLY for Figma-sourced designs. It is the mapping that /dev:figma-refresh-plan consumes. -->

### Figma Mapping

Confirmed with the user. Logical pieces, their Figma node IDs per breakpoint, and code identity (from Code Connect where mapped).

| Logical piece | Level | Desktop node | Tablet node | Mobile node | Code target |
|---------------|-------|--------------|-------------|-------------|-------------|
| {{e.g. Login}} | view | `{{node}}` | `{{node}}` | `{{node}}` | `{{Code Connect / path}}` |
| {{e.g. Card}} | component | `{{node}}` | `{{node}}` | `{{node}}` | `{{Code Connect / path}}` |
| {{e.g. Button}} | primitive | `{{node}}` | — | — | `{{Code Connect / path}}` |

Views, layouts, and components are responsive — fill every breakpoint node they have. Tokens and simple primitives that don't vary by breakpoint use a single node (`—` for the rest).

**Ignore (do not build):** {{flow/annotation frames to exclude, by name or node ID}}

### Design System & Tokens

{{Component library and/or token set: colors, typography, spacing, radii. Be explicit.}}

**Token naming convention:** {{e.g. text `{role}-{size}-{weight}` → `body-lg-regular`; spacing `space-{step}`. This is how families and size steps are recognized.}}

**Breakpoints:** {{e.g. mobile ≤480, tablet 481–1024, desktop ≥1024}}
**Container max-widths:** {{e.g. content 1200px, centered with 24px gutters}}

**Responsive scaling** (strategy per axis — fluid or stepped):

| Family / axis | Strategy | Rule |
|---------------|----------|------|
| {{e.g. body text}} | stepped | desktop `body-lg-regular`, tablet `body-md-regular`, mobile `body-sm-regular` |
| {{e.g. section spacing}} | fluid | `clamp` from `space-4`(min) to `space-8`(max) over 480–1024px |

### Screens & Components

| Screen / Component | Purpose | Key elements |
|--------------------|---------|--------------|
| {{name}} | {{what it's for}} | {{elements}} |

### States

For each screen/component, the states that must be implemented:

| Screen / Component | States required |
|--------------------|-----------------|
| {{name}} | loading, empty, error, success, disabled (list those that apply) |

### Responsive Behavior

{{Target breakpoints and how the layout adapts. e.g. desktop ≥1024px, tablet, mobile ≤480px.}}

### Accessibility

{{Keyboard navigation, visible focus states, contrast, labelled controls, any specific requirements.}}

---

## Phase 1: Acceptance Test Scaffolds

**Type:** Tooling

### Goal

Create all acceptance tests as skipped/pending stubs. After this phase, `./do check` passes with skipped tests.

### Changes

| File | Action | Details |
|------|--------|---------|
| `{{test path}}` | Create | Skipped acceptance tests for criteria #1–#N |

### Verification

- All acceptance tests exist and are skipped
- Run `./do check` — passes (skipped tests don't fail)

---

## Phase 2: {{Phase Title}}

**Type:** {{Backend | Frontend | DDD | ADR | Tooling}}

### Goal

What this phase accomplishes.

<!-- Include this subsection ONLY for Frontend phases. It tells the implementer how to build the UI and gives the visual review something concrete to check against. -->

### Design Notes

- **Screens/components in this phase:** {{from the UI/UX Spec}}
- **States to implement:** {{loading, empty, error, success, disabled — those that apply}}
- **Design direction & tokens:** {{reference the UI/UX Spec; note anything phase-specific}}
- **Responsive/accessibility:** {{phase-specific notes, or "per UI/UX Spec"}}

### Acceptance Test (Red)

Unskip and implement the acceptance test(s) for this phase:

| Test | Criterion | Expected Behavior |
|------|-----------|-------------------|
| `{{test name}}` | #1 | {{What the test asserts}} |

Verify the test **fails** (red) before implementing production code.

### Changes

| File | Action | Details |
|------|--------|---------|
| `{{path}}` | Create / Modify | {{Specific, unambiguous description of what changes}} |

### Verification

- Acceptance test(s) pass (green)
- Run `./do check` — all checks pass

---

<!-- Repeat for each implementation phase, always starting with the acceptance test -->

<!-- Include DDD, ADR, and ./do phases as needed: -->

## Phase N: DDD — {{Update Glossary / Create Context Doc / Update Context Map}}

**Type:** DDD

### Goal

Document new domain concepts introduced by this feature.

### Changes

| File | Action | Details |
|------|--------|---------|
| `product/DDD/glossary.md` | Modify | Add definitions for {{new terms}} |
| `product/DDD/contexts/{{name}}.md` | Create | Bounded context document for {{context}} |
| `product/DDD/context-map.md` | Modify | Add {{context}} and its relationships |

### Verification

- Review documentation for accuracy and completeness

---

## Phase N: ADR — {{Decision Title}}

**Type:** ADR

### Goal

Record the architectural decision made for this feature.

### Changes

| File | Action | Details |
|------|--------|---------|
| `product/ADR/{{NNN}}-{{slug}}.md` | Create | ADR documenting {{decision}} |

### Verification

- Review ADR for completeness (context, decision, rationale, consequences)

---

## Phase N: `./do` Script Updates

**Type:** Tooling

### Goal

Update the task runner to support the new feature's requirements.

### Changes

| File | Action | Details |
|------|--------|---------|
| `./do` | Modify | {{Specific changes: new commands, updated dependency checks, updated check pipeline}} |

### Verification

- `./do check` includes new acceptance tests
- `./do run` still works from a fresh checkout
- Any new `./do` subcommands work correctly

---

## Files Summary

| File | Action | Purpose |
|------|--------|---------|
| `{{path}}` | Create / Modify | {{Purpose}} |

---

## End-to-End Verification

After all phases are complete:

1. All acceptance tests pass (none skipped)
2. `./do check` passes — full verification pipeline
3. `./do run` works from a fresh checkout
4. {{Additional step-by-step verification of the complete feature}}

---

## Update Considerations

How will this feature behave when updating from an older version?

- **Config changes**: New keys with defaults / None
- **Storage changes**: New directories created on demand / None
- **Dependency changes**: New packages / None
- **Migration needed**: Yes (describe) / No
- **Backwards compatibility**: How old installations are affected
````

### 12. Review Plan via Sub-Agents

Before presenting the plan to the user, run an automated review using specialized sub-agents in parallel (four for backend-only plans, five when the plan has a frontend):

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

**Frontend caveat:** purely visual micro-details (exact pixel spacing, hover/focus styling, transitions, micro-copy) do NOT need to be pinned down — the implementer is expected to apply design judgment there, guided by the UI/UX Spec and the `frontend-design` skill. Flag a frontend phase only when the *design direction, tokens, screens, or required states* are missing — not when a transition duration is unspecified.

#### Reviewer 5: UI/UX Completeness (frontend plans only)

Skip this reviewer if the plan has no frontend. Otherwise checks:
- Is there a UI/UX Spec section, and does it cover design direction, design system/tokens, screens, and states?
- Does every screen/component list its required states (loading, empty, error, success, disabled)?
- Are responsive breakpoints and accessibility requirements specified?
- Does every Frontend phase have Design Notes that connect it to the UI/UX Spec?
- Is the design direction concrete enough that an implementer won't fall back to templated defaults — without being so rigid that it removes all visual judgment?
- Do frontend acceptance criteria cover UX (states, responsiveness, a11y), not just behavior?

**Process:**
1. Launch the reviewers in parallel as sub-agents (Reviewer 5 only if the plan has a frontend). Provide each with the full plan text and the relevant codebase context.
2. Collect all findings.
3. Synthesize: resolve each finding by updating the plan.
4. Run one final holistic review sub-agent with fresh eyes on the updated plan. This reviewer checks the whole plan end-to-end, including that the fixes from step 3 didn't introduce new issues.
5. Apply any final fixes.

### 13. Final Review with User

Present the complete, reviewed plan to the user. Include a summary of what the review process found and fixed:

- Walk through each phase briefly
- Highlight key decisions and their rationale
- Highlight the acceptance test strategy
- Note any `./do` script changes

Then use `AskUserQuestion` to present a selection with two options:
- **"Commit and push (Recommended)"** — The plan is approved. Proceed to step 14.
- **"I have changes"** — The user wants to iterate. Listen to their feedback, update the plan, and present the selection again.

Do NOT ask an open-ended question. Always use the selection box so the user can approve quickly.

### 14. Commit and Push

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

6. **The plan must be decision-complete.** An implementer must be able to execute every phase mechanically. No ambiguity, no judgment calls, no unresolved choices. **One exception:** on frontend phases, the implementer applies design judgment to small unspecified visual details, guided by the UI/UX Spec and the `frontend-design` skill. The plan fixes the design *direction*, system, screens, and states — not every pixel.

7. **The plan is reviewed before the user sees it.** Specialized sub-agent reviewers check feasibility, ATDD coverage, architecture, decision completeness, and — for frontend plans — UI/UX completeness. Findings are fixed before presenting to the user.

8. **Plans follow the format defined in step 11.** The plan format is embedded above — do not look for external template files.

9. **Plans go in `product/plans/todo/`.** After implementation via `/dev:implement`, they are moved to `product/plans/done/`.

10. **Always commit and push.** The approved plan is committed to main and pushed immediately.

11. **Frontend features get a UI/UX Spec.** Any user-facing frontend feature must include the UI/UX Spec section (design direction, system/tokens, screens, states, responsive, accessibility), mark its UI phases as `**Type:** Frontend` with Design Notes, and decide the design *with the user*. This is what lets `/dev:implement` build intentional UI and visually review it instead of falling back to defaults.
