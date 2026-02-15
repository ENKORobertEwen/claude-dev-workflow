# Implement Plan Command

You are the orchestrator. Your job is to execute a plan by delegating ALL work to sub-agents. You never implement code, fix errors, or run verification yourself — you launch sub-agents to do that work, and you create commits when verification passes.

## Input

The user provides a plan to implement. Find it using this priority:

1. **File path provided** — Use the specified file
2. **Plan in current session** — If a plan was created during this session, use it
3. **Scan `product/plans/todo/`** — If one plan exists, use it. If multiple exist, pick the one with the lowest plan number. If none exist, error: "No plans found in product/plans/todo/"

## Three Types of Sub-Agents

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
- Wait for completion

**b) Launch verification sub-agent**
- Wait for result

**c) Handle verification result**

- **PASS** → Proceed to commit
- **CODE FAILURE** → Launch fix sub-agent with the log file path, then launch verification sub-agent again. Repeat up to 3 verify-fix cycles. If still failing after 3 cycles, STOP, send a push notification (see Notifications below), and inform the user.
- **INFRASTRUCTURE FAILURE** → STOP. Send a push notification (see Notifications below). Inform the user about the infrastructure issue and the log file path. Do NOT retry, do NOT ask questions.

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

Start the application so it can be tested:

```bash
./do run
```

This runs in the background. The application will be accessible via the public HOST URLs (see Runtime Environment in CLAUDE.md).

Determine which `$HOST1`–`$HOST4` URLs are relevant for this project by reading CLAUDE.md's Runtime Environment section.

### 6. Create Pull Request (GitHub only)

Check if `GITHUB_TOKEN` is set in the environment. If so, create a pull request using the GitHub REST API.

**Determine the repo:** Parse the `origin` remote URL to extract `owner/repo`.

**Generate PR body from the plan AND the runtime summary.** The PR body must include everything a reviewer needs — the same information shown in the CLI summary. Specifically:

```
## Summary

[from plan overview]

## Live URLs

The application is running and can be tested at:
- **Frontend:** $HOST1 (or whichever is relevant)
- **API:** $HOST2 (or whichever is relevant)

## Acceptance Criteria

[from plan — as a checklist, all checked]

## Phases

- [x] Phase 1: ...
- [x] Phase 2: ...
```

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

### 7. Send Completion Notification

Send a push notification that everything is done:

```bash
curl -s -H "Title: Implementation Complete" -H "Tags: white_check_mark" \
  -d "Plan [plan-name] fully implemented. Branch: [branch]. PR: [PR-url-or-'no PR']" \
  ntfy.sh/${NTFY_TOPIC:-robertscodeagents101}
```

### 8. Summary

Print the same summary to the CLI that was included in the PR body:
- List of phases completed
- List of commits created
- Branch name
- PR URL (if created)
- Application URLs (from `$HOST1`–`$HOST4` environment variables, whichever are relevant)
- Confirmation that the plan is fully implemented and the app is running

**The CLI summary and the PR body must contain the same information.** The PR is the permanent record — anyone looking at the PR should see exactly what the implementer saw.

## Notifications

Send push notifications via ntfy.sh at terminal states so the user knows to come back. Use the `dev:notify-user` skill for full details. The orchestrator sends notifications directly (not via sub-agents).

**When to notify:**

| Event | Command |
|-------|---------|
| All phases complete, PR created | `curl -s -H "Title: Implementation Complete" -H "Tags: white_check_mark" -d "Plan [plan-name] fully implemented. Branch: [branch]. PR: [url]" ntfy.sh/${NTFY_TOPIC:-robertscodeagents101}` |
| Fix cycles exhausted (3 cycles, still failing) | `curl -s -H "Title: Claude Code - Blocked" -H "Priority: high" -H "Tags: x" -d "Phase [N] of [plan-name] failed after 3 fix cycles. Need your help. Check terminal." ntfy.sh/${NTFY_TOPIC:-robertscodeagents101}` |
| Infrastructure failure | `curl -s -H "Title: Claude Code - Infra Error" -H "Priority: high" -H "Tags: rotating_light" -d "Infrastructure failure during [plan-name] phase [N]: [brief description]. Execution stopped." ntfy.sh/${NTFY_TOPIC:-robertscodeagents101}` |
| `./do run` fails | `curl -s -H "Title: Claude Code - App Start Failed" -H "Priority: default" -H "Tags: warning" -d "[plan-name]: ./do run failed. PR created but app not running. Check terminal." ntfy.sh/${NTFY_TOPIC:-robertscodeagents101}` |
| `git push` fails | `curl -s -H "Title: Claude Code - Push Failed" -H "Priority: default" -H "Tags: warning" -d "[plan-name]: git push failed. Commits are local. Check terminal." ntfy.sh/${NTFY_TOPIC:-robertscodeagents101}` |

**Do NOT notify for:** individual phase completions, successful intermediate verification, or routine progress.

## Critical Rules

### The Orchestrator NEVER:

- **Asks questions** — NEVER use `AskUserQuestion`. This is a fully headless process. All operational decisions have predefined behaviors (see below). No exceptions.
- **Runs `./do check`** — Delegate to a verification sub-agent
- **Runs any shell commands** — Except `git` commands for branch/commit operations, `curl` for PR creation, `./do run` to start the app, and `mv` for moving the plan file
- **Implements code changes** — Delegate to implementation sub-agents
- **Fixes errors** — Delegate to fix sub-agents
- **Skips verification** — Every phase must pass `./do check` before committing

### The Orchestrator ALWAYS:

- **Creates a feature branch** from main before starting
- **Resumes from where it left off** if the branch already exists
- **Delegates implementation** to sub-agents
- **Delegates verification** to sub-agents
- **Delegates error fixing** to sub-agents
- **Creates commits** after verification passes
- **Sends push notifications** at terminal states (complete, blocked, infrastructure failure) via ntfy.sh
- **Stops on infrastructure failures** — Full stop, send notification, inform the user, do NOT ask or retry
- **Creates small, focused commits** — One per phase, not batched
- **Amends the last commit** to include the plan move to `done/`
- **Pushes the branch** after all phases complete
- **Starts the application** with `./do run` before creating the PR
- **Creates a PR** via GitHub REST API if `GITHUB_TOKEN` is available, with live URLs in the body
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
| `./do check` fails with code errors | Launch fix sub-agent. Retry verification. Repeat up to 3 cycles, then full stop. **Send push notification.** |
| `./do check` fails with infrastructure errors (DNS, network, permissions, missing tools) | Full stop. Inform the user what happened. Do NOT retry. **Send push notification.** |
| `./do run` fails | Inform user of the error. Continue to PR creation — the failure is noted in the summary. **Send push notification.** |
| Multiple plans in `product/plans/todo/` | Pick the one with the lowest plan number. |
| No plans in `product/plans/todo/` | Error: "No plans found in product/plans/todo/". Stop. |
| Branch already exists | Resume: check git log for completed phases, skip them, continue from next. |
| `git push` fails | Inform user. Do NOT retry. The commits are local — they can push manually. **Send push notification.** |
| Fix sub-agent cannot resolve errors after 3 verify-fix cycles | Full stop. Inform user what failed and which phase. Do NOT keep retrying. **Send push notification.** |
| Ambiguous or unclear plan instructions | Implement the most literal, straightforward interpretation. Do NOT ask for clarification. |
