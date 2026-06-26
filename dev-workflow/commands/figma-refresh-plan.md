# Figma Refresh Plan Command

Pull the current Figma design into the repo, maintain a per-piece
implementation-status ledger, and write the **UI plan** from that ledger — the
phases for every piece that is open (not yet implemented against the current
design). This is the single source of UI plans, first build and later rework
alike.

This command runs **after `/dev:plan`**, which resolves the Figma mapping and
design-system spec it depends on. It does the pixels → ledger → plan bridge:

- **Pulling the design and maintaining the ledger always happens.**
- **A UI plan is written for every open piece** (`to-implement` / `to-review`),
  in dependency order. The only time no plan is written is when nothing is open
  (everything implemented and unchanged).

There is **no automatic refresh inside `/implement`** — refreshing is always
this explicit step, and `/dev:implement` only executes plans.

## Where this sits in the workflow

```
1. /dev:plan                 Resolves the Figma mapping + design-system spec
                             (naming convention, responsive rules) into the
                             UI/UX Spec, and plans the technical groundwork
                             (env, framework, routing, architecture). Does NOT
                             write per-piece UI phases.

2. /dev:figma-refresh-plan   Pull the design (using the mapping), create/update
                             the ledger, and write the UI plan from it — first
                             run: all open pieces; later runs: only what changed.

3. /dev:implement            Executes the plans; writes the ledger back to
                             "implemented".

4. /dev:figma-refresh-plan   Later, design changed: pull fresh, diff via the
                             ledger, write a rework plan for the changed
                             (and dependent) pieces.

5. /dev:implement            Executes the rework plan; writes the ledger back.
```

## Input

Find what to refresh using this priority:

1. **Figma URL argument** — If the user passes a Figma URL and there is no
   current plan context, run in **ad-hoc mode** (see below).
2. **Current plan** — If a plan exists in `product/plans/todo/` or `done/` with
   a UI/UX Spec that references Figma, use its References and its confirmed
   mapping table.
3. **No Figma reference anywhere** — Report: "No Figma reference found. The
   feature has no Figma-sourced design — nothing to refresh." Stop.

The plan's UI/UX Spec is the source of the **mapping** (which Figma nodes are
which logical pieces, at which breakpoint, and which frames to ignore). The
mapping is resolved with the user during `/dev:plan` step 8 — this command does
not re-resolve it; it consumes it. If the mapping is missing for a referenced
Figma file, report that `/dev:plan` must establish it first, and stop.

## Prerequisites

- The **Figma MCP server** must be available. If it is not, report clearly and
  stop — do not guess at design state.
