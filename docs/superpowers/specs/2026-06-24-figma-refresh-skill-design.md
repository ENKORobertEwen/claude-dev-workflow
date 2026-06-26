# Figma Refresh Plan Command — Design

**Status:** Implemented. v2.9.0 added commands/figma-refresh-plan.md +
plan.md/implement.md reconciliation; v2.10.0 added commands/figma-accept.md
(human acceptance dimension); v2.11.0 corrected the lifecycle to plan → refresh
→ implement and made figma-refresh-plan the single UI-plan source. Full scope.
**Date:** 2026-06-24 (acceptance 2026-06-26; lifecycle correction 2026-06-26)

> **Form:** This is a **command** (`/dev:figma-refresh-plan`,
> `commands/figma-refresh-plan.md`), not a skill — consistent with the plugin
> convention where `commands/` are invoked workflows and `skills/` are
> auto-activating guardrails. It outputs a **rework plan**, hence the `-plan`
> suffix. There is **no auto-refresh inside `/implement`** — refresh is an
> explicit, plan-producing step; `/implement` only executes plans.

## Problem

The dev plugin treats frontend design as a first-class concern: a plan's UI/UX
Spec can reference a Figma file, and `/implement` builds against it in frontend
mode. But a Figma design changes over time, and across iterative implement runs
individual pieces drift out of sync. We need to always know which design pieces
are implemented against the current Figma state and which are not — without an
expensive "recompute everything" command.

## Goal

A single `/dev:figma-refresh-plan` command that always re-pulls the current Figma
design, maintains a layered, dependency-aware per-piece implementation-status
ledger, and writes the **UI plan** from that ledger for every open piece.
`/dev:implement` executes plans and writes the ledger back to `implemented`.

Key insight (corrected): the command **runs after `/dev:plan`**, because it needs
the mapping `/dev:plan` resolves. `/dev:plan` does not need the pull (it inspects
Figma live to build the mapping). And the command is the **single source of UI
plans** — it always plans the open pieces from the ledger, whether that's the
first build (all open) or a later rework (only changed). The only no-plan case is
nothing open.

## Lifecycle and role split

```
1. /dev:plan                 Resolves the Figma MAPPING + design-system spec
                             (naming convention, responsive rules, breakpoints)
                             into the UI/UX Spec, and plans the NON-UI technical
                             groundwork (framework, env, routing, architecture).
                             Does NOT write per-piece UI phases. Inspects Figma
                             live for the mapping — no pull needed.

2. /dev:figma-refresh-plan   Pulls the design (using the mapping), creates/updates
                             the ledger + design files, and writes the UI plan
                             from the ledger: first run = all open pieces, later
                             runs = only what changed. Dependency order tokens ->
                             primitives -> components -> layouts -> views x bp.

3. /dev:implement            Executes the plans; on completing a frontend piece,
                             writes status.json back (lastImplementedHash + flag
                             = implemented).

4. /dev:figma-refresh-plan   Later, design changed: pull fresh, diff via the
                             ledger, write the rework plan for changed/dependent
                             pieces.

5. /dev:implement            Executes the rework plan, writes the ledger back.
```

This separates concerns: `/dev:plan` does the one-time mapping + groundwork; the
command is the repeatable design **pull + ledger + UI plan**; `/dev:implement` is
execution + ledger write-back. The command writes its own numbered plan file in
`product/plans/todo/` rather than editing other plans in place.

(Superseded earlier design: an "adaptive, plan-only-with-a-baseline" model where
`figma-refresh-plan` ran *before* `/dev:plan` on greenfield. That contradicted
the mapping dependency — the command needs the mapping `/dev:plan` produces — so
the order is now uniformly plan → refresh → implement, and the command is the
single UI-plan source.)

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
- **Tracking granularity:** per breakpoint for views, layouts, AND components
  (all are responsive; desktop/tablet/mobile are separate entries), and per
  piece at the token/primitive levels.
- **Every planning pass re-checks all five levels.** Projects are never "done"
  at the lower levels: design system and primitives change rarely, but
  components and pages are constantly added/changed (and a new component may
  need a new primitive/token). `/plan` must sweep all five for additions and
  changes and keep the mapping current; `/dev:figma-refresh-plan` enforces the
  same sweep via the ledger for pieces present in the mapping.
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
6. **Write the UI plan from the ledger** — collect the open pieces
   (`to-implement` / `to-review`):
   - **Open pieces exist** (first build = all open; later = only changed): emit
     the UI plan in `product/plans/todo/` with phases for each, in dependency
     order (tokens → primitives → components → layouts → views×breakpoint), using
     the plugin's plan format (frontend phases with Design Notes pointing at the
     design files). Single source of UI plans.
   - **Nothing open** (all implemented, unchanged): write no plan, report
     "nothing to do."
