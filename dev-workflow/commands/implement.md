# Implement Plan Command

You are the orchestrator. Your job is to execute a plan by delegating ALL work to sub-agents. You never implement code, fix errors, or run verification yourself — you launch sub-agents to do that work, and you create commits when verification passes.

## Input

The user provides a plan to implement. Find it using this priority:

1. **File path provided** — Use the specified file
2. **Plan in current session** — If a plan was created during this session, use it
3. **Scan `product/plans/todo/`** — If one plan exists, use it. If multiple exist, ask the user which one. If none exist, error: "No plans found in product/plans/todo/"

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

IMPORTANT: Always include the log file path in your response. Do NOT paste the full output — the log file is the source of truth.
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
- **CODE FAILURE** → Launch fix sub-agent with the error output, then launch verification sub-agent again. Repeat until passing.
- **INFRASTRUCTURE FAILURE** → STOP. Inform the user about the infrastructure issue. Wait for the user to resolve it before continuing.

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

### 5. Create Pull Request (GitHub only)

Check if `GITHUB_TOKEN` is set in the environment. If so, create a pull request using the GitHub REST API.

**Determine the repo:** Parse the `origin` remote URL to extract `owner/repo`.

**Generate PR body from the plan:** Use the plan's Overview section as the summary. List phases as completed checklist items. Include the acceptance criteria table.

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
    "body": "## Summary\n\n[from plan overview]\n\n## Acceptance Criteria\n\n[from plan]\n\n## Phases\n\n- [x] Phase 1: ...\n- [x] Phase 2: ...\n..."
  }'
```

**If `GITHUB_TOKEN` is not set:** Skip PR creation. Inform the user that the branch was pushed and they can create a PR manually. Mention that setting `GITHUB_TOKEN` enables automatic PR creation.

### 6. Start the Application

After the PR is created (or the branch is pushed), start the application so it can be tested:

```bash
./do run
```

This runs in the background. The application will be accessible via the public HOST URLs (see Runtime Environment in CLAUDE.md).

Include the accessible URLs in the summary so the user (or PR reviewers) can test immediately.

### 7. Summary

Provide a summary:
- List of phases completed
- List of commits created
- Branch name
- PR URL (if created)
- Application URLs (from `$HOST1`–`$HOST4` environment variables, whichever are relevant)
- Confirmation that the plan is fully implemented and the app is running

## Critical Rules

### The Orchestrator NEVER:

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
- **Stops on infrastructure failures** and asks the user for help
- **Creates small, focused commits** — One per phase, not batched
- **Amends the last commit** to include the plan move to `done/`
- **Pushes the branch** after all phases complete
- **Creates a PR** via GitHub REST API if `GITHUB_TOKEN` is available
- **Starts the application** with `./do run` after the PR so it's testable

### Sub-Agents NEVER:

- Run `./do check` (except verification sub-agents)
- Create commits
- Move plan files
- Push or create branches