- **Code Connect** is used for component identity where mappings exist (requires
  the project's Figma Organization/Enterprise plan with a Dev seat). Where a
  component is not mapped in Code Connect, fall back to the plan's confirmed
  mapping.

## Process

### 1. Cheap change gate

Read the last pulled file version from `product/design/<plan-slug>/source.md`
(if it exists). Fetch the current Figma file-level `version` via the Figma MCP
(`get_metadata` on the file). If the version is unchanged since the last pull:

- Report "No design changes since last refresh." 
- Make **no commit** and write **no plan**. Stop.

Figma exposes no per-node version, only this file-level one — it is purely an
early-out. If it changed (or there is no prior snapshot), continue.

### 2. Pull the current design

Using the Figma MCP, for the mapped nodes:

- `get_code_connect_map` — authoritative Figma→code identity for mapped
  components. Record the code target for each mapped piece.
- `get_design_context` (and node subtrees) — structure, layout, code hints per
  node. This is also the source for the per-node hash (step 4).
- `get_variable_defs` — design tokens (colors, typography, spacing, radii).
- `get_metadata` — page/frame names and sizes (used only as a cross-check
  against the plan's mapping; the mapping itself is authoritative).

### 3. Materialize the design files

Write into `product/design/<plan-slug>/`:

| File | Contents |
|------|----------|
| `source.md` | Figma file key + URL, mapped node IDs, file `version`, fetched-at timestamp |
| `context.md` | Design context per piece (structure/layout/code hints from `get_design_context`) |
| `tokens.json` | Design tokens in **W3C DTCG format** (`$value` / `$type` / `$description`), generated from `get_variable_defs` — the raw Figma-named tokens plus any derived responsive tokens (see below) |
| `status.json` | The per-piece ledger (see schema below) |

Do **not** store screenshots — the `/dev:implement` visual-review loop captures
live screenshots via Playwright when needed; stored Figma images bloat the repo
and are unreliable as a hash source.

**Responsive tokens.** Figma exposes tokens as discrete named styles
(`body-lg-regular`, `body-md-regular`, …); the responsive relationship lives in
the **naming convention** captured in the UI/UX Spec, not in a single value.
Using that convention and the spec's **responsive scaling** rules:

- Parse the token names to group families and their size steps.
- **Stepped families:** keep the discrete named tokens. The per-breakpoint
  mapping in the spec tells `/dev:implement` which named token to apply at each
  breakpoint.
- **Fluid families:** derive a `clamp(min, preferred, max)` token from the
  family's min/max steps over the spec's viewport range, and write it into
  `tokens.json` alongside the raw named tokens (e.g. `body-regular-fluid`). The
  implementer uses the derived token verbatim.

If the spec defines no responsive rule for a family, keep its named tokens
as-is and report it as a gap rather than guessing a scaling behavior.

### 4. Compute per-piece hashes and refresh the ledger

For each logical piece in the mapping, compute a stable hash over its node's
serialized design JSON from `get_design_context` (a SHA-256 over the
canonicalized node properties — layout, styles, token references, children
structure). This is the change-detection primitive, because Figma has no native
per-node version.

Then update `status.json`:

- **New piece** (not in the ledger): add it with `status: "to-implement"` and
  `lastImplementedHash: null` — **unless** it is on the plan's adoption `matches`
  list (see "Adoption seeding" below), in which case seed it as `implemented`
  with `lastImplementedHash` set to its current hash.
- **Direct change** (current hash ≠ `lastImplementedHash`): set
  `status: "to-implement"`.
- **Unchanged** (current hash == `lastImplementedHash`): leave
  `status: "implemented"`.
- **Soft cascade**: after the direct changes are marked, for every piece that
  (transitively) `dependsOn` a piece now flagged `to-implement`, set its status
  to `to-review` — *unless* it is itself directly changed (then `to-implement`
  wins). `to-review` means "a dependency changed; a human/implementer must
  decide whether rework is actually needed" (well-built frontends pass tokens
  through, so a token change often needs no per-view rework).

Dependency edges (`dependsOn`) come from instance → main-component references
(a view/component depends on the primitives/components it instantiates) and from
token references (a primitive/component depends on the tokens it uses). Build
them from the pulled design context.

#### `status.json` schema

```json
{
  "figmaFileKey": "abc123",
  "figmaUrl": "https://www.figma.com/design/abc123/...",
  "fileVersion": "987654321",
  "fetchedAt": "2026-06-24T10:00:00Z",
  "pieces": [
    {
      "id": "login.view.mobile",
      "level": "view",
      "breakpoint": "mobile",
      "figmaNodeIds": ["123:45"],
      "codeTarget": "src/pages/Login.tsx",
      "lastImplementedHash": "sha256:...",
      "currentHash": "sha256:...",
      "status": "to-implement",
      "dependsOn": ["button.primitive", "color.primary.token"],
      "acceptedHash": null,
      "acceptedBy": null,
      "acceptedAt": null
    }
  ]
}
```

- `level`: one of `token`, `primitive`, `component`, `layout`, `view`.
- `breakpoint`: `desktop` / `tablet` / `mobile` for any piece that has
  breakpoint-specific variants — **views, layouts, AND components** (components
  are responsive and rendered at multiple breakpoints). `null` for pieces that
  don't vary by breakpoint (typically tokens and simple primitives). Each
  breakpoint is a separate piece, so only the changed breakpoint flips.
- `codeTarget`: from Code Connect where mapped, else the plan's fallback mapping,
  else `null`.
- `status`: `implemented` | `to-implement` | `to-review`. This is the
  *implementation* status only.
- `acceptedHash` / `acceptedBy` / `acceptedAt`: the human sign-off record, set by
  `/dev:figma-accept` — a separate dimension from `status`. A piece is **accepted**
  only when `acceptedHash == lastImplementedHash`. A built piece whose
  `acceptedHash != lastImplementedHash` is **awaiting-acceptance**. Acceptance is
  hash-bound, so any rebuild after a design change re-opens it automatically.
  This command never sets acceptance; it only preserves existing acceptance
  records and recomputes whether they still match.

#### Adoption seeding (existing / in-progress projects)

Normally the first pull has no baseline, so every piece is `to-implement`. When a
project is being adopted into the methodology, `/dev:plan`'s base-straightening
pass produces an **adoption `matches` list** — the pieces whose existing code
already matches the current design (recorded in the plan). On the first ledger
build, seed each piece on that list as `status: "implemented"` with
`lastImplementedHash` set to its current hash, instead of `to-implement`.

This makes the ledger reflect reality from the start: already-correct pieces are
not re-flagged, and only the `refactor` / `rebuild` / `missing` pieces show as
open. Pieces not on the list follow the normal rules. Adoption seeding applies
only on the first build for that plan; afterwards the project is on the normal
lifecycle.

### 5. Write the UI plan from the ledger

Collect the **open pieces**: everything with `status: "to-implement"` or
`status: "to-review"`. (Adoption-seeded `implemented` pieces are not open, so
they get no phase — exactly the intended effect.)

- **If nothing is open** (every piece `implemented` and unchanged): write **no
  plan**. Commit the design/ledger snapshot if anything changed, then report
  "nothing to do."
- **Otherwise**: write the UI plan (step 6). This is the same whether it's the
  first build (all pieces open) or a later rework (only changed pieces open) —
  the ledger drives it either way.

This is the single source of UI plans. `/dev:plan` does not write per-piece UI
phases; it only resolves the mapping + design-system spec and plans the
non-UI technical groundwork.

### 6. Write the UI plan

Determine the next plan number by scanning `product/plans/todo/` and
`product/plans/done/` for the highest number; use the next one.

Write `product/plans/todo/XXX-PLAN-FIGMA-UI-<feature>.md` (or `-REWORK-` for a
later diff run) using the plugin's plan format. Specifics:

- **Acceptance Criteria**: each open piece must visually match the current
  design.
- **Phase ordering**: dependency order — `token` (design system) → `primitive`
  (with browseable preview) → `component` → `layout` → `view×breakpoint`. Lower
  levels first so the design system exists before anything references it.
- **One phase per piece** (or per tightly-coupled small group). Each phase:
  - `**Type:** Frontend`
  - **Design Notes** pointing at the design files
    (`product/design/<plan-slug>/context.md`, `tokens.json`) and the specific
    `figmaNodeIds` / `codeTarget` from the ledger.
  - States to verify (per the UI/UX Spec).
- **`to-review` pieces**: include them as phases too, but mark them clearly as
  *"Review only — a dependency changed; confirm this piece still matches and
  re-implement only if needed."* The implementer decides; if no change is
  needed it just confirms and marks the ledger.
- **Do NOT add phases for `awaiting-acceptance` pieces.** Those are built and
  unchanged — there is nothing to rebuild; they are a human sign-off gap,
  surfaced in the report and resolved via `/dev:figma-accept`, not the UI plan.
- Note in the plan overview which file version this rework targets and which
  pieces are `to-implement` vs `to-review`.

### 7. Commit

Make a **dedicated snapshot commit**, separate from any implementation phase
commits:

```bash
git add product/design/<plan-slug>/ product/plans/todo/   # plan dir only if a UI plan was written
git commit -m "Design snapshot: <feature> (figma file v<version>)"
```

**Skip the commit entirely if nothing changed** (the step 1 gate should already
have caught the no-change case; if step 4 found no piece changes either, do not
create a no-op commit).

### 8. Report

Print a concise summary:

- File version pulled and fetched-at.
- Counts: `to-implement`, `to-review`, `awaiting-acceptance` (built but not
  signed off), `accepted`.
- Whether a UI plan was written (and its path), or "nothing to do" when no piece
  is open.
- The next step: run `/dev:implement` to execute the plan(s).
- The design files path.

## Ledger write-back (performed by `/dev:implement`, documented here)

When `/dev:implement` completes a frontend piece, it updates that piece in
`status.json`: sets `lastImplementedHash` to the piece's `currentHash` and
`status` to `implemented`. A `to-review` piece that the implementer confirms
needs no change is likewise set back to `implemented` (its hash updated to
current). This is what makes the next refresh's diff meaningful.

## Ad-hoc mode (free Figma URL, no plan)

When invoked as `/dev:figma-refresh-plan <figma-url>` with no plan context:

- Pull the design and write `source.md`, `context.md`, `tokens.json` under
  `product/design/_adhoc/<figma-node>/`.
- Do **not** maintain a ledger and do **not** write a rework plan — there is no
  mapping and no baseline. This is a raw fetch/snapshot for exploration.
- Commit as `Design snapshot (ad-hoc): <figma-node>`.

## Conflict resolution vs `frontend-design`

For Figma-sourced designs, **Figma wins**. The implementer reproduces the Figma
state exactly; the `frontend-design` skill only fills what Figma leaves
unspecified (states, hover/focus, micro-copy, responsive behavior between the
defined breakpoints). If the Figma design appears to violate a design principle,
the implementer **reports** it (summary/PR) rather than changing it unilaterally.
This keeps `implemented` well-defined: a piece is `implemented` exactly when it
matches the current Figma state.

## Hardcoded Operational Decisions

This command is headless where possible. Predefined behaviors:

| Scenario | Behavior |
|----------|----------|
| Figma MCP unavailable | Report clearly. Stop. Do NOT guess design state. |
| File version unchanged since last pull | Report "no changes". No commit, no plan. Stop. |
| No Figma reference in plan / no plan and no URL | Report "nothing to refresh". Stop. |
| No mapping yet (`/dev:plan` not run for this Figma file) | Report that `/dev:plan` must establish the mapping first. Stop. This is the ONLY case that points back to `/dev:plan`. |
| Component not mapped in Code Connect | Use the plan's confirmed fallback mapping for identity. |
| Open pieces exist (first build or later diff) | Pull + ledger + write the UI plan in `product/plans/todo/` + commit. Point to `/dev:implement`. |
| Nothing open (all implemented, unchanged) | Report "nothing to do". No plan; no commit unless the snapshot itself changed. |
| Ad-hoc free URL, no plan | Raw snapshot under `product/design/_adhoc/`. No ledger, no plan. |

## Critical Rules

1. **Runs after `/dev:plan`, never before.** It consumes the mapping +
   design-system spec that `/dev:plan` produces. If no mapping exists yet, stop
   and point to `/dev:plan`.
2. **Single source of UI plans.** This command writes the UI plan from the ledger
   for every open piece — first build and later rework alike. `/dev:plan` does
   not write per-piece UI phases. The only no-plan case is nothing open.
3. **The mapping is established in `/dev:plan`, consumed here.** Do not invent or
   re-resolve the Figma-to-piece mapping in this command.
3. **Figma is the source of truth** for sourced designs; deviations are reported,
   not applied.
4. **Per-breakpoint granularity** for views, layouts, AND components (all are
   responsive); only the changed breakpoint flips.
5. **Soft cascade**: direct change → `to-implement`; dependents → `to-review`.
6. **No no-op commits.** Skip the commit when nothing changed.
7. **Tokens in DTCG format.** Generate `tokens.json` as W3C DTCG even though the
   project does not use Tokens Studio, for interop.
8. **No screenshots stored.** Visual checking happens live in `/dev:implement`.