7. Commit: a dedicated snapshot commit (`Design snapshot: …`) for the design
   files + ledger (+ rework plan, when emitted), separate from implementation
   phase commits. **Skip entirely if nothing changed** (no no-op commits).

`/dev:implement` later executes the plan and, per completed piece, writes back
to `status.json` (last-implemented hash + flag = `implemented`).

## Invocation

- **`/dev:figma-refresh-plan` (with a plan / current plan):** always pulls +
  updates the ledger + writes the UI plan for open pieces (see Behavior step 6).
  Runs after `/dev:plan`, which must have resolved the mapping first.
- **`/dev:figma-refresh-plan <figma-url>` (no plan):** ad-hoc pull + commit
  under `product/design/_adhoc/<figma-node>/`, **no ledger, no rework plan** —
  just fetch and snapshot.
- **No auto-refresh inside `/implement`.** Refresh is always an explicit step.

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

## Design-to-code guardrails (`/implement` frontend mode)

Governing rule: derive everything from the design artifacts; for anything not
derivable, report it — never invent. Lives in `implement.md` frontend mode.

- **Design system first:** the first frontend step/phase always establishes the
  theme/token layer (colors, spacing, text styles, radii) from `tokens.json`.
  Nothing downstream can reference a token that doesn't exist yet.
- **Responsive behavior is part of the design system, recorded as a contract.**
  Figma exposes tokens as discrete named styles (`body-lg-regular`, …); the
  responsive relationship lives in the **naming convention** + a **responsive
  scaling rule** captured in the UI/UX Spec during `/plan` (strategy per
  project: fluid or stepped). `figma-refresh-plan` parses the convention and,
  for fluid families, derives `clamp()` tokens from the family min/max into
  `tokens.json`; for stepped families it keeps the named tokens + the
  per-breakpoint mapping. `/implement` applies them strictly by the rule and
  never guesses which size belongs at which breakpoint.
- **Primitives second, with a browseable preview:** build primitives next and
  expose them (Storybook stories, or a dev route like `/__design`) in all their
  states, so the visual-review loop can check non-screen pieces. Add components
  to the same preview as built. Best-effort ("when feasible").
- **Tokens strict:** every value references a token; a value matching no token
  is a reported deviation, never a silent hardcode.
- **Reuse mapped components:** use the `codeTarget` (Code Connect / mapping);
  no parallel components; dependency order primitives → components → layouts →
  views.
- **Layout = intent:** Figma auto-layout → flex/grid, semantic and fluid; no
  absolute/canvas-pixel positioning; only the mapped breakpoints.
- **Semantic HTML + a11y:** semantic elements, keyboard access, visible focus,
  per the UI/UX Spec.
- **All states, real assets, verbatim copy:** implement every spec'd state;
  export assets (don't redraw); use exact copy (don't paraphrase).
- **Follow project conventions:** units, CSS approach, file structure from
  CLAUDE.md / existing code — not an imposed standard.
- **Narrow judgment exception:** only genuinely unspecified visual micro-details
  (transition timing, undefined hover/focus, missing micro-copy).

## Human acceptance (`/dev:figma-accept`)

Building a piece and passing the automated visual review is not the same as a
tester signing off. Acceptance is a **separate dimension** in the ledger, not a
`status` value:

