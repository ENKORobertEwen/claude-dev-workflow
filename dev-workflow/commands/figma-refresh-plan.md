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
  mapping. Code Connect is optional and commonly empty — never required.
- **Node.js** must be available — the deterministic ledger hash runs through the
  committed `product/design/bin/ledger-hash.mjs` script (step 4). If `node` is
  unavailable, stop with a clear message rather than improvising hashes.

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

- `get_metadata` — per mapped node: `id`, `name`, `type`, `width`, `height`, and
  variant properties (for component sets). **This is the source for the
  structural hash (step 4).** It is chosen over `get_design_context` because it
  is stable across runs — it contains only ids, names, sizes, and variant
  structure, none of the volatile content that breaks hashing.
- `get_design_context` (and node subtrees) — structure, layout, code hints, used
  ONLY as reference material for the implementer (written to `context.md`).
  **Never used for hashing:** its output is unstable run-to-run (it embeds a
  screenshot, freshly *generated* reference code, asset/download URLs, and
  absolute x/y canvas positions), so two pulls of an unchanged design produce
  different bytes.
- `get_variable_defs` — design tokens (colors, typography, spacing, radii).
  Source for token-piece hashes (step 4).
- `get_code_connect_map` — Figma→code identity for mapped components. **Optional**
  — commonly empty; never required for hashing or drift detection.

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

### 4. Compute per-piece hashes (deterministic)

Figma has no per-node version, so change detection relies on a per-piece
SHA-256 hash. **This hash MUST be byte-identical across sessions, runs, and MCP
response ordering** — otherwise an unchanged design reads as drift and the
baseline is destroyed. Determinism is therefore not described in prose and
re-derived each run; it is produced by a committed reference script that is the
single source of truth.

#### 4.1 The reference hash script (normative)

Current hash spec: **`hashSpecVersion: 1`** (`hashSpec: "node-metadata-structural-v1"`).

On every run, ensure `product/design/bin/ledger-hash.mjs` exists with **exactly**
the content below; if it is missing, materialize it (write it) and commit it. It
requires Node.js (ubiquitous in web/frontend projects); if `node` is unavailable,
stop with a clear message — do not fall back to ad-hoc hashing.

```js
#!/usr/bin/env node
// ledger-hash.mjs — normative ledger hash for the dev plugin's design ledger.
// hashSpecVersion: 1  (hashSpec: node-metadata-structural-v1)
//
// Reads ONE JSON object from stdin and prints "sha256:<hex>" of its canonical
// form. This script is the single source of truth for the hash: any session,
// run, or model that feeds the same object gets the same hash.
//
// Canonical form (deterministic, order-independent):
//   - objects: keys sorted ascending by Unicode code point; serialized as
//     {"k":canon(v),...} with no insignificant whitespace.
//   - arrays: each element canonicalized, the resulting strings sorted ascending;
//     serialized as [a,b,...]. Input array order is therefore irrelevant
//     (robust against MCP response ordering).
//   - strings/booleans/null: JSON.stringify (standard JSON escaping).
//   - numbers: callers MUST pre-format size numbers as fixed-precision strings
//     (see the command doc). Raw numbers are serialized via JSON.stringify and
//     are only used for values that come byte-stable from the MCP (token values).
import { createHash } from "node:crypto";

function cmpCodePoint(a, b) {
  for (let i = 0; ; i++) {
    const ca = a.codePointAt(i);
    const cb = b.codePointAt(i);
    if (ca === undefined && cb === undefined) return 0;
    if (ca === undefined) return -1;
    if (cb === undefined) return 1;
    if (ca !== cb) return ca - cb;
  }
}

function canon(v) {
  if (Array.isArray(v)) {
    const parts = v.map(canon);
    parts.sort();
    return "[" + parts.join(",") + "]";
  }
  if (v && typeof v === "object") {
    const keys = Object.keys(v).sort(cmpCodePoint);
    return "{" + keys.map((k) => JSON.stringify(k) + ":" + canon(v[k])).join(",") + "}";
  }
  return JSON.stringify(v);
}

let input = "";
process.stdin.setEncoding("utf8");
process.stdin.on("data", (d) => (input += d));
process.stdin.on("end", () => {
  const obj = JSON.parse(input);
  const hex = createHash("sha256").update(Buffer.from(canon(obj), "utf8")).digest("hex");
  process.stdout.write("sha256:" + hex);
});
```

