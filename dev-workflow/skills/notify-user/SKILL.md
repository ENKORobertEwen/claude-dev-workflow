---
name: notify-user
description: Use when a long-running process completes, fails, or needs user attention — especially during plan execution, background builds, or multi-phase implementations where the user may be away from their terminal
---

# Notify User via Bark

## Overview

Send push notifications to the user's iPhone when important events occur. Uses [Bark](https://github.com/Finb/Bark) — a simple URL-based iOS push notification service.

## When to Use

- Long-running implementation finishes (all phases complete)
- Verification (`./do check`) fails after exhausting fix cycles
- Infrastructure failure stops execution
- Any situation where you're blocked and need user input
- Background process completes or errors

**Do NOT notify for:** routine progress updates, individual phase completions, or successful intermediate steps. Only notify at terminal states (done, blocked, or failed).

## Configuration

**Bark Key:** `ykExqiREfYAbHToaf9cK5X`

To change the key, set the environment variable `BARK_KEY`. Falls back to the hardcoded default above.

## How to Send

```bash
# Basic notification
curl -s "https://api.day.app/${BARK_KEY:-ykExqiREfYAbHToaf9cK5X}/Claude%20Code/Your%20message%20here"

# With URL encoding for spaces and special chars — use --data-urlencode:
curl -s -G "https://api.day.app/${BARK_KEY:-ykExqiREfYAbHToaf9cK5X}/" \
  --data-urlencode "title=Claude Code" \
  --data-urlencode "body=Your message here"
```

## Message Templates

**Implementation complete:**
```bash
curl -s -G "https://api.day.app/${BARK_KEY:-ykExqiREfYAbHToaf9cK5X}/" \
  --data-urlencode "title=Implementation Complete" \
  --data-urlencode "body=Plan [plan-name] fully implemented. Branch: [branch]. PR: [url]"
```

**Verification failure (blocked):**
```bash
curl -s -G "https://api.day.app/${BARK_KEY:-ykExqiREfYAbHToaf9cK5X}/" \
  --data-urlencode "title=Claude Code - Blocked" \
  --data-urlencode "body=Phase [N] failed after 3 fix cycles. Need your help. Check terminal."
```

**Infrastructure failure:**
```bash
curl -s -G "https://api.day.app/${BARK_KEY:-ykExqiREfYAbHToaf9cK5X}/" \
  --data-urlencode "title=Claude Code - Infra Error" \
  --data-urlencode "body=Infrastructure failure: [brief description]. Execution stopped."
```

**Needs user input:**
```bash
curl -s -G "https://api.day.app/${BARK_KEY:-ykExqiREfYAbHToaf9cK5X}/" \
  --data-urlencode "title=Claude Code - Input Needed" \
  --data-urlencode "body=Waiting for your input: [brief context]"
```

## Integration Points

When using this skill alongside other skills:

- **`/dev:implement`** — Notify on: all phases complete, fix cycles exhausted, infrastructure failure
- **`superpowers:executing-plans`** — Notify on: plan execution complete, checkpoint failure
- **`superpowers:subagent-driven-development`** — Notify on: all tasks complete, unrecoverable error

## Quick Reference

| Event | Title |
|-------|-------|
| Implementation complete | Implementation Complete |
| Verification blocked | Claude Code - Blocked |
| Infrastructure failure | Claude Code - Infra Error |
| Needs user input | Claude Code - Input Needed |

## Common Mistakes

- **Over-notifying:** Don't send per-phase notifications. Only terminal states.
- **Missing env fallback:** Always use `${BARK_KEY:-ykExqiREfYAbHToaf9cK5X}` with the fallback.
- **Forgetting `-s` flag:** Use `curl -s` to suppress progress output that clutters logs.
- **Broken URL encoding:** Use `--data-urlencode` instead of manual `%20` encoding for messages with spaces/special chars.
- **Vague messages:** Include what happened, which plan/phase, and what the user should do.
