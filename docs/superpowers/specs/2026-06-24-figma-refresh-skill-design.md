# Figma Refresh Skill — Design

**Status:** Design discussed and refined. Pending final user review before
writing the implementation plan.
**Date:** 2026-06-24

## Problem

The dev plugin treats frontend design as a first-class concern: a plan's UI/UX
Spec can reference a Figma file, and `/implement` builds against it in frontend
mode. But a Figma design changes over time. The design captured when the plan
was written can be stale by the time implementation runs, and across iterative
implement runs individual components drift out of sync. We need to always know
which components are implemented against the current design and which are not.

## Goal

A `dev:figma-refresh` skill that re-pulls the current Figma design context,
maintains a per-component implementation-status ledger, and uses the fresh
design for the implement round in progress. The ledger gives a standing
overview of what is still open — without an expensive "recompute everything"
command.

## Core model: a living status ledger (not snapshot-diffing)

Rather than diffing two snapshots on demand, the skill maintains a per-component
ledger as the source of truth for implementation state.

- One ledger per plan: `product/design/<plan-slug>/status.json`.
- Each entry = one design component/element:
  - Figma node ID
  - design hash/version last implemented against (from Figma metadata)
  - status flag: `implemented` / `to-implement`
- **On refresh:** pull current Figma metadata, compare each component's hash to
  its last-implemented hash. Changed → flag flips to `to-implement`. Only
  changed elements flip; unchanged ones stay `implemented`. Cheap — a per-node
  hash compare, not a full re-check.
- **After a component is implemented** against the current design: update its
  last-implemented hash and set the flag to `implemented`.
- Result: `status.json` is always an accurate "what's open" overview.

Baseline for "what changed" is therefore implicitly **since the last implement
run** (the last time hashes were marked implemented).

This subsumes the previously-considered separate `figma-diff` skill: the diff
view falls out of the ledger directly (list `to-implement` components). No
separate diff skill is planned; a reporting view over the ledger is enough.

## Behavior

1. **Resolve the source** — Figma URL + node IDs from the current plan's UI/UX
   Spec (References). The manual command may take a plan path or a free Figma
   URL.
2. **Pull fresh context** via the Figma MCP: `get_design_context`,
   `get_metadata` (for per-node hash/version), `get_variable_defs`, Code
   Connect map when present.
3. **Materialize design files** under `product/design/<plan-slug>/`:
   - `source.md` — Figma URL, node IDs, file version, fetched-at
   - `context.md` — design context (structure/layout/code hints)
   - `tokens.json` — variable definitions (colors, typography, spacing)
   - `status.json` — the per-component ledger (node id, last-implemented hash,
     flag)

   Screenshots are NOT stored — the implement visual-review loop captures live
   screenshots via Playwright when needed; stored Figma images would bloat the
   repo and are unreliable as a hash source.
4. **Update the ledger** — compare current Figma metadata hashes to
   `status.json`; flip changed components to `to-implement`.
5. **Commit** — a dedicated snapshot commit (`Design snapshot: <feature>
   (figma <node>)`), separate from implementation phase commits. **Skip the
   commit entirely if nothing changed** (no no-op commits).
6. **Hand off** — the implementer (frontend mode) reads design from
   `product/design/<plan-slug>/` and works the `to-implement` components.

## Entry points

- **Automatic in `/implement`:** once per implement run, before the first
  frontend phase. Only when the plan's UI/UX Spec references Figma.
- **Manual:** `/dev:figma-refresh [plan|url]`.
  - With a plan (or current plan): full behavior incl. ledger.
  - With a **free Figma URL** and no plan: ad-hoc pull + commit under
    `product/design/_adhoc/<figma-node>/`, **no ledger flags** — just fetch and
    snapshot.

## Conflict resolution vs `frontend-design`

For Figma-sourced designs, **Figma wins**. The implementer reproduces the Figma
state exactly; `frontend-design` only fills what Figma leaves unspecified
(states, hover/focus, micro-copy, responsive behavior between defined
breakpoints). If the implementer notices the Figma design violates a design
principle, it **reports the deviation** in the summary/PR rather than changing
it unilaterally. (For Claude-generated designs with no Figma source,
`frontend-design` leads instead.)

This keeps "implemented" well-defined: a component is `implemented` exactly when
it matches the current Figma state.

## Edge cases

- **No Figma reference (Claude-generated design):** skill is a no-op / skipped.
- **Figma MCP unavailable:** report clearly, do not block implement.
- **Nothing changed since last run:** report "no changes", skip commit.

## Decisions log

| Topic | Decision |
|-------|----------|
| Change tracking | Living per-component status ledger, not on-demand snapshot diff |
| Baseline | Since last implement run (last marked-implemented hash) |
| Diff skill | Subsumed by the ledger; no separate skill |
| Ledger location | `product/design/<plan-slug>/status.json`, one per plan |
| Commit style | Dedicated snapshot commit, separate from phase commits |
| Files committed | `tokens.json`, `context.md`, `source.md`, `status.json` (no screenshots) |
| Hash source | Figma metadata (`get_metadata`) |
| Manual scope | Plan-based, and free Figma URL allowed (ad-hoc, no ledger) |
| No-op commits | Avoided — skip commit when nothing changed |
| Auto trigger | Once per implement run, before the first frontend phase |
| Figma vs frontend-design | Figma wins; deviations reported, not applied |

## Open / to decide before/while planning

- Exact shape of `status.json` (field names) and how components map to plan
  phases for "what to (re)implement".
- How the implementer marks a component `implemented` (who writes back to the
  ledger, and at which point in the phase cycle).
