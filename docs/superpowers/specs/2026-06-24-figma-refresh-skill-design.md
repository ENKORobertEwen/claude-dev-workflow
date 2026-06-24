# Figma Refresh Skill — Design

**Status:** Design approved for capture; further discussion pending before implementation.
**Date:** 2026-06-24

## Problem

The dev plugin now treats frontend design as a first-class concern: a plan's
UI/UX Spec can reference a Figma file, and `/implement` builds against it in
frontend mode. But a Figma design changes over time. The design captured when
the plan was written can be stale by the time implementation runs, so the
implementer may build against an outdated design.

## Goal

A skill that re-pulls the current Figma design context and uses it for the
implement round in progress, while leaving a git-native trail that a future
diff skill can build on.

## Scope (now vs later)

**Built now — `dev:figma-refresh`:**
- Pull fresh Figma context and use it for the current implement round.
- Commit the pulled design files so git history is the diff substrate.
- Two entry points: automatic in `/implement`, and a manual command.

**Deferred (NOT built now) — `dev:figma-diff`:**
- Compare design files between the last two design commits (essentially
  `git diff` over `product/design/<plan-slug>/`), interpret the changes, and
  flag affected screens/phases.
- Designed against now only so it docks in cleanly later. No diff code yet.

## Behavior

1. **Resolve the source** — Figma URL + node IDs from the current plan's UI/UX
   Spec (References section). The manual command may take a plan path or Figma
   URL as an argument instead.
2. **Pull fresh context** via the Figma MCP: `get_design_context`,
   `get_screenshot`, `get_variable_defs`, `get_metadata`, and the Code Connect
   map when present.
3. **Materialize design files** under `product/design/<plan-slug>/`:
   - `source.md` — Figma URL, node IDs, file version, fetched-at timestamp
   - `context.md` — design context (structure/layout/code hints)
   - `tokens.json` — variable definitions (colors, typography, spacing)
   - `screenshots/<screen>.png` — per-frame screenshots
4. **Commit** the files: `Design snapshot: <feature> (figma <node>)`. The git
   history of this directory is the diff source for the future skill.
5. **Hand off** — the implementer (frontend mode) reads the design from
   `product/design/<plan-slug>/` instead of fetching ad hoc.

## Entry points

- **Automatic in `/implement`:** once per implement run, before the first
  frontend phase (not per phase — the design does not change mid-run, and this
  saves MCP calls). Only when the plan's UI/UX Spec references Figma.
- **Manual:** `/dev:figma-refresh [plan|url]` — re-snapshot when the design is
  known to have changed, without starting a full implement run.

## Edge cases

- **No Figma reference (Claude-generated design):** no-op / skipped — the skill
  only applies to Figma-sourced designs.
- **Figma MCP unavailable:** report clearly, do not block implement (consistent
  with the plugin's non-blocking stance).

## Open questions (to resolve before implementation)

1. Path `product/design/<plan-slug>/` — fits the `product/` convention
   (DDD/ADR/plans). Confirm or adjust.
2. Auto-trigger granularity: once per run before the first frontend phase
   (recommended) vs. per frontend phase.

## Why git-commit over a custom snapshot file

A dedicated snapshot format would need its own diffing later. Committing the
design files directly makes git the versioning and diff mechanism for free —
text files (md/json) diff cleanly, screenshots are versioned as binaries. The
future diff skill becomes a thin interpretation layer over `git diff`.
