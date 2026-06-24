# Figma Refresh Skill — Design

**Status:** Design complete pending final user review, then writing-plans.
**Date:** 2026-06-24

## Problem

The dev plugin treats frontend design as a first-class concern: a plan's UI/UX
Spec can reference a Figma file, and `/implement` builds against it in frontend
mode. But a Figma design changes over time, and across iterative implement runs
individual pieces drift out of sync. We need to always know which design pieces
are implemented against the current Figma state and which are not — without an
expensive "recompute everything" command.

## Goal

A `dev:figma-refresh` skill that re-pulls the current Figma design, maintains a
layered, dependency-aware per-piece implementation-status ledger, and uses the
fresh design for the implement round in progress. The ledger gives a standing
"what's open" overview.

## Build on standards, not reinvented parts

Research (mid-2026) confirmed which parts are already solved; we consume those
and only build the genuinely missing glue.

- **Component identity ← Figma Code Connect** (the user is on Org/Enterprise
  with Dev seats, so it is available). Exposed via the Figma MCP
  (`get_code_connect_map`, `list_code_components`, `get_code_component_info`).
  Reliable **only for components the team explicitly mapped** — no auto
  inference. So: Code Connect is the **primary** identity source; for unmapped
  nodes we **fall back** to "Claude proposes a mapping from structure / frame
  sizes / screenshots, user confirms in the plan."
- **Tokens ← W3C DTCG JSON.** The user does NOT use Tokens Studio, so the skill
  generates `tokens.json` itself from `get_variable_defs`, written in **DTCG
  format** (`$value`/`$type`/`$description`) for interop and a possible later
  switch to a Tokens-Studio git source.
- **Per-node change detection is DIY.** Figma exposes only **file-level**
  `version`/`lastModified` — there is NO per-node version/hash. So we compute
  our **own per-node hash** from the node JSON (`get_design_context` / node
  subtree), gated by the file-level `version` as a cheap "did anything change at
  all" early-out.
- **The status ledger driving an implement loop is a genuine gap** — no
  off-the-shelf tool (Builder.io Fusion, Anima, Locofy, Supernova, zeroheight)
  maintains a git-native per-piece implemented-vs-current ledger. This is the
  glue we build.

## Core model: a layered, dependency-aware status ledger

Design is built bottom-up (atomic design). The ledger mirrors that, so a change
to a low-level piece is fixed once and only cascades awareness upward.

```
Tokens / Design-System   (Figma Variables/Styles -> DTCG tokens.json)
   └─ Primitives          (Figma Components / Code Connect)
        └─ Components/Blocks
             └─ Layouts
                  └─ Views / Pages  (top-level frames, per breakpoint)
```

- One ledger per plan: `product/design/<plan-slug>/status.json`.
- Each entry = one design piece at any level, with: its identity (Code Connect
  id where mapped, else the plan-confirmed mapping id), the Figma node id(s),
  the self-computed hash last implemented against, and a status flag.
- **Tracking granularity:** per breakpoint for views (desktop/tablet/mobile are
  separate entries), and per piece at the token/primitive/component levels.
- **Dependency edges** come from instance → main-component references (which
  view/component uses which primitive) plus Code Connect identity.
- **Cascade (soft):** on refresh, a piece whose hash changed flips to
  `to-implement`. Pieces that *depend* on it flip to the softer `to-review`
  (not auto `to-implement`) — because well-built frontends pass tokens through
  (e.g. CSS variables), so a token change often needs no per-view rework. The
  human/implementer decides whether a `to-review` piece is actually affected.
- **Write-back:** when a piece is implemented against the current design, its
  last-implemented hash is updated and its flag set to `implemented`.

This subsumes a separate diff skill — the "what changed / what's open" view
falls out of the ledger (list `to-implement` and `to-review`).

## Figma-structure mapping (resolved in the plan, not guessed)

Figma's node tree carries no semantics (which frames are the same screen at
different breakpoints, which are flow/annotation frames to ignore, which layer
is a primitive). This is resolved once, with the human, during `/plan` step 8:

1. Claude calls `get_code_connect_map` (authoritative for mapped components),
   `get_metadata` (pages/frames with names + sizes — size hints breakpoint),
   `get_variable_defs` (tokens), and takes screenshots.
