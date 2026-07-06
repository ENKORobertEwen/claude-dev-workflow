---
name: verification-required
description: Enforces that ./do check must pass before any commit, except commits whose changed files ALL match check-exempt paths (product/** plus project-declared globs). Use whenever creating commits or finishing implementation work.
---

# Verification Required Skill

## Purpose

This skill enforces that `./do check` MUST pass before any commits are made. The single exemption is defined mechanically in rule 0 — everything else has NO exceptions.

## Rules

### 0. The ONLY exemption: check-exempt commits

`./do check` is a pure function of its inputs. If no changed file is an input to the check pipeline, the result cannot have changed — re-running it is provably redundant. That is the only basis for skipping it, and it is decided by path match, never by judgment:

1. Determine the exact file set of the commit: `git diff --cached --name-only` (stage everything first, then decide).
2. The commit is exempt **iff EVERY file** matches a check-exempt glob:
   - **Plugin default:** `product/**` — plans, DDD docs, ADRs, design snapshots and the design ledger. The build never consumes these; commands transcribe them into code, and that transcription is a separate (checked) commit.
   - **Project additions:** the globs listed in the project CLAUDE.md under `### Check-Exempt Paths`. Additive only — a project can widen the exemption, never narrow `./do check` itself.
3. **Fail-closed.** One non-matching file → full `./do check`. No CLAUDE.md section, malformed globs, any doubt about a path → full `./do check`. Mixed commits are never exempt — do not split a commit just to dodge verification of the code part; the code part still gets its full check.
4. **Judgment is not a criterion.** "Small", "trivial", "docs-only-ish", "doesn't touch the logic", "can't possibly break anything" — none of these qualify. Only the path match counts.
5. **Cheap guard for machine-readable files.** If the exempt commit contains `.json`/`.yaml`/`.yml` files (e.g. `product/design/status.json`), verify they parse before committing (`jq empty <file>`, or the YAML equivalent). A malformed ledger breaks the workflow even though it never touches the build.

When a commit is exempt, say so explicitly in your report ("check-exempt: all N files under product/**") — silence looks like a skipped check.

### 1. `./do check` is MANDATORY

Before creating ANY non-exempt commit (rule 0), you MUST run `./do check` and it MUST pass completely. This is non-negotiable.

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

The check-exempt rule (rule 0) does not weaken this: it only skips runs whose result is already known, because none of the commit's files feed the pipeline. The assumption behind the `product/**` default is that the build never reads `product/` directly — the methodology transcribes design/docs content into code via commands, and those code commits are fully checked. If a project breaks that assumption (its build consumes `product/` files at build time), it must not rely on the default exemption.
