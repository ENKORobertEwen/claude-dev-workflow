# Implement Plan Command

You are the orchestrator. Your job is to execute a plan by delegating ALL work to sub-agents. You never implement code, fix errors, or run verification yourself — you launch sub-agents to do that work, and you create commits when verification passes.

## Input

The user provides a plan to implement. Find it using this priority:

1. **File path provided** — Use the specified file
2. **Plan in current session** — If a plan was created during this session, use it
3. **Scan `product/plans/todo/`** — If one plan exists, use it. If multiple exist, pick the one with the lowest plan number. If none exist, error: "No plans found in product/plans/todo/"

## Detecting Frontend Phases

Each phase in the plan declares a `**Type:**` (Backend, Frontend, DDD, ADR, Tooling). A phase is a **frontend phase** when its type is `Frontend` — or, if the plan predates typed phases, when it creates/modifies user-facing UI (components, pages, views, styles, templates).

Frontend phases get two extra things the others don't:
1. The implementation sub-agent is launched in **frontend mode** (design guidance + permission to apply taste).
2. After `./do check` passes, the phase goes through a **visual review loop** before the commit.

Non-frontend phases follow the plain implement → verify → commit cycle.

## Four Types of Sub-Agents

### 1. Implementation Sub-Agent

Receives a phase description and implements it. Instructions to give the sub-agent:

```
Implement the following phase from the plan:

[paste the phase description, including goal, changes table, and any relevant context]

Context:
- Read CLAUDE.md for project conventions
- Read any files you need to modify before editing them
- Follow existing patterns in the codebase

Rules:
- Do NOT run ./do check
- Do NOT create any commits
- NEVER ask questions or use AskUserQuestion — just implement what the phase says
- Report back what you implemented when done
```

**For frontend phases, append this to the instructions (frontend mode):**