Invoke it once per piece: build the piece's hash-input object (below), pipe it to
`node product/design/bin/ledger-hash.mjs`, and record the printed `sha256:...`
as the piece's `currentHash`.

#### 4.2 Building the hash input per piece

**Visual pieces** (`primitive` / `component` / `layout` / `view`) — from
`get_metadata` only. For each mapped Figma node build a node descriptor with
EXACTLY these fields and nothing else:

```json
{
  "id":   "<node id in colon form, e.g. 86:2628>",
  "name": "<node name>",
  "type": "<node type, e.g. COMPONENT_SET, COMPONENT, FRAME>",
  "w":    "<width  as a string, fixed 2 decimals, e.g. 120.00>",
  "h":    "<height as a string, fixed 2 decimals, e.g. 40.00>",
  "variants": { "<axisName>": ["<value>", ...], ... }
}
```

- `id`: normalize to colon form (`86:2628`, never `86-2628`).
- `w` / `h`: format the metadata number to **fixed 2 decimals as a string**
  (e.g. `375` → `"375.00"`) — this removes float-representation ambiguity. Never
  pass raw numbers for sizes.
- `variants`: only for component sets — a map of variant axis name → its values.
  Include **both** the axis names and their values. Use `{}` when there are no
  variants. The script sorts axis names and the value arrays, so input order does
  not matter.
- EXCLUDE everything else: x/y position, fills/strokes, children geometry,
  screenshots, generated code, asset URLs. None of it is in `get_metadata`, which
  is exactly why `get_metadata` is the source.

The piece's hash input is `{ "nodes": [ <descriptor>, ... ] }`. A piece that
maps to several nodes (e.g. one logical screen across frames) lists all their
descriptors; the script sorts the array, so a fixed delimiter or manual ordering
is unnecessary — drop them all in.

**Token pieces** (`token` level) — from the DTCG `tokens.json` you wrote in
step 3. The hash input is `{ "tokens": <the token group's DTCG subtree> }` (the
exact object for that group, `$value`/`$type`/`$description` included). The
script's canonicalization (recursively sorted keys, no whitespace) makes this
byte-exact; token values come byte-stable from `get_variable_defs`.

#### 4.3 Worked example (reproducible)

Button component, node `86:2628`. Suppose `get_metadata` yields name `Button`,
type `COMPONENT_SET`, width `120`, height `40`, variant axes `Size` =
{sm, md, lg} and `State` = {default, hover, disabled}. The hash input is:

```json
{"nodes":[{"id":"86:2628","name":"Button","type":"COMPONENT_SET","w":"120.00","h":"40.00","variants":{"Size":["lg","md","sm"],"State":["default","disabled","hover"]}}]}
```

Canonical form produced by the script:

```
{"nodes":[{"h":"40.00","id":"86:2628","name":"Button","type":"COMPONENT_SET","variants":{"Size":["lg","md","sm"],"State":["default","disabled","hover"]},"w":"120.00"}]}
```

SHA-256 → `sha256:3ac8e91181ab4fdba92c0e10932ca009b001aa69a65e0376bf939b0057917052`.

Feeding the same node with its keys and variant arrays in any other order yields
the identical hash. (Substitute the real `get_metadata` values for the live node;
the *method* is what reproduces.)

### 4.4 Refresh the ledger (drift detection, gated by hash spec)

First read `status.json`'s top-level `hashSpecVersion`.

**If it is absent, or differs from the current spec (1)** — the ledger was
hashed with an unknown/older algorithm whose values are not comparable. Do a
controlled **re-baseline**, NEVER false drift:

1. Recompute `currentHash` for every piece with the current script.
2. For every piece that is `implemented` (or a legacy `adoptionSeed: true` piece,
   or any piece with a non-null `lastImplementedHash`): set
   `lastImplementedHash = currentHash`. Carrying the baseline forward — an
   algorithm change is not a design change.
3. **Acceptance carry-forward:** for each such piece, if the stored
   `acceptedHash == lastImplementedHash` *under the old values* (i.e. it was
   accepted before the migration), set `acceptedHash = currentHash` too, so the
   piece stays `accepted`. Testers are not forced to re-accept unchanged UI just
   because of an internal algorithm change. Pieces that were not accepted keep
   their `acceptedHash` (now ≠ the new `lastImplementedHash` → `awaiting-acceptance`,
   correctly). This equality check uses the two stored OLD values, so it works
   even though the old algorithm itself is unrecoverable.