- Per piece: `acceptedHash`, `acceptedBy`, `acceptedAt`.
- A piece is **accepted** only when `acceptedHash == lastImplementedHash` (the
  tester reviewed exactly what's built). Built but not signed off →
  **awaiting-acceptance**. Hash-bound, so a rebuild after a design change
  re-opens acceptance automatically.
- `/dev:implement` sets `implemented` but never touches acceptance.
- `/dev:figma-accept` records sign-off — two channels: a CLI command (explicit
  pieces, `--all`, interactive, or `--from-pr <n>` to sync a PR's acceptance
  checklist) and the acceptance checklist `/dev:implement` writes into the PR.
- **Soft tracking:** acceptance never blocks merges or plan completion;
  `awaiting-acceptance` pieces stay visible until accepted. The rework plan does
  not include them (nothing to rebuild — it's a sign-off gap).

## Adopting an existing / in-progress project

For a project already mid design-implementation (e.g. on an older flow), the
greenfield assumption breaks: code exists, but there is no ledger, mapping, or
clean design-system layer, and the ledger can't auto-know what already matches
the design. One-time base-straightening pass:

1. `/dev:plan` runs as a base-straightening pass: resolve mapping + design system
   as usual, then **audit existing code against the design across all five
   levels**, classify each piece (`matches` / `refactor` / `rebuild` /
   `missing`), and record the `matches` list (plus any non-UI groundwork). It
   does NOT write the per-piece UI remediation phases.
2. `/dev:figma-refresh-plan` pulls the design and builds the ledger, **seeding**
   the `matches` pieces as `implemented` (current hash) and the rest as
   `to-implement`, then writes the UI remediation plan for the open (non-matching)
   pieces.
3. `/dev:implement` executes the remediation, writing the ledger back.

After this one-time pass the project is on the normal lifecycle.

## Decisions log

| Topic | Decision |
|-------|----------|
| Form | A **command** `/dev:figma-refresh-plan` (commands/figma-refresh-plan.md), not a skill |
| Order | `/dev:plan` → `/dev:figma-refresh-plan` → `/dev:implement`. Refresh runs AFTER plan (needs the mapping plan resolves), never before |
| Output | Always: pull + ledger update + UI plan for open pieces (single source of UI plans; first build = all open, later = diff). No plan only when nothing is open |
| UI-plan source | `/dev:figma-refresh-plan` writes all UI plans from the ledger; `/dev:plan` writes only mapping + design-system spec + non-UI groundwork |
| Implement integration | **No auto-refresh**; /implement only executes plans and writes the ledger back |
| Change tracking | Living, layered, dependency-aware per-piece ledger |
| Layers | tokens → primitives → components → layouts → views×breakpoint |
| Baseline | Since last implement run (last marked-implemented hash) |
| Cascade | Soft: direct hit `to-implement`, dependents `to-review` |
| Granularity | Per breakpoint for views, layouts AND components (all responsive); per piece at token/primitive levels |
| Coverage | Every planning pass re-checks all five levels for additions/changes; lower levels are never assumed frozen |
| Separate diff skill | Subsumed by the ledger |
| Component identity | Code Connect primary (Org/Enterprise available); Claude-proposed + user-confirmed mapping as fallback |
| Mapping resolution | In `/plan` step 8: Claude proposes, user confirms; explicit ignore list |
| Tokens | Self-generated from `get_variable_defs`, DTCG format (no Tokens Studio) |
| Responsive tokens | Contract = Figma naming convention + per-project scaling rule (fluid/stepped) recorded in the UI/UX Spec; fluid derives `clamp()` tokens, stepped keeps named tokens + per-breakpoint mapping |
| Hash source | Self-computed per-node hash from node JSON, gated by file `version` (no native per-node hash exists) |
| Ledger location | `product/design/<plan-slug>/status.json`, one per plan |
| Commit style | Dedicated snapshot commit, separate from phase commits; skip no-op |
| Files committed | `tokens.json`, `context.md`, `source.md`, `status.json` (no screenshots) |
| Invocation | `/dev:figma-refresh-plan` (plan-based, incl. first pull) + free Figma URL (ad-hoc, no ledger/plan) |
| Figma vs frontend-design | Figma wins; deviations reported, not applied |
| Human acceptance | Separate ledger dimension (`acceptedHash`/`By`/`At`), hash-bound; `/dev:figma-accept` via CLI + PR checklist; soft (never blocks) |

## Open / to resolve during planning

- Exact `status.json` schema (field names; how dependency edges are stored).
- Exact write-back point: who marks a piece `implemented` and at which step of
  the phase cycle, and how the visual-review loop interacts with `to-review`.
- Whether to phase implementation (e.g. v1: tokens + views×breakpoint; v2:
  full primitive/component dependency graph) — to decide in writing-plans.
- **Reconcile `implement.md`:** the frontend-mode line added in v2.8.0 ("if the
  UI/UX Spec references a Figma file, use the Figma MCP to pull…") must change —
  `/implement` no longer pulls Figma. Instead it reads design files from
  `product/design/<plan-slug>/` (produced by `figma-refresh-plan`) and writes
  the ledger back on completion.
- **Update `plan.md` step 8** to make the Figma path resolve the *mapping*
  (screens×breakpoint, Code Connect primitives/components, ignore list) into the
  UI/UX Spec, and to note that pixels + frontend phases come from
  `figma-refresh-plan`.
