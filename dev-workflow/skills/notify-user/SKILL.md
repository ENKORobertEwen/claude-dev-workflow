---
name: notify-user
description: Use when a long-running process completes, fails, or needs user attention — especially during plan execution, background builds, or multi-phase implementations where the user may be away from their terminal
---

# Notify User via ntfy.sh

## Overview

Send push notifications to the user's phone/desktop when important events occur. Uses [ntfy.sh](https://ntfy.sh) — a simple HTTP-based pub/sub notification service.

## When to Use

- Long-running implementation finishes (all phases complete)
- Verification (`./do check`) fails after exhausting fix cycles
- Infrastructure failure stops execution
- Any situation where you're blocked and need user input
- Background process completes or errors

**Do NOT notify for:** routine progress updates, individual phase completions, or successful intermediate steps. Only notify at terminal states (done, blocked, or failed).

## Configuration

**Topic:** `robertscodeagents101`

To change the topic, set the environment variable `NTFY_TOPIC`. Falls back to the hardcoded default above.

## How to Send

```bash
# Basic notification
curl -s -d "Your message here" ntfy.sh/${NTFY_TOPIC:-robertscodeagents101}

# With title and priority
curl -s \
  -H "Title: Claude Code" \
  -H "Priority: default" \
  -d "Your message here" \
  ntfy.sh/${NTFY_TOPIC:-robertscodeagents101}

# High priority (for errors/failures)
curl -s \
  -H "Title: Claude Code - Action Required" \
  -H "Priority: high" \
  -H "Tags: warning" \
  -d "Your message here" \
  ntfy.sh/${NTFY_TOPIC:-robertscodeagents101}
```

## Message Templates

**Implementation complete:**
```bash
curl -s -H "Title: Implementation Complete" -H "Tags: white_check_mark" \
  -d "Plan [plan-name] fully implemented. Branch: [branch]. PR: [url]" \
  ntfy.sh/${NTFY_TOPIC:-robertscodeagents101}
```

**Verification failure (blocked):**
```bash
curl -s -H "Title: Claude Code - Blocked" -H "Priority: high" -H "Tags: x" \
  -d "Phase [N] failed after 3 fix cycles. Need your help. Check terminal." \
  ntfy.sh/${NTFY_TOPIC:-robertscodeagents101}
```

**Infrastructure failure:**
```bash
curl -s -H "Title: Claude Code - Infra Error" -H "Priority: high" -H "Tags: rotating_light" \
  -d "Infrastructure failure: [brief description]. Execution stopped." \
  ntfy.sh/${NTFY_TOPIC:-robertscodeagents101}
```

**Needs user input:**
```bash
curl -s -H "Title: Claude Code - Input Needed" -H "Priority: default" -H "Tags: question" \
  -d "Waiting for your input: [brief context]" \
  ntfy.sh/${NTFY_TOPIC:-robertscodeagents101}
```

## Integration Points

When using this skill alongside other skills:

- **`/dev:implement`** — Notify on: all phases complete, fix cycles exhausted, infrastructure failure
- **`superpowers:executing-plans`** — Notify on: plan execution complete, checkpoint failure
- **`superpowers:subagent-driven-development`** — Notify on: all tasks complete, unrecoverable error

## Quick Reference

| Event | Priority | Tags |
|-------|----------|------|
| Implementation complete | default | white_check_mark |
| Verification blocked | high | x |
| Infrastructure failure | high | rotating_light |
| Needs user input | default | question |

## Common Mistakes

- **Over-notifying:** Don't send per-phase notifications. Only terminal states.
- **Missing env fallback:** Always use `${NTFY_TOPIC:-robertscodeagents101}` with the fallback.
- **Forgetting `-s` flag:** Use `curl -s` to suppress progress output that clutters logs.
- **Vague messages:** Include what happened, which plan/phase, and what the user should do.
