# Figma Refresh Plan Command

Pull the current Figma design into the repo, maintain a per-piece
implementation-status ledger, and — when there is already an implemented
baseline — write a rework plan for everything that has drifted from the design.

This command is **adaptive**:

- **Pulling the design and maintaining the ledger always happens.**
- **Writing a rework plan only happens when an implemented baseline exists.** On
  the first/greenfield pull there is no technical UI to plan rework against and
  nothing to diff, so the command just sets the baseline and points you at
  `/dev:plan`.

There is **no automatic refresh inside `/implement`** — refreshing is always
this explicit step.

## Where this sits in the workflow

```
1. /dev:figma-refresh-plan   First/greenfield run: pull design, create the
                             ledger (all pieces "to-implement"), commit. No
                             plan. Reports: "Design pulled — run /dev:plan."

2. /dev:plan                 Design-informed: resolves the Figma mapping into
                             the UI/UX Spec and plans the technical groundwork
                             plus the UI build against the pulled design.

3. /dev:implement            Builds it; writes the ledger back to "implemented".

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
  `lastImplementedHash: null`.
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
      "dependsOn": ["button.primitive", "color.primary.token"]
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
- `status`: `implemented` | `to-implement` | `to-review`.

### 5. Decide whether to write a rework plan (adaptive)

Determine whether an **implemented baseline** exists: any piece in the ledger
with `status: "implemented"` (i.e. something was built against the design
before).

- **No baseline** (first/greenfield pull — every piece is `to-implement` and
  none has ever been implemented): write **no plan**. Continue to commit, then
  report: *"Design pulled and baseline set. There is no implemented UI yet — run
  `/dev:plan` to plan the technical groundwork and the initial UI build against
  the design."*
- **Baseline exists** (design changed after prior implementation): write a
  rework plan (step 6).

### 6. Write the rework plan (baseline case only)

Determine the next plan number by scanning `product/plans/todo/` and
`product/plans/done/` for the highest number; use the next one.

Write `product/plans/todo/XXX-PLAN-FIGMA-REWORK-<feature>.md` using the plugin's
plan format (the same format `/dev:plan` produces). Specifics:

- **Acceptance Criteria**: each changed piece that must visually match the
  updated design.
- **Phase ordering**: dependency order — `token` → `primitive` → `component` →
  `layout` → `view×breakpoint`. Lower levels first so fixes cascade naturally.
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
- Note in the plan overview which file version this rework targets and which
  pieces are `to-implement` vs `to-review`.

### 7. Commit

Make a **dedicated snapshot commit**, separate from any implementation phase
commits:

```bash
git add product/design/<plan-slug>/ product/plans/todo/   # plan dir only if a rework plan was written
git commit -m "Design snapshot: <feature> (figma file v<version>)"
```

**Skip the commit entirely if nothing changed** (the step 1 gate should already
have caught the no-change case; if step 4 found no piece changes either, do not
create a no-op commit).

### 8. Report

Print a concise summary:

- File version pulled and fetched-at.
- Counts: `to-implement`, `to-review`, `implemented`.
- Whether a rework plan was written (and its path), or the "run /dev:plan"
  pointer for the first pull.
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
| Plan references Figma but has no confirmed mapping | Report that `/dev:plan` must establish the mapping first. Stop. |
| Component not mapped in Code Connect | Use the plan's confirmed fallback mapping for identity. |
| First/greenfield pull (no implemented baseline) | Pull + ledger + commit. No plan. Point to `/dev:plan`. |
| Baseline exists, pieces changed | Write rework plan in `product/plans/todo/`. |
| Baseline exists, nothing changed | Report "nothing to do". No commit, no plan. |
| Ad-hoc free URL, no plan | Raw snapshot under `product/design/_adhoc/`. No ledger, no plan. |

## Critical Rules

1. **Pulling always; planning only with a baseline.** Never write a rework plan
   on the first/greenfield pull — there is no UI to rework and nothing to diff.
2. **The mapping is established in `/dev:plan`, consumed here.** Do not invent or
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
