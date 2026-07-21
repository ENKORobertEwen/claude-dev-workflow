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

- **You are the final authority on visual fidelity. There is no check after you.** For design-sourced phases the result must match the referenced design frame 100% — a deviation you do not fix will not be fixed by anyone else. Never defer a visible deviation to a later step; there is no later step.
- **Tests never block a fidelity fix.** If an existing test — including one written in an earlier phase of this same plan — pins something that prevents matching the design (plain-text content where the design shows formatting, specific glyph characters, text vs. vector asset), change the test to match the design; do NOT leave the deviation in place because a test pins the old state. A test you wrote yourself is not evidence the design is wrong. Tests follow the design, never the reverse.

- Invoke the `frontend-design` skill (if available) before writing UI code, and follow its guidance on visual hierarchy, typography, spacing, and avoiding templated defaults.
- **Design system first.** Before building any primitive, component, or view, establish the design system from `tokens.json` (colors, spacing, text styles, radii) as the project's theme / CSS variables. Everything downstream references these — you cannot reference a token that does not exist yet. If this project has no theme layer yet, create it from the tokens as the first step.
- **Tokens, not hardcoded values (strict).** Every color/space/typography/radius value MUST reference a token. A measured value that matches NO existing token is the ONLY thing you may classify as a deviation instead of fixing: report it in your completion report — do NOT silently hardcode a raw value (e.g. `15px`, `#3B82F6`). Every other visible mismatch with the design MUST be fixed, not reported.
- **Responsive tokens by the recorded rule.** Apply typography and spacing across breakpoints exactly as the UI/UX Spec's responsive scaling rules say — never guess which size belongs at which breakpoint. For *stepped* families, apply the named token mapped to each breakpoint via media queries (e.g. `body-lg-regular` desktop → `body-md-regular` tablet → `body-sm-regular` mobile). For *fluid* families, use the derived `clamp()` token from `tokens.json` verbatim.
- **Reuse mapped components.** If a piece has a `codeTarget` (from Code Connect or the plan's mapping in `product/design/status.json`), use or extend that component — do NOT build a parallel one. Respect dependency order: design system → primitives → components → layouts → views.
- **Primitives second, with a browseable preview.** After the design system, build the primitives — and expose them on a browseable preview so they can be visually checked in isolation: Storybook stories if the project uses Storybook, otherwise a simple dev route/page (e.g. `/__design`) rendering each primitive in all its states. Add components to the same preview as you build them. The preview is the only place non-screen pieces are ever rendered — a piece missing from it is invisible. Build it when feasible for the project's setup; if genuinely not feasible, note why in the report.
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

Renders the running application and critiques the UI against the plan's UI/UX Spec. This is the visual equivalent of the verification sub-agent — it catches what `./do check` cannot (looks broken, wrong hierarchy, missing states, not responsive, inaccessible).

**This reviewer runs blind.** Never give it the implementation sub-agent's report, any findings/audit document, or any list of "known", "accepted", or "reported" deviations — the agent that produced a deviation does not get to decide what the reviewer may see. The reviewer reports EVERY visible deviation; whether a deviation is acceptable is decided exclusively by the human at sign-off (`/dev:figma-accept`), never by any agent in this pipeline. For Figma-sourced phases, pass it the mapped Figma node IDs for this phase's pieces (from the plan's Figma Mapping / `product/design/status.json`) so it can pull reference screenshots itself. Instructions to give the sub-agent:

```
You are reviewing the UI produced by the phase just implemented. Use the Playwright browser tools (load their schemas via ToolSearch with query "browser_navigate browser_take_screenshot browser_resize browser_snapshot").

Setup:
1. Make sure the app is running. Check if it is already up at the known URL; if not, start it with: ./do run 2>&1 | tee /tmp/do-run-ui.log & then wait ~5s.
2. Determine the frontend URL using the same priority as the orchestrator: $HOST1–$HOST4 env vars → URLs printed in the run log → http://localhost:$PORT1 → the ./do script. Use the public HOST URL, never localhost, if HOST vars are set.

Review, for the screen(s) this phase touched:
[list the screens/components from the phase and the states from the UI/UX Spec]

[For Figma-sourced phases, additionally:]
This phase implements Figma-sourced pieces. Their mapped Figma nodes:
[list piece → Figma node ID(s) per breakpoint]

Pull a reference screenshot for each node via the Figma MCP `get_screenshot` tool (load its schema via ToolSearch with query "select:mcp__claude_ai_Figma__get_screenshot" or by searching "figma screenshot"). These references are your comparison baseline: compare each rendered screen/state against its reference and report EVERY visible deviation — layout, spacing, typography, color, assets (icon/glyph shape and weight, vector vs. text rendering), text formatting (underlines, emphasis, line breaks), and content. Do NOT classify any deviation as known, accepted, or intentional — you have no information about what anyone accepted, and acceptance is not your call. A deviation you can see goes in the list.

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
- **Create the visible task list — MANDATORY, via the task tools.** Before any other
  work (before branching, before the first sub-agent), create one task per plan phase
  using the harness task tools (`TaskCreate`; if unavailable, `TodoWrite` — whatever
  this environment provides for the user-visible task list). This is NOT optional and a
  markdown checklist in your reply is NOT a substitute — if no task-tool call happens,
  step 1 is incomplete.
  - One task per phase, titled exactly `Phase N: <phase title from the plan>`, in plan
    order.
  - Status discipline: exactly one task `in_progress` at any time (the phase currently
    executing); set it `completed` immediately after that phase's commit, then set the
    next phase `in_progress`. Never batch status updates at the end.
  - Extra pipeline steps that the plan doesn't model as phases (final PR, plan → done/
    move, work-item wiring) get their own tasks appended after the phase tasks.

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
5. Sync the task list from step 1: mark every already-committed phase's task
   `completed` right away, and set the phase you resume at to `in_progress`

### 3. Execute Each Phase

For EACH phase in the plan:

**a) Launch implementation sub-agent**
- Provide the phase description with full context
- For frontend phases, launch it in **frontend mode** (see sub-agent type 1)
- **For frontend phases, filter the phase text before pasting it:** remove every mention of downstream review stages, visual sign-off, human acceptance, `figma-accept`, or the PR process — even if the plan's phase text mentions them. The implementation sub-agent must believe it is the last instance; knowledge of a later review invites deferring fixes to it.
- Wait for completion

**b) Launch verification sub-agent — unless the phase is check-exempt**

Before launching, check whether this phase's changes are check-exempt: run `git status --porcelain` and collect every changed/untracked file. The phase is exempt **iff EVERY file** matches a check-exempt glob — plugin default `product/**`, plus any globs the project CLAUDE.md declares under `### Check-Exempt Paths`. Decide on the actual file list, never on what the plan's phase text claims to change.

- **Exempt** (typical for DDD, ADR, and plan-only phases) → skip the verification sub-agent; `./do check`'s inputs are untouched, its result cannot have changed. If the changes include `.json`/`.yaml` files, verify they parse (`jq empty`) before committing. Note "check-exempt" in the phase's commit message body and the summary.
- **Not exempt** (any single non-matching file, no matter how small the change) → launch the verification sub-agent as below. Fail-closed: missing CLAUDE.md section or any doubt → verify.

- Wait for result

**c) Handle verification result**

- **PASS** → Proceed to visual review (frontend phases) or commit (other phases)
- **CODE FAILURE** → Launch fix sub-agent with the log file path, then launch verification sub-agent again. Repeat up to 3 verify-fix cycles. If still failing after 3 cycles, STOP and inform the user.
- **INFRASTRUCTURE FAILURE** → STOP. Inform the user about the infrastructure issue and the log file path. Do NOT retry, do NOT ask questions.

**c2) Visual review (frontend phases only)**

After `./do check` passes, and only for frontend phases:

- Launch the **UI review sub-agent** for the screens/states this phase touched — blind (see sub-agent type 4): for Figma-sourced phases pass the mapped node IDs; never pass the implementer's report, findings docs, or any deviation list. Wait for result.
- **UI PASS** → Proceed to commit.
- **UI ISSUES** → Launch a fix sub-agent with the issue list (in frontend mode), then re-run the verification sub-agent (`./do check` must stay green), then launch the UI review sub-agent again. Repeat up to 3 review-fix cycles. If issues remain after 3 cycles, do NOT block: proceed to commit, and record the remaining UI issues so they surface in the final summary and PR body.

This keeps any visual fixes folded into the same phase commit. The app may stay running between frontend phases — the UI review sub-agent reuses it rather than restarting each time.

**c3) Ledger write-back (Figma-sourced frontend phases only)**

If the phase implemented Figma-sourced pieces tracked in the global ledger
`product/design/status.json`, update each implemented piece (by its
`<source>:…` id) before
committing: set its `lastImplementedHash` to its `currentHash` and its `status`
to `implemented`. A `to-review` piece that turned out to need no change is also
set back to `implemented` (hash updated to current). This is what makes the next
`/dev:figma-refresh-plan` diff meaningful. Stage `product/design/status.json` with the phase
commit.

Do NOT touch the acceptance fields (`acceptedHash` / `acceptedBy` /
`acceptedAt`). Implementation is not acceptance — the piece becomes
`awaiting-acceptance` until a human signs off via `/dev:figma-accept`. Passing
the automated visual review does not accept the piece.

Do NOT recompute hashes here. Copy the `currentHash` that `/dev:figma-refresh-plan`
already computed (via its committed `ledger-hash.mjs` reference script) into
`lastImplementedHash`. Leave `hashSpecVersion` untouched — only
`/dev:figma-refresh-plan` produces hashes and migrates the spec, so the ledger
stays internally consistent.

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

### 6. Create Pull Request (host-aware: GitHub + Azure DevOps)

After the branch is pushed, open a PR against `main`. The PR body is identical regardless of host (see below); only the creation mechanism differs. Detect the host from the `origin` remote URL (`git remote get-url origin`) and use the matching path — prefer the platform CLI (it handles auth), fall back to REST or a manual hand-off:

- `github.com` or a GitHub Enterprise host → **GitHub**.
- `dev.azure.com` or `*.visualstudio.com` → **Azure DevOps**.
- anything else → **unknown host** (manual hand-off, below).

**Determine the repo:** Parse the `origin` remote URL — for GitHub extract `owner/repo`; for Azure DevOps let the `az` CLI auto-detect organization/project/repository from the remote (`--detect true`), so nothing is hardcoded.

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

**Create the PR — GitHub.** Prefer the `gh` CLI (it handles auth via its own login / `GH_TOKEN` / `GITHUB_TOKEN`):

```bash
gh pr create --base main --head feat/XXX-feature-name \
  --title "XXX — Feature Name" --body-file <pr-body-file>
```

If `gh` is unavailable but `GITHUB_TOKEN` is set, fall back to the REST API (parse `OWNER/REPO` from `origin`):

```bash
curl -s -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  "https://api.github.com/repos/OWNER/REPO/pulls" \
  -d '{ "title": "XXX — Feature Name", "head": "feat/XXX-feature-name", "base": "main", "body": "[the full body above]" }'
```

**Create the PR — Azure DevOps.** Use the Azure CLI with the `azure-devops` extension. `--detect true` reads the **organization** from the git remote; project and repository are resolved from git config — all three are embedded in the Azure DevOps remote URL, so no IDs are hardcoded:

```bash
az repos pr create --detect true \
  --source-branch feat/XXX-feature-name --target-branch main \
  --title "XXX — Feature Name" --description "$(cat <pr-body-file>)"
```

`az repos pr create` has **no** `--description-file`, so pass the same body file inline via `--description "$(cat <pr-body-file>)"` (markdown is supported). Auth is non-interactive via `AZURE_DEVOPS_EXT_PAT` (a PAT with Code → Read & Write, including Pull Request), or an existing `az login`. If the extension is missing, `az extension add --name azure-devops` installs it.

**Capture the PR number/URL** the tool returns: put it in the CLI summary — it is the `<PR number>` that `/dev:figma-accept --from-pr` consumes.

**Fallback — unknown host, missing CLI, or no token/PAT:** do NOT fail. The branch is already pushed. Report that, print the branch name and a compare/PR URL if one can be derived from `origin`, and tell the user to open the PR manually. Name the credential that would enable automation for their host: `gh` login / `GITHUB_TOKEN` for GitHub, or the `azure-devops` extension + `AZURE_DEVOPS_EXT_PAT` for Azure DevOps.

### 7. Summary

Print the same summary to the CLI that was included in the PR body:
- List of phases completed
- List of commits created
- Branch name
- PR URL (if created)
- Application URLs (as determined in step 5 — from HOST variables, `./do run` output, PORT variables, or `./do` script)
- Any unresolved UI issues from the visual review loop (or "none")
- For Figma-sourced work: how many pieces are now `awaiting-acceptance` (built, not yet signed off) and the pointer to `/dev:figma-accept`
- Confirmation that the plan is fully implemented and the app is running

**Do NOT suggest running `/dev:figma-refresh-plan` after implementing.** `/implement` already keeps the ledger current via the per-phase write-back (step c3), so the ledger is clean when this command finishes. `/dev:figma-refresh-plan` is only for the first design pull or when the Figma design *changes* — never a post-implement cleanup step. The only acceptance-related follow-up to mention is `/dev:figma-accept` (human sign-off), which is separate.

**The CLI summary and the PR body must contain the same information.** The PR is the permanent record — anyone looking at the PR should see exactly what the implementer saw.

## Critical Rules

### The Orchestrator NEVER:

- **Asks questions** — NEVER use `AskUserQuestion`. This is a fully headless process. All operational decisions have predefined behaviors (see below). No exceptions.
- **Runs `./do check`** — Delegate to a verification sub-agent
- **Runs any shell commands** — Except `git` commands for branch/commit operations, `curl` for PR creation, `./do run` to start the app, and `mv` for moving the plan file
- **Implements code changes** — Delegate to implementation sub-agents
- **Fixes errors** — Delegate to fix sub-agents
- **Skips verification** — Every phase that touches any non-exempt path must pass `./do check` before committing. The only skip is the mechanical check-exempt rule (ALL changed files match `product/**` or the project's declared Check-Exempt Paths) — never a judgment call like "small change" or "docs-ish"
- **Reviews UI itself** — Delegate visual review of frontend phases to the UI review sub-agent (it drives Playwright, not the orchestrator)

### The Orchestrator ALWAYS:

- **Creates a feature branch** from main before starting
- **Resumes from where it left off** if the branch already exists
- **Delegates implementation** to sub-agents (frontend mode for frontend phases)
- **Delegates verification** to sub-agents
- **Delegates visual review** to a UI review sub-agent for frontend phases, before committing — blind: the reviewer never sees the implementer's report or any list of known/accepted deviations
- **Delegates error fixing** to sub-agents
- **Creates commits** after verification passes (and, for frontend phases, after the visual review loop)
- **Stops on infrastructure failures** — Full stop, inform the user, do NOT ask or retry
- **Creates small, focused commits** — One per phase, not batched
- **Amends the last commit** to include the plan move to `done/`
- **Pushes the branch** after all phases complete
- **Starts the application** with `./do run` before creating the PR
- **Creates a PR on the detected host** — GitHub (via `gh`, else `GITHUB_TOKEN` REST) or Azure DevOps (via `az repos`) — when its credential is available; otherwise pushes and hands off to a manual PR. Live URLs in the body are a mandatory section — never omitted
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
| PR host detected but its CLI/credential is missing (`gh`/`GITHUB_TOKEN`, or `az`+`azure-devops` extension/`AZURE_DEVOPS_EXT_PAT`) | Push the branch. Skip PR creation. Print the branch + a compare/PR URL and name the credential that enables automation for that host. |
| Unknown git host, or PR creation fails | Inform user of the error/host. The branch is already pushed — they can create a PR manually. |
| Phase changes only check-exempt paths (every changed file matches `product/**` or a CLAUDE.md `### Check-Exempt Paths` glob) | Skip the verification sub-agent — the check result cannot have changed. Parse-guard any `.json`/`.yaml` in the commit (`jq empty`). Note "check-exempt" in the commit body. One non-matching file → full verification. Fail-closed on any doubt. |
| `./do check` fails with code errors | Launch fix sub-agent. Retry verification. Repeat up to 3 cycles, then full stop. |
| `./do check` fails with infrastructure errors (DNS, network, permissions, missing tools) | Full stop. Inform the user what happened. Do NOT retry. |
| Frontend phase: UI review returns UI ISSUES | Launch fix sub-agent (frontend mode), re-verify `./do check`, re-run UI review. Up to 3 review-fix cycles, then commit anyway and record remaining issues in the summary/PR. |
| Frontend phase: implementer's report lists "known", "accepted", or "reported" deviations | The UI review sub-agent never sees that list — it reviews blind. Any deviation it finds goes through the fix loop like every other UI issue. Only a human can accept a deviation (at sign-off), never the agent that produced it. |
| Frontend phase: app won't start for UI review | UI review sub-agent reports it could not render. Record as a known issue, proceed to commit. Do NOT block the pipeline. |
| Phase has no `**Type:**` and isn't obviously UI | Treat as non-frontend: skip the visual review loop. |
| Frontend phase is Figma-sourced but the global ledger `product/design/status.json` has no matching pieces (`figma-refresh-plan` was never run for that source) | Full stop BEFORE that phase, once, with: "This plan is Figma-sourced — run `/dev:figma-refresh-plan` first to pull the design and create the ledger, then re-run `/dev:implement`." Do NOT improvise, do NOT proceed to guess the design, do NOT repeat the reminder. |
| Implement finished; ledger written back | The ledger is current. Do NOT suggest running `figma-refresh-plan` afterward — it is not a cleanup step. The only follow-up to mention is `/dev:figma-accept` for human sign-off. |
| `./do run` fails | Inform user of the error. Continue to PR creation — the failure is noted in the summary. |
| Multiple plans in `product/plans/todo/` | Pick the one with the lowest plan number. |
| No plans in `product/plans/todo/` | Error: "No plans found in product/plans/todo/". Stop. |
| Branch already exists | Resume: check git log for completed phases, skip them, continue from next. |
| `git push` fails | Inform user. Do NOT retry. The commits are local — they can push manually. |
| Fix sub-agent cannot resolve errors after 3 verify-fix cycles | Full stop. Inform user what failed and which phase. Do NOT keep retrying. |
| Ambiguous or unclear plan instructions | Implement the most literal, straightforward interpretation. Do NOT ask for clarification. |
