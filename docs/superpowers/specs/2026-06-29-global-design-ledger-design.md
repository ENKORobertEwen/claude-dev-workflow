# Global Design Ledger — Design

## Problem

The design ledger is currently **plan-specific**: one ledger per plan slug at
`product/design/<plan-slug>/status.json` (see
`2026-06-24-figma-refresh-skill-design.md` line 125,
`dev-workflow/commands/figma-refresh-plan.md:109`).

That partitioning is wrong for what the ledger actually is. The ledger is the
**persistent, global status/roadmap of the implementation** — for every design
piece, the truth of "implemented / open / accepted" plus the drift baseline
(`lastImplementedHash` vs `currentHash`). None of that is plan-scoped:

- A piece's hashes are unique to the piece, not to a plan. A `button` primitive
  has one Figma hash regardless of which plan touches it.
- The atomic-design lower levels (tokens, primitives, components) are a **shared
  design system** consumed by every feature. Splitting the ledger per plan means
  a shared `button` implemented in plan A is invisible to plan B's ledger — it
  gets re-tracked from scratch, and drift in a shared component does not
  propagate to the plans that depend on it.
- The UI plan is already only a **transient projection** of the ledger's open
  pieces. The persistent thing — the ledger — is the roadmap, and it was
  accidentally partitioned along the disposable artifact (the plan).

The intended concept: the ledger is a better, global TODO/status list — a
roadmap. Plans are disposable, selective slices of it.

## Goal

Make the ledger **global**: one ledger for the whole project, keyed by design
piece, spanning multiple Figma source files. Plans become selective,
conversationally-chosen projections of the open entries. Snapshots of the raw
Figma design stay per-source.

## Decisions

These were settled during brainstorming and are fixed for this design:

1. **One global ledger**, snapshots per Figma source, multiple Figma files
   expected per project.
2. **Source-namespaced piece identity** — keys carry their source as a namespace
   (`lib:button.primitive`, `login:view.mobile`) plus an explicit `source`
   field. `dependsOn` references the namespaced keys and may cross source
   boundaries (feature → library).
3. **The UI plan file stays** — a transient, *selective* projection. The user
   picks which open entries go into a given plan; it is no longer "all open at
   once."
4. **Selection is conversational**, driven by `/plan`. No selection DSL or rigid
   mechanism — the model and user converse over the open ledger entries.
5. **`/plan` compares code against the materialized snapshot (option A)**, not a
   fresh live Figma pull. The snapshot is the pinned design truth for the cycle;
   the ledger baseline and the plan therefore rest on the same design stand.

## Storage layout

```
product/design/
  status.json                  ← ONE global ledger (all pieces, all sources)
  bin/ledger-hash.mjs          ← unchanged; normative deterministic hash script
  sources/
    <figma-file-a>/ { source.md, context.md, tokens.json }
    <figma-file-b>/ { source.md, context.md, tokens.json }
```

- `<figma-file>` is a stable per-source slug (the `source` value used in piece
  keys), one directory per Figma file the project pulls from.
- Per-source metadata — `figmaFileKey`, `figmaUrl`, `fileVersion`, `fetchedAt` —
  moves out of the ledger top level into the source's `source.md`.
- The global `status.json` keeps only project-global fields at the top level:
  `hashSpecVersion`, `hashSpec`.
- The hash script stays a single committed file at `product/design/bin/`. The
  hash algorithm and its determinism guarantees are unchanged.

## Ledger schema

Top level is now global; per-source metadata is gone from it. Each piece carries
its `source` and uses namespaced ids:

```json
{
  "hashSpecVersion": 1,
  "hashSpec": "node-metadata-structural-v1",
  "pieces": [
    {
      "id": "login:view.mobile",
      "source": "login",
      "level": "view",
      "breakpoint": "mobile",
      "figmaNodeIds": ["123:45"],
      "codeTarget": "src/pages/Login.tsx",
      "lastImplementedHash": "sha256:…",
      "currentHash": "sha256:…",
      "status": "to-implement",
      "dependsOn": ["lib:button.primitive", "lib:color.primary.token"],
      "acceptedHash": null,
      "acceptedBy": null,
      "acceptedAt": null
    }
  ]
}
```

