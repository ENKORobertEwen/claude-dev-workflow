---
name: verification-required
description: Enforces that ./do check must pass before any commit. Use whenever creating commits or finishing implementation work.
---

# Verification Required Skill

## Purpose

This skill enforces that `./do check` MUST pass before any commits are made. There are NO exceptions.

## Rules

### 1. `./do check` is MANDATORY

Before creating ANY commit, you MUST run `./do check` and it MUST pass completely. This is non-negotiable.

Always capture output to a unique log file using `tee`:

```bash
LOG_FILE="/tmp/do-check-$(date +%s)-$$.log"
./do check 2>&1 | tee "$LOG_FILE"
```

The log file is the source of truth. If verification fails, read the log file to diagnose errors — never re-run `./do check` just to see the output again.

### 2. Infrastructure Failures are a FULL STOP

If `./do check` fails due to infrastructure issues (network, permissions, missing tools, SSH, DNS, environment configuration, etc.), this is a **FULL STOP**. You CANNOT:

- Proceed with commits
- Skip `./do check` and use `./do lint` or `./do build` as a substitute
- Assume the code is correct because lint/build passed
- Work around the infrastructure issue

Instead, you MUST:

1. Stop all implementation work immediately
2. Inform the user that `./do check` failed due to infrastructure
3. Explain specifically what infrastructure issue occurred
4. Ask the user to fix the infrastructure issue or provide guidance
5. Wait for the user to confirm the infrastructure is fixed before continuing

### 3. Code Failures Must Be Fixed

If `./do check` fails due to code issues (lint errors, build errors, test failures), you MUST:

1. Fix all errors (or delegate to sub-agents to fix them)
2. Run `./do check` again
3. Repeat until `./do check` passes completely
4. Only then create commits

### 4. No Partial Verification

The following are NOT acceptable substitutes for `./do check`:

- `./do lint` alone
- `./do build` alone
- `./do test` alone
- Any combination that doesn't include the full `./do check` pipeline
- Any other project-specific commands that only cover part of the verification

`./do check` is the single source of truth for whether code is ready to commit.

## Rationale

The `./do check` command runs the full verification pipeline for the project. Partial checks are insufficient because:

1. Integration tests catch issues that unit tests miss
2. The full pipeline validates the complete build and deployment flow
3. Skipping `./do check` means shipping unverified code