2. Claude **proposes** a layered mapping table (screens × breakpoints,
   primitives/components, and an explicit **ignore list** for flow/annotation
   frames), marking anything uncertain (e.g. "these buttons aren't Figma
   components — treat as primitives?").
3. The user confirms/corrects. The confirmed mapping is written into the plan's
   UI/UX Spec and is what the ledger keys on.

## Behavior

1. Resolve source: Figma URL + node IDs from the plan's UI/UX Spec (or a free
   URL for ad-hoc use).
2. Cheap gate: compare file-level `version`; if unchanged, report "no changes"
   and stop.
3. Pull fresh: Code Connect map, `get_design_context` / node subtrees,
   `get_variable_defs`, `get_metadata`.
4. Materialize design files under `product/design/<plan-slug>/`:
   - `source.md` — Figma URL, node IDs, file version, fetched-at
   - `context.md` — design context (structure/layout/code hints)
   - `tokens.json` — DTCG-format tokens
   - `status.json` — the layered ledger (no screenshots stored; the implement
     visual-review loop captures live screenshots via Playwright)
5. Update the ledger: recompute per-node hashes, flip changed pieces to
   `to-implement`, cascade dependents to `to-review`.
6. Commit: a dedicated snapshot commit (`Design snapshot: …`), separate from
   phase commits. **Skip entirely if nothing changed** (no no-op commits).
7. Hand off: the implementer (frontend mode) reads from
   `product/design/<plan-slug>/` and works `to-implement` / `to-review` pieces.

## Entry points

- **Automatic in `/implement`:** once per run, before the first frontend phase,
  when the plan references Figma.
- **Manual:** `/dev:figma-refresh [plan|url]`. With a plan: full behavior incl.
  ledger. With a **free Figma URL** and no plan: ad-hoc pull + commit under
  `product/design/_adhoc/<figma-node>/`, **no ledger**.

## Conflict resolution vs `frontend-design`

For Figma-sourced designs, **Figma wins**: reproduce the Figma state exactly;
`frontend-design` only fills what Figma leaves unspecified (states,
hover/focus, micro-copy, responsive behavior between defined breakpoints).
Noticed violations of design principles are **reported** in the summary/PR, not
applied unilaterally. (For Claude-generated designs with no Figma source,
`frontend-design` leads.) This keeps `implemented` well-defined: a piece is
`implemented` exactly when it matches the current Figma state.

## Edge cases

- No Figma reference (Claude-generated design): skill is a no-op / skipped.
- Figma MCP unavailable: report clearly, do not block implement.
- Nothing changed since last run: report "no changes", skip commit.
- Component unmapped in Code Connect: use the plan-confirmed fallback mapping.

## Decisions log

| Topic | Decision |
|-------|----------|
| Change tracking | Living, layered, dependency-aware per-piece ledger |
| Layers | tokens → primitives → components → layouts → views×breakpoint |
| Baseline | Since last implement run (last marked-implemented hash) |
| Cascade | Soft: direct hit `to-implement`, dependents `to-review` |
| Granularity | Per breakpoint for views; per piece at lower levels |
| Separate diff skill | Subsumed by the ledger |
| Component identity | Code Connect primary (Org/Enterprise available); Claude-proposed + user-confirmed mapping as fallback |
| Mapping resolution | In `/plan` step 8: Claude proposes, user confirms; explicit ignore list |
| Tokens | Self-generated from `get_variable_defs`, DTCG format (no Tokens Studio) |
| Hash source | Self-computed per-node hash from node JSON, gated by file `version` (no native per-node hash exists) |
| Ledger location | `product/design/<plan-slug>/status.json`, one per plan |
| Commit style | Dedicated snapshot commit, separate from phase commits; skip no-op |
| Files committed | `tokens.json`, `context.md`, `source.md`, `status.json` (no screenshots) |
| Manual scope | Plan-based + free Figma URL (ad-hoc, no ledger) |
| Auto trigger | Once per implement run, before the first frontend phase |
| Figma vs frontend-design | Figma wins; deviations reported, not applied |

## Open / to resolve during planning

- Exact `status.json` schema (field names; how dependency edges are stored).
- Exact write-back point: who marks a piece `implemented` and at which step of
  the phase cycle, and how the visual-review loop interacts with `to-review`.
- Whether to phase implementation (e.g. v1: tokens + views×breakpoint; v2:
  full primitive/component dependency graph) — to decide in writing-plans.