- `id` is `<source>:<piece-path>`, where the namespace prefix is **identical** to
  the `source` field and to the `sources/<source>/` snapshot directory name.
  `source` duplicates the namespace as an explicit field so a snapshot directory
  is locatable without parsing the id.
- `lib:*` design-system pieces (tokens / primitives / components) exist **once**
  and are shared. Feature sources reference them through `dependsOn`.
- All other fields (`level`, `breakpoint`, hashes, `status`, acceptance) keep
  their current meaning and the existing status/cascade semantics
  (`to-implement` / `to-review` / `implemented`, soft dependency cascade).

## Command roles

The substantive behavioral change. `figma-refresh-plan` stops writing UI plans;
that responsibility — now selective and conversational — moves to `/plan`.

| Command | Role |
|---|---|
| **`/figma-refresh-plan <source>`** | The only Figma **pull**. Pulls one source, materializes its snapshot (`source.md`, `context.md`, `tokens.json`), and updates the global ledger: add missing pieces, flip drifted pieces to `to-implement`/`to-review`, recompute `currentHash`. **Writes no UI plan.** Prints an "open per piece" overview to stdout — ephemeral, not a committed file. |
| **`/plan`** | Reads open pieces directly from the global ledger. The user conversationally selects a subset of open entries (e.g. by referencing `login:view.*`). For the selected pieces, `/plan` compares the current code against the **snapshot** (`sources/<source>/context.md`, `tokens.json`) and writes the UI plan to `product/plans/todo/`. No live pull, no hashing. First time per source it additionally resolves the Figma mapping + design-system spec. |
| **`/implement`** | Unchanged role; ledger write-back now targets the global `product/design/status.json`. |
| **`/figma-accept`** | Reads/writes the global ledger; locates pieces by namespaced id. |

### "What's open" is a projection, not a file

There is deliberately **no persistent "open per piece" file**. "What's open" is
`pieces.filter(status ∈ {to-implement, to-review})` — a trivial projection any
command reads directly off the ledger. `figma-refresh-plan` prints it to stdout
as a run summary; `/plan` filters it in memory. Materializing it as a second
committed artifact would reintroduce exactly the drift/duplication the global
ledger removes.

### Two Figma "stock-takings", kept separate

1. **Design-drift stock-taking** — pull Figma, compute hashes, compare to
   `lastImplementedHash`, flip status. Needs the pull → lives in
   `figma-refresh-plan`, across all pieces of a source.
2. **Code-vs-design stock-taking** — for the selected pieces, compare current
   code against the pinned snapshot to plan precisely. Needs no pull → lives in
   `/plan`, over the selected subset.

`/plan` never re-pulls Figma. If the snapshot is stale, run
`figma-refresh-plan <source>` first.

## Lifecycle

- **New source:** `/plan` (resolve mapping + design-system spec) →
  `/figma-refresh-plan <source>` (seed ledger + snapshot) → `/plan` (select open
  pieces, compare code ↔ snapshot, write UI plan) → `/implement`.
  - The two-stage `/plan` is accepted: mapping resolution must precede the pull;
    UI-plan selection must follow the seeded ledger.
- **Design change on an existing source:** `/figma-refresh-plan <source>`
  (drift into ledger) → `/plan` (select + plan) → `/implement`.

## Migration

One-time, documented:

1. Merge every existing `product/design/<plan-slug>/status.json` into the single
   `product/design/status.json`.
2. Raise piece ids to the `<source>:` namespace and populate the `source` field;
   rewrite `dependsOn` entries to namespaced keys.
3. Move snapshot files from `product/design/<plan-slug>/` to
   `product/design/sources/<source>/`.
4. Lift `figmaFileKey` / `figmaUrl` / `fileVersion` / `fetchedAt` from the old
   ledger top level into each source's `source.md`.
5. The hash spec (`hashSpecVersion`) is unchanged, so baselines and acceptances
   carry forward without re-baselining — this is a relocation, not a drift event.

## Out of scope

- No change to the hash algorithm, `ledger-hash.mjs`, or determinism guarantees.
- No change to status flags, the soft dependency cascade, or acceptance
  semantics.
- No selection DSL — selection stays conversational.
- Live re-pull inside `/plan` (option B) is explicitly rejected.