4. Set top-level `hashSpecVersion = 1` and `hashSpec = "node-metadata-structural-v1"`.
5. Commit (`Migrate design ledger to hashSpec v1`). **Write NO UI plan this run**
   — a migration is not drift. Report: *"Ledger migrated to hashSpec v1 — N
   pieces re-baselined, M acceptances carried forward. Re-run
   `/dev:figma-refresh-plan` to detect real drift and plan any genuinely open
   pieces."* Then stop.

This is the only safe behavior when the old hashes are not comparable: assume the
current design equals the last-implemented one and re-baseline. A genuine design
change that happens to coincide with the migration is caught on the next run
(normal drift, below) — never silently turned into a bogus rework plan.

**If `hashSpecVersion` matches the current spec (1)** — normal drift detection:

- **New piece** (not in the ledger): add it with `status: "to-implement"` and
  `lastImplementedHash: null` — unless it is on the plan's adoption `matches`
  list (see "Adoption seeding"), in which case seed it `implemented` with
  `lastImplementedHash = currentHash`.
- **Direct change** (`currentHash != lastImplementedHash`): set
  `status: "to-implement"`.
- **Unchanged** (`currentHash == lastImplementedHash`): leave `implemented`.
- **Soft cascade**: after direct changes are marked, for every piece that
  (transitively) `dependsOn` a piece now `to-implement`, set its status to
  `to-review` — unless it is itself directly changed (`to-implement` wins).

Dependency edges (`dependsOn`) come from instance → main-component references and
from token references. Build them from the pulled metadata/context.

#### `status.json` schema

```json
{
  "hashSpecVersion": 1,
  "hashSpec": "node-metadata-structural-v1",
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

- `hashSpecVersion` / `hashSpec`: which hash algorithm produced the stored
  hashes. The command compares this to its current spec; a mismatch (or absence)
  triggers the controlled re-baseline in step 4.4, never false drift. Bump these
  together whenever the canonicalization changes.
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
lifecycle. Mark each seeded piece `adoptionSeed: true`.

**Back-compat / legacy ledgers.** A ledger written before hash specs existed has
no `hashSpecVersion` (and may carry `adoptionSeed: true` pieces or hashes from
the old, unrecoverable prose method). Do **not** compare those hashes — the
absent `hashSpecVersion` routes the next run through the step 4.4 re-baseline,
which recomputes with the current script and carries baselines and acceptances
forward. Such pieces are never treated as drift.

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
| `status.json` has no `hashSpecVersion`, or it differs from the current spec (legacy / `adoptionSeed` / null hashes) | Controlled re-baseline (step 4.4): recompute with the current script, carry baselines + accepted pieces forward, set `hashSpecVersion`, commit "migrated", write NO plan, point to a re-run. NEVER false drift. |
| `node` unavailable for the reference hash script | Stop with a clear message. Do NOT fall back to ad-hoc/prose hashing. |
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
4. **Deterministic hashing only.** Hashes come exclusively from the committed
   `ledger-hash.mjs` reference script over the pinned `get_metadata`-derived
   descriptor (visual pieces) or DTCG subtree (token pieces). Never hash
   `get_design_context` output, never improvise a canonicalization, never compute
   a hash by hand. Identical design ⇒ identical hash, across sessions and MCP
   ordering.
5. **Never false drift on a spec change.** If `hashSpecVersion` is absent or
   differs, re-baseline (step 4.4) — recompute, carry baselines + acceptances
   forward, report "migrated," write no plan. Bump `hashSpecVersion` whenever the
   canonicalization changes.
6. **Figma is the source of truth** for sourced designs; deviations are reported,
   not applied.
7. **Per-breakpoint granularity** for views, layouts, AND components (all are
   responsive); only the changed breakpoint flips.
8. **Soft cascade**: direct change → `to-implement`; dependents → `to-review`.
9. **No no-op commits.** Skip the commit when nothing changed.
10. **Tokens in DTCG format.** Generate `tokens.json` as W3C DTCG even though the
    project does not use Tokens Studio, for interop.
11. **No screenshots stored.** Visual checking happens live in `/dev:implement`.