```
This is a FRONTEND phase. UI quality matters as much as correctness. The
governing rule: **derive everything from the design artifacts; for anything not
derivable, REPORT it — never invent.**

- Invoke the `frontend-design` skill (if available) before writing UI code, and follow its guidance on visual hierarchy, typography, spacing, and avoiding templated defaults.
- **Design system first.** Before building any primitive, component, or view, establish the design system from `tokens.json` (colors, spacing, text styles, radii) as the project's theme / CSS variables. Everything downstream references these — you cannot reference a token that does not exist yet. If this project has no theme layer yet, create it from the tokens as the first step.
- **Tokens, not hardcoded values (strict).** Every color/space/typography/radius value MUST reference a token. A measured value that matches NO token is a deviation: report it in the summary/PR — do NOT silently hardcode a raw value (e.g. `15px`, `#3B82F6`).
- **Responsive tokens by the recorded rule.** Apply typography and spacing across breakpoints exactly as the UI/UX Spec's responsive scaling rules say — never guess which size belongs at which breakpoint. For *stepped* families, apply the named token mapped to each breakpoint via media queries (e.g. `body-lg-regular` desktop → `body-md-regular` tablet → `body-sm-regular` mobile). For *fluid* families, use the derived `clamp()` token from `tokens.json` verbatim.
- **Reuse mapped components.** If a piece has a `codeTarget` (from Code Connect or the plan's mapping in `status.json`), use or extend that component — do NOT build a parallel one. Respect dependency order: design system → primitives → components → layouts → views.
- **Primitives second, with a browseable preview.** After the design system, build the primitives — and expose them on a browseable preview so they can be visually checked in isolation: Storybook stories if the project uses Storybook, otherwise a simple dev route/page (e.g. `/__design`) rendering each primitive in all its states. Add components to the same preview as you build them. This is what the visual-review loop checks for non-screen pieces. Build it when feasible for the project's setup; if genuinely not feasible, note why in the report.
- **Layout = intent, not canvas coordinates.** Translate Figma auto-layout to flex/grid, semantic and fluid. Do NOT use absolute positioning or fixed x/y pixel coordinates from the Figma canvas. Only the breakpoints named in the UI/UX Spec / mapping — never invented ones.
- **Components are responsive too.** Components, layouts, and views all have breakpoint-specific behavior — implement each across its mapped breakpoints (desktop/tablet/mobile), not just full pages. Build/check the responsive behavior of a component in the browseable preview.
- **Semantic HTML + accessibility.** Use semantic elements (`<button>`, `<nav>`, headings, labelled controls) — not `<div>` for everything. Implement the a11y requirements from the UI/UX Spec; keyboard-accessible with visible focus states.
- **All states.** Implement EVERY state the UI/UX Spec lists for this phase's screens: loading, empty, error, success, disabled, plus hover/focus/active — not just the happy path.
- **Assets and copy.** Use exported assets / the existing icon set — do NOT redraw icons. Use the exact text/copy from the design — do NOT paraphrase.
- **Follow project conventions.** Units (px/rem), CSS approach, and file structure come from CLAUDE.md and the existing code — match them; do NOT impose a different convention.
- **Figma is the source of truth** for sourced designs; read the design files `/dev:figma-refresh-plan` produced (`context.md`, `tokens.json`) — do NOT pull from Figma yourself. If the design appears to violate a design principle, report it rather than changing it.
- **Only unspecified visual micro-details** (exact transition timing, micro-copy where the design has none, hover/focus styling Figma doesn't define) may use your design judgment — this is the one narrow exception to "implement the most literal interpretation." Do NOT invent functional requirements or scope.
```

### 2. Verification Sub-Agent

Runs `./do check` with output captured to a unique log file. Instructions to give the sub-agent:

```
Run the project's full verification command, capturing all output to a unique log file:

LOG_FILE="/tmp/do-check-$(date +%s)-$$.log"
./do check 2>&1 | tee "$LOG_FILE"

Report back with one of:
1. PASS — All checks passed. Include the log file path.
2. CODE FAILURE — Lint errors, build errors, or test failures. Include the log file path.
3. INFRASTRUCTURE FAILURE — Network issues, missing tools, permission errors, environment problems. Include the log file path.

IMPORTANT:
- Always include the log file path in your response. Do NOT paste the full output — the log file is the source of truth.
- NEVER ask questions or use AskUserQuestion — just report the result.
```

### 3. Fix Sub-Agent

Reads the verification log file and fixes the issues. Instructions to give the sub-agent:

```
The code has verification errors. Your task is to fix ALL errors.

The verification log is at: [log file path]

Instructions:
1. Read the log file to understand what failed
2. Identify which files have errors
3. Read those files to understand the issues
4. Fix ALL errors in the code
5. Report back what you fixed

Rules:
- Do NOT run ./do check
- Do NOT create any commits
- NEVER ask questions or use AskUserQuestion — just fix and report
```

### 4. UI Review Sub-Agent (frontend phases only)

Renders the running application and critiques the UI against the plan's UI/UX Spec. This is the visual equivalent of the verification sub-agent — it catches what `./do check` cannot (looks broken, wrong hierarchy, missing states, not responsive, inaccessible). Instructions to give the sub-agent:

```
You are reviewing the UI produced by the phase just implemented. Use the Playwright browser tools (load their schemas via ToolSearch with query "browser_navigate browser_take_screenshot browser_resize browser_snapshot").

Setup:
1. Make sure the app is running. Check if it is already up at the known URL; if not, start it with: ./do run 2>&1 | tee /tmp/do-run-ui.log & then wait ~5s.
2. Determine the frontend URL using the same priority as the orchestrator: $HOST1–$HOST4 env vars → URLs printed in the run log → http://localhost:$PORT1 → the ./do script. Use the public HOST URL, never localhost, if HOST vars are set.

Review, for the screen(s) this phase touched:
[list the screens/components from the phase and the states from the UI/UX Spec]

For primitive/component phases that have no full screen, review the browseable
preview page instead (the Storybook URL or the dev route like `/__design`)
where those pieces are rendered in all their states.

3. Navigate to each screen. Take a screenshot at desktop width (1280px) and mobile width (390px) — use browser_resize.
4. For each state listed in the UI/UX Spec (loading, empty, error, success, disabled), exercise it if reachable and screenshot it.
5. Check against the UI/UX Spec and the frontend-design principles:
   - Visual hierarchy, spacing, alignment, typography — does it look intentional, not templated/broken?
   - Are all required states present and handled (no raw error dumps, no blank empty states)?
   - Responsive: does the mobile layout work (no overflow, tap targets usable)?
   - Accessibility: visible focus states, readable contrast, labelled controls.
   - Does it match the design direction/tokens fixed in the plan?

Report back with one of:
1. UI PASS — The UI matches the spec and looks intentional. Briefly note what you checked.
2. UI ISSUES — A concrete, numbered list of problems, each with: the screen/state, what's wrong, and what it should be. Be specific and actionable (a fix sub-agent will act on this list verbatim).

IMPORTANT:
- Judge against the plan's spec and sound design principles, NOT personal preference for a different design language.
- NEVER ask questions or use AskUserQuestion — just report the result.
```

## Orchestration Process

### 1. Read and Analyze the Plan

- Read the plan file completely
- Identify all phases
- Create a task list with all phases

### 2. Create or Resume Branch

Derive a branch name from the plan filename. For example, plan `003-PLAN-USER-AUTH.md` becomes branch `feat/003-user-auth`.

**New implementation:**
1. Ensure you're on `main` and it's up to date: `git checkout main && git pull`
2. Create and switch to the branch: `git checkout -b feat/XXX-feature-name`

**Resuming a partial implementation:**
If the branch already exists (from a previous interrupted run):
1. Switch to the existing branch: `git checkout feat/XXX-feature-name`
2. Check `git log --oneline` to determine which phases were already committed
3. Skip completed phases and continue from the next incomplete phase
4. Inform the user which phases were already done and where you're resuming from

### 3. Execute Each Phase

For EACH phase in the plan:

**a) Launch implementation sub-agent**
- Provide the phase description with full context
- For frontend phases, launch it in **frontend mode** (see sub-agent type 1)
- Wait for completion

**b) Launch verification sub-agent**
- Wait for result

**c) Handle verification result**

- **PASS** → Proceed to visual review (frontend phases) or commit (other phases)
- **CODE FAILURE** → Launch fix sub-agent with the log file path, then launch verification sub-agent again. Repeat up to 3 verify-fix cycles. If still failing after 3 cycles, STOP and inform the user.
- **INFRASTRUCTURE FAILURE** → STOP. Inform the user about the infrastructure issue and the log file path. Do NOT retry, do NOT ask questions.

**c2) Visual review (frontend phases only)**

After `./do check` passes, and only for frontend phases:

- Launch the **UI review sub-agent** for the screens/states this phase touched. Wait for result.
- **UI PASS** → Proceed to commit.
- **UI ISSUES** → Launch a fix sub-agent with the issue list (in frontend mode), then re-run the verification sub-agent (`./do check` must stay green), then launch the UI review sub-agent again. Repeat up to 3 review-fix cycles. If issues remain after 3 cycles, do NOT block: proceed to commit, and record the remaining UI issues so they surface in the final summary and PR body.

This keeps any visual fixes folded into the same phase commit. The app may stay running between frontend phases — the UI review sub-agent reuses it rather than restarting each time.

**c3) Ledger write-back (Figma-sourced frontend phases only)**

If the phase implemented Figma-sourced pieces tracked in
`product/design/<plan-slug>/status.json`, update each implemented piece before
committing: set its `lastImplementedHash` to its `currentHash` and its `status`
to `implemented`. A `to-review` piece that turned out to need no change is also
set back to `implemented` (hash updated to current). This is what makes the next
`/dev:figma-refresh-plan` diff meaningful. Stage `status.json` with the phase
commit.

Do NOT touch the acceptance fields (`acceptedHash` / `acceptedBy` /
`acceptedAt`). Implementation is not acceptance — the piece becomes
`awaiting-acceptance` until a human signs off via `/dev:figma-accept`. Passing
the automated visual review does not accept the piece.

**d) Create commit**

Once verification passes, YOU create the commit:

1. Run `git status` to see all changes
2. Stage all relevant files with `git add`
3. Create a focused commit for this phase:

```bash
git commit -m "$(cat <<'EOF'
Summary line describing what this phase accomplished

- Detail about what changed
- Detail about why (if not obvious)
EOF
)"
```

**e) Mark phase complete and move to next**

### 4. Finalize

After all phases are complete:

1. Move the plan from `product/plans/todo/` to `product/plans/done/`:
   ```bash
   git mv product/plans/todo/XXX-PLAN-FEATURE-NAME.md product/plans/done/
   ```
2. Amend the last phase commit to include the plan move:
   ```bash
   git add product/plans/
   git commit --amend --no-edit
   ```
3. Push the branch:
   ```bash
   git push -u origin feat/XXX-feature-name
   ```

### 5. Start the Application

Start the application and capture its output to determine the live URLs:

```bash
./do run 2>&1 | tee /tmp/do-run-output.log &
sleep 5
```

**Determine live URLs** using this priority:

1. **`$HOST1`–`$HOST4` environment variables** — If set, use these (they are public URLs). Check CLAUDE.md's Runtime Environment section for which ones are relevant.
2. **Parse `./do run` output** — If HOST variables are not set, read `/tmp/do-run-output.log` and extract any URLs the server printed (look for patterns like `http://localhost:XXXX`, `https://...`, `listening on ...`, `ready at ...`).
3. **Construct from PORT variables** — If neither HOST vars nor output URLs are available, check `$PORT1`–`$PORT4` and construct `http://localhost:$PORT1` etc.
4. **Read `./do` script** — As a last resort, read the `./do` script's `run` command to find which port(s) it uses and construct localhost URLs from that.

**This step must produce at least one URL.** If none of the above yields a URL, note "URL could not be determined — check `./do run` manually" but still include the Live URLs section in the PR.

### 6. Create Pull Request (GitHub only)

Check if `GITHUB_TOKEN` is set in the environment. If so, create a pull request using the GitHub REST API.

**Determine the repo:** Parse the `origin` remote URL to extract `owner/repo`.

**Generate PR body from the plan AND the runtime summary.** The PR body must include everything a reviewer needs — the same information shown in the CLI summary. Specifically:

```
## Summary

[from plan overview]

## Live URLs

The application is running and can be tested at:
- **Frontend:** [URL determined in step 5]
- **API:** [URL determined in step 5, if separate]

## Acceptance Criteria

[from plan — as a checklist, all checked]

## Phases

- [x] Phase 1: ...
- [x] Phase 2: ...

## Known UI Issues

[Only if the visual review loop ended with unresolved UI issues on any frontend phase. List them as a checklist of follow-ups. Omit this section entirely if there are none.]

## Acceptance

[Only for Figma-sourced frontend work. List each implemented piece as an unchecked box with the live URL, for a tester to sign off:]

- [ ] `login.view.mobile` — review at [live URL] and accept
- [ ] `card.component.desktop` — review at [live URL] and accept

Tick a box once you've reviewed and accept that piece on the live URL. Acceptance is tracked in the design ledger — run `/dev:figma-accept --from-pr <this PR number>` to sync the checked items. **Acceptance does NOT block merging** — open items remain visible as `awaiting-acceptance` and can be accepted later.
```

**The "Live URLs" section is mandatory.** Always include it using the URLs determined in step 5. Never omit this section — even if no URL could be determined, include the section with a note that the URL needs to be checked manually.

**Create the PR:**

```bash
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  "https://api.github.com/repos/OWNER/REPO/pulls" \
  -d '{
    "title": "XXX — Feature Name",
    "head": "feat/XXX-feature-name",
    "base": "main",
    "body": "[the full body as described above]"
  }'
```

**If `GITHUB_TOKEN` is not set:** Skip PR creation. Inform the user that the branch was pushed and they can create a PR manually. Mention that setting `GITHUB_TOKEN` enables automatic PR creation.

### 7. Summary

Print the same summary to the CLI that was included in the PR body:
- List of phases completed
- List of commits created
- Branch name
- PR URL (if created)
- Application URLs (as determined in step 5 — from HOST variables, `./do run` output, PORT variables, or `./do` script)
- Any unresolved UI issues from the visual review loop (or "none")
- Confirmation that the plan is fully implemented and the app is running

**The CLI summary and the PR body must contain the same information.** The PR is the permanent record — anyone looking at the PR should see exactly what the implementer saw.

## Critical Rules

### The Orchestrator NEVER:

- **Asks questions** — NEVER use `AskUserQuestion`. This is a fully headless process. All operational decisions have predefined behaviors (see below). No exceptions.
- **Runs `./do check`** — Delegate to a verification sub-agent
- **Runs any shell commands** — Except `git` commands for branch/commit operations, `curl` for PR creation, `./do run` to start the app, and `mv` for moving the plan file
- **Implements code changes** — Delegate to implementation sub-agents
- **Fixes errors** — Delegate to fix sub-agents
- **Skips verification** — Every phase must pass `./do check` before committing
- **Reviews UI itself** — Delegate visual review of frontend phases to the UI review sub-agent (it drives Playwright, not the orchestrator)

### The Orchestrator ALWAYS:

- **Creates a feature branch** from main before starting
- **Resumes from where it left off** if the branch already exists
- **Delegates implementation** to sub-agents (frontend mode for frontend phases)
- **Delegates verification** to sub-agents
- **Delegates visual review** to a UI review sub-agent for frontend phases, before committing
- **Delegates error fixing** to sub-agents
- **Creates commits** after verification passes (and, for frontend phases, after the visual review loop)
- **Stops on infrastructure failures** — Full stop, inform the user, do NOT ask or retry
- **Creates small, focused commits** — One per phase, not batched
- **Amends the last commit** to include the plan move to `done/`
- **Pushes the branch** after all phases complete
- **Starts the application** with `./do run` before creating the PR
- **Creates a PR** via GitHub REST API if `GITHUB_TOKEN` is available, with live URLs in the body (mandatory section — never omitted)
- **Keeps CLI summary and PR body in sync** — same information in both

### Sub-Agents NEVER:

- Ask questions or request clarification — NEVER use `AskUserQuestion`
- Run `./do check` (except verification sub-agents)
- Create commits
- Move plan files
- Push or create branches

## Hardcoded Operational Decisions

This command is fully headless. Every operational scenario has a predefined behavior. The orchestrator and all sub-agents follow these rules without exception — no asking, no prompting, no confirmation.

| Scenario | Behavior |
|----------|----------|
| `GITHUB_TOKEN` not set | Push the branch. Skip PR creation. Inform user that setting `GITHUB_TOKEN` enables automatic PRs. |
| `GITHUB_TOKEN` set but PR creation fails | Inform user of the error. The branch is already pushed — they can create a PR manually. |
| `./do check` fails with code errors | Launch fix sub-agent. Retry verification. Repeat up to 3 cycles, then full stop. |
| `./do check` fails with infrastructure errors (DNS, network, permissions, missing tools) | Full stop. Inform the user what happened. Do NOT retry. |
| Frontend phase: UI review returns UI ISSUES | Launch fix sub-agent (frontend mode), re-verify `./do check`, re-run UI review. Up to 3 review-fix cycles, then commit anyway and record remaining issues in the summary/PR. |
| Frontend phase: app won't start for UI review | UI review sub-agent reports it could not render. Record as a known issue, proceed to commit. Do NOT block the pipeline. |
| Phase has no `**Type:**` and isn't obviously UI | Treat as non-frontend: skip the visual review loop. |
| `./do run` fails | Inform user of the error. Continue to PR creation — the failure is noted in the summary. |
| Multiple plans in `product/plans/todo/` | Pick the one with the lowest plan number. |
| No plans in `product/plans/todo/` | Error: "No plans found in product/plans/todo/". Stop. |
| Branch already exists | Resume: check git log for completed phases, skip them, continue from next. |
| `git push` fails | Inform user. Do NOT retry. The commits are local — they can push manually. |
| Fix sub-agent cannot resolve errors after 3 verify-fix cycles | Full stop. Inform user what failed and which phase. Do NOT keep retrying. |
| Ambiguous or unclear plan instructions | Implement the most literal, straightforward interpretation. Do NOT ask for clarification. |
