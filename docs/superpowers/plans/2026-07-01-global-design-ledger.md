# Global Design Ledger Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convert the design ledger from one-per-plan (`product/design/<plan-slug>/status.json`) to a single global ledger (`product/design/status.json`) keyed by source-namespaced pieces, and move UI-plan writing from `/dev:figma-refresh-plan` to `/dev:plan`.

**Architecture:** This is a documentation/prompt-engineering change to the `dev` plugin's Markdown command files — there is no application code and no test suite. Each task edits one command doc (or README/CHANGELOG) to a self-consistent state. "Verification" is a `grep` invariant check per task: the old artifact strings must be gone and the new ones present. Follow the spec `docs/superpowers/specs/2026-06-29-global-design-ledger-design.md` verbatim.

**Tech Stack:** Markdown command docs under `dev-workflow/commands/`, `dev-workflow/README.md`, `dev-workflow/CHANGELOG.md`, `dev-workflow/.claude-plugin/plugin.json`. Verification via `grep`. Commits via `git`.

## Global Constraints

These strings are the shared contract. Every task MUST use these exact forms — do not paraphrase paths or the id format, or the docs will contradict each other:

- **Global ledger path (the one source of truth):** `product/design/status.json`
- **Per-source snapshot dir:** `product/design/sources/<source>/` containing `source.md`, `context.md`, `tokens.json`
- **Hash script (unchanged location):** `product/design/bin/ledger-hash.mjs`
- **Piece id format:** `<source>:<piece-path>` where the namespace prefix is **identical** to the piece's `source` field and to its `sources/<source>/` directory name. Examples: `lib:button.primitive`, `login:view.mobile`.
- **`dependsOn`** entries are namespaced ids and may cross sources (feature → library), e.g. `["lib:button.primitive"]`.
- **Shared design-system pieces** (`token` / `primitive` / `component` levels) live under a single library source (conventionally `lib`) and exist **once**.
- **Command role split (the behavioral change):**
  - `/dev:figma-refresh-plan <source>` = the ONLY Figma pull. Pulls one source, materializes its snapshot, updates the global ledger (add missing, drift → `to-implement`/`to-review`, recompute `currentHash`). **Writes NO UI plan.** Prints an "open per piece" overview to stdout (ephemeral, never a committed file).
  - `/dev:plan` = reads open pieces directly from the global ledger; user conversationally selects a subset; for the selected pieces compares current code against the **snapshot** (`sources/<source>/context.md`, `tokens.json`); writes the UI plan to `product/plans/todo/`. No live pull, no hashing. First time per source it also resolves the mapping + design-system spec.
  - `/dev:implement` = unchanged role; ledger write-back targets the global `product/design/status.json`.
  - `/dev:figma-accept` = reads/writes the global ledger; locates pieces by namespaced id.
- **Out of scope (do not touch):** the hash algorithm, `ledger-hash.mjs` contents, determinism guarantees, status flags, the soft dependency cascade, acceptance semantics. No selection DSL. No live re-pull in `/dev:plan`.
- **Version bump:** `dev-workflow/.claude-plugin/plugin.json` `version` → `2.13.0`.

---

### Task 1: `figma-refresh-plan.md` — global ledger, source snapshots, drop UI-plan writing

**Files:**
- Modify: `dev-workflow/commands/figma-refresh-plan.md`

**Interfaces:**
- Produces (consumed by every later task): the global ledger at `product/design/status.json`, snapshots at `product/design/sources/<source>/`, the source-namespaced `status.json` schema, and the rule that this command writes NO UI plan.
- Consumes: nothing (first task).

- [ ] **Step 1: Rewrite the header framing (lines 1–18).**

Replace the current top block:

```markdown
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
```

with:

```markdown
# Figma Refresh Plan Command

Pull the current Figma design for **one source** into the repo and update the
**global** per-piece implementation-status ledger (`product/design/status.json`).
This is the pixels → ledger step. It does **not** write a UI plan — selecting
open pieces and writing the UI plan is `/dev:plan`'s job (it reads this ledger).

This command runs **after `/dev:plan`** has resolved the Figma mapping and
design-system spec for the source it pulls:

- **Pulling the design and maintaining the global ledger always happens.**
- **No UI plan is ever written here.** The command reports an "open per piece"
  overview to stdout so you can see what is open, then points at `/dev:plan`.

There is **no automatic refresh inside `/implement`** — refreshing is always
this explicit step, and `/dev:implement` only executes plans.
```

- [ ] **Step 2: Rewrite the workflow diagram (lines 20–41).**

Replace the ```` ``` ````-fenced block under `## Where this sits in the workflow` with:

```markdown
```
1. /dev:plan                 Resolves the Figma mapping + design-system spec
                             (naming convention, responsive rules) for a source.
                             Later runs: selects open pieces from the global
                             ledger and writes the UI plan. Does NOT pull.

2. /dev:figma-refresh-plan   Pull one source (using the mapping), create/update
     <source>                the GLOBAL ledger. Writes NO UI plan; prints the
                             open-piece overview to stdout.

3. /dev:plan                 Select the open pieces to work on now; compare code
                             against the snapshot; write the UI plan.

4. /dev:implement            Executes the plan; writes the global ledger back to
                             "implemented".

5. /dev:figma-refresh-plan   Later, design changed: pull fresh, diff via the
     <source>                global ledger (drift → to-implement/to-review).
```
```

- [ ] **Step 3: Add the `<source>` argument to the Input section (lines 45–59).**

After the priority list (ends at the line beginning `3. **No Figma reference anywhere**`), and before `The plan's UI/UX Spec is the source of the **mapping**`, insert:

```markdown
**Source selection.** A project may pull from several Figma files. The
`<source>` argument names which one — it is the stable slug used as the piece-id
namespace and the snapshot directory (`product/design/sources/<source>/`).
Conventionally the shared design-system library is the source `lib`; each
feature file is its own source (e.g. `login`). If `<source>` is omitted and the
current plan references exactly one Figma file, use that file's source slug;
if it references several, list them and ask which to refresh.
```

- [ ] **Step 4: Repoint the cheap-change-gate path (line 77).**

Replace:

```markdown
Read the last pulled file version from `product/design/<plan-slug>/source.md`
```

with:

```markdown
Read the last pulled file version from `product/design/sources/<source>/source.md`
```

Also, in the same block (line 82), replace `Make **no commit** and write **no plan**. Stop.` with `Make **no commit**. Stop.`

- [ ] **Step 5: Repoint the materialize-files heading (line 109) and its table.**

Replace:

```markdown
Write into `product/design/<plan-slug>/`:
```

with:

```markdown
Write into `product/design/sources/<source>/`:
```

Then in the file table below it, replace the `source.md` row description so per-source metadata is explicit. Change the `source.md` row to:

```markdown
| `source.md` | Figma file key + URL, mapped node IDs, file `version`, fetched-at timestamp — the per-source metadata (no longer stored in the ledger) |
```

- [ ] **Step 6: Rewrite the `status.json` schema block (lines 329–353) to global + namespaced form.**

Replace the JSON block with:

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
      "lastImplementedHash": "sha256:...",
      "currentHash": "sha256:...",
      "status": "to-implement",
      "dependsOn": ["lib:button.primitive", "lib:color.primary.token"],
      "acceptedHash": null,
      "acceptedBy": null,
      "acceptedAt": null
    }
  ]
}
```

- [ ] **Step 7: Update the schema field notes (lines 356–376).**

Immediately after the JSON block, replace the first bullet (`- `hashSpecVersion` / `hashSpec`: ...`) by inserting two new bullets ahead of it:

```markdown
- **Top level is global.** `product/design/status.json` holds every piece from
  every source. Per-source metadata (`figmaFileKey`, `figmaUrl`, `fileVersion`,
  `fetchedAt`) lives in each `sources/<source>/source.md`, NOT here.
- `id` is `<source>:<piece-path>`; the namespace prefix equals the `source`
  field and the `sources/<source>/` directory name. `dependsOn` uses the same
  namespaced ids and may cross sources (feature → library). Shared
  design-system pieces (`lib:*`) exist once and are referenced by every feature.
```

- [ ] **Step 8: Delete UI-plan writing — steps 5 and 6 (lines 400–444).**

Remove the entire `### 5. Write the UI plan from the ledger` section and the entire `### 6. Write the UI plan` section. Replace both with a single new section:

```markdown
### 5. Report the open pieces (no UI plan)

This command writes **no** UI plan. After the ledger is updated, print an
"open per piece" overview to stdout: for each piece with `status` in
`to-implement` / `to-review`, list its `id`, `level`, `breakpoint`, and status.
Group by source. This is ephemeral console output — never a committed file;
"what's open" is always re-derived from the ledger.

Then point the user at `/dev:plan` to select which open pieces to plan and
implement next.
```

- [ ] **Step 9: Repoint the commit block (lines 446–458).**

In the `### 7. Commit` (now following the renumbered sections), replace the `git add` line:

```markdown
git add product/design/<plan-slug>/ product/plans/todo/   # plan dir only if a UI plan was written
```

with:

```markdown
git add product/design/status.json product/design/sources/<source>/
```

And change the commit message example and surrounding prose so it refers only to the ledger + snapshot (no plan dir).

- [ ] **Step 10: Fix the Report section (lines 460–470).**

In `### 8. Report`, replace the bullet:

```markdown
- Whether a UI plan was written (and its path), or "nothing to do" when no piece
  is open.
```

with:

```markdown
- The open-piece overview (counts of `to-implement` / `to-review` /
  `awaiting-acceptance` / `accepted`), printed to stdout — no plan is written.
```

And change the "next step" bullet to point at `/dev:plan` (select + write the UI plan), not `/dev:implement`.

- [ ] **Step 11: Repoint the write-back doc and ad-hoc path (lines 472–488).**

In `## Ledger write-back`, replace `updates that piece in` / `` `status.json` `` prose so it names `product/design/status.json`. In `## Ad-hoc mode`, replace `product/design/_adhoc/<figma-node>/` — keep ad-hoc snapshots at `product/design/sources/_adhoc/<figma-node>/` for consistency with the new `sources/` layout.

- [ ] **Step 12: Rewrite the Operating-Decisions table rows (lines 507–515).**

- Line 507: `File version unchanged since last pull | Report "no changes". No commit, no plan. Stop.` → `... No commit. Stop.`
- Line 511: keep the re-baseline row but change `write NO plan` → `write nothing but the migrated ledger`.
- Line 513 (`Open pieces exist ... write the UI plan ...`): replace the whole row with `Open pieces exist | Pull + update the global ledger + commit. Print the open overview. Point to /dev:plan.`
- Line 514 (`Nothing open ...`): replace `No plan; no commit unless the snapshot itself changed.` → `No commit unless the snapshot itself changed. Point to /dev:plan only if something is open.`
- Line 515 (`Ad-hoc free URL ...`): update the path to `product/design/sources/_adhoc/`.

- [ ] **Step 13: Rewrite Critical Rule 2 (lines 522–524).**

Replace:

```markdown
2. **Single source of UI plans.** This command writes the UI plan from the ledger
   for every open piece — first build and later rework alike. `/dev:plan` does
   not write per-piece UI phases. The only no-plan case is nothing open.
```

with:

```markdown
2. **Never writes a UI plan.** This command only pulls one source and updates
   the global ledger, then reports the open pieces. `/dev:plan` is the single
   source of UI plans — it selects open ledger pieces and writes them.
```

- [ ] **Step 14: Verify Task 1 invariants.**

Run:

```bash
grep -n "plan-slug\|write the UI plan\|single source of UI plans" dev-workflow/commands/figma-refresh-plan.md
```

Expected: **no output** (every `<plan-slug>` path is gone; no UI-plan-writing claims remain).

Run:

```bash
grep -n "product/design/status.json\|product/design/sources/<source>/\|<source>:" dev-workflow/commands/figma-refresh-plan.md
```

Expected: multiple matches (global path, snapshot path, namespaced ids all present).

- [ ] **Step 15: Commit.**

```bash
git add dev-workflow/commands/figma-refresh-plan.md
git commit -m "figma-refresh-plan: global ledger + source snapshots, drop UI-plan writing"
```

---

### Task 2: `figma-refresh-plan.md` — per-plan → global migration behavior

**Files:**
- Modify: `dev-workflow/commands/figma-refresh-plan.md`

**Interfaces:**
- Consumes: the global paths/schema from Task 1.
- Produces: the documented one-time migration that later reviewers and `/dev:implement` rely on (old per-plan ledgers become the global one).

- [ ] **Step 1: Add a migration subsection after the schema notes.**

After the `#### Adoption seeding` subsection (around line 378+), add a new subsection:

```markdown
#### Migrating legacy per-plan ledgers (one-time, automatic, safe)

Before hash-spec v-checks, on any run: if one or more legacy
`product/design/<plan-slug>/status.json` files exist and `product/design/status.json`
does not yet contain their pieces, migrate them — this is a **relocation, not a
drift event**:

1. For each legacy ledger, derive its source slug (the `<plan-slug>` dir name, or
   a slug confirmed with the user) and namespace every piece id to `<source>:…`.
2. Rewrite each piece's `dependsOn` entries to the namespaced ids. Populate the
   new `source` field.
3. Move `source.md` / `context.md` / `tokens.json` from
   `product/design/<plan-slug>/` to `product/design/sources/<source>/`.
4. Lift `figmaFileKey` / `figmaUrl` / `fileVersion` / `fetchedAt` from each legacy
   ledger's top level into that source's `source.md`.
5. Merge all pieces into a single `product/design/status.json` with only
   `hashSpecVersion` / `hashSpec` at the top level.
6. Because `hashSpecVersion` is unchanged, carry `lastImplementedHash` and all
   acceptance fields forward untouched — do **not** recompute or flip status.
7. Commit `Migrate design ledger to global (product/design/status.json)`. Write
   nothing else this run; report the migration and point to `/dev:plan`.
```

- [ ] **Step 2: Add the migration to the Operating-Decisions table.**

Add a row to the `## Hardcoded Operational Decisions` table:

```markdown
| Legacy per-plan ledger(s) exist (`product/design/<plan-slug>/status.json`) | One-time migration to `product/design/status.json` (relocation, not drift): namespace ids, move snapshots to `sources/<source>/`, carry baselines + acceptances forward, commit "migrate", write nothing else. |
```

- [ ] **Step 3: Verify Task 2 invariants.**

Run:

```bash
grep -n "Migrating legacy per-plan ledgers\|relocation, not a" dev-workflow/commands/figma-refresh-plan.md
```

Expected: both match (migration subsection present).

- [ ] **Step 4: Commit.**

```bash
git add dev-workflow/commands/figma-refresh-plan.md
git commit -m "figma-refresh-plan: document one-time per-plan to global ledger migration"
```

---

### Task 3: `plan.md` — `/dev:plan` writes the UI plan from the global ledger

**Files:**
- Modify: `dev-workflow/commands/plan.md`

**Interfaces:**
- Consumes: the global ledger path, snapshot path, and namespaced ids from Task 1.
- Produces: the recurring `/dev:plan` UI-plan-writing behavior that `/dev:implement` (Task 4) executes.

- [ ] **Step 1: Rewrite the "does not write UI phases" paragraph (line 139).**

Replace:

```markdown
**`/plan` does not write the per-piece UI phases.** For Figma-sourced frontends, the UI plan (design system → primitives → components → layouts → views×breakpoint) is produced by `/dev:figma-refresh-plan` from the ledger — it is the single source of UI plans. `/plan`'s job for the frontend is to resolve the **mapping + design-system spec** (above) and to plan the **non-UI technical groundwork** (env, framework, routing, component architecture, build/`./do` changes). After `/plan`, the order is: `/dev:figma-refresh-plan` (pull + ledger + UI plan) → `/dev:implement`.
```

with:

```markdown
**`/plan` writes the per-piece UI plan — from the global ledger.** For Figma-sourced frontends the flow is two-stage per source. First `/plan` resolves the **mapping + design-system spec** (above) so `/dev:figma-refresh-plan <source>` can pull and seed the global ledger. Then `/plan` runs again as the **UI-planning** pass: it reads the open pieces (`to-implement` / `to-review`) directly from `product/design/status.json`, and **you conversationally select** which of them to plan now (e.g. "just `login:view.*` plus its open `lib:` dependencies"). For each selected piece `/plan` compares the current code against the pinned **snapshot** (`product/design/sources/<source>/context.md`, `tokens.json`) — it does NOT pull Figma or compute hashes — and writes the UI plan (design system → primitives → components → layouts → views×breakpoint, dependency order) to `product/plans/todo/`. `/plan` also plans the **non-UI technical groundwork** (env, framework, routing, component architecture, build/`./do` changes). Order: `/dev:plan` (mapping) → `/dev:figma-refresh-plan <source>` (pull + global ledger) → `/dev:plan` (select + UI plan) → `/dev:implement`.
```

- [ ] **Step 2: Fix the dependency-order reference (line 141).**

Replace `the dependency order `/dev:figma-refresh-plan` uses is bottom-up` with `the dependency order the UI plan uses is bottom-up`.

- [ ] **Step 3: Fix the five-levels-sweep parenthetical (line 143).**

Replace `(`/dev:figma-refresh-plan` enforces the same sweep automatically via the ledger, but only for pieces present in the mapping — so keep the mapping current here when new pieces appear.)` with `(`/dev:figma-refresh-plan` keeps the global ledger's pieces current via the mapping — so keep the mapping current here when new pieces appear; `/plan` then selects from those pieces.)`

- [ ] **Step 4: Fix the mapping-consumer note (line 116).**

Replace `Pulling the design, the per-piece status ledger, and the (re)work plan are produced by `/dev:figma-refresh-plan`, which consumes this mapping.` with `Pulling the design and maintaining the global per-piece status ledger are done by `/dev:figma-refresh-plan`, which consumes this mapping; `/plan` then selects open pieces from that ledger and writes the UI plan.`

- [ ] **Step 5: Fix the adoption-seeding hand-off (lines 164–173).**

Replace the sentence at lines 166–169 (`here — `/dev:figma-refresh-plan` writes those from the seeded ledger ...`) and lines 171–173 (`Then run `/dev:figma-refresh-plan`: it seeds the ledger from the `matches` list, pulls the design, and writes the UI remediation plan.`) so the write-the-plan step is attributed to a follow-up `/dev:plan` pass, not to `figma-refresh-plan`:

```markdown
   here — after the ledger is seeded, a `/dev:plan` UI-planning pass writes those
   from the global ledger (every `refactor` / `rebuild` / `missing` piece is
   `to-implement` and selectable; `matches` pieces are seeded `implemented` and
   get no phase).
```

and

```markdown
Then run `/dev:figma-refresh-plan <source>`: it seeds the global ledger from the
`matches` list and pulls the design. Run `/dev:plan` again to select the open
pieces and write the UI remediation plan. After this one-time pass the project
is on the normal lifecycle.
```

- [ ] **Step 6: Verify Task 3 invariants.**

Run:

```bash
grep -n "single source of UI plans\|does not write the per-piece UI phases" dev-workflow/commands/plan.md
```

Expected: **no output**.

Run:

```bash
grep -n "product/design/status.json\|conversationally select\|sources/<source>/context.md" dev-workflow/commands/plan.md
```

Expected: matches present.

- [ ] **Step 7: Commit.**

```bash
git add dev-workflow/commands/plan.md
git commit -m "plan: /dev:plan selects open pieces from the global ledger and writes the UI plan"
```

---

### Task 4: `implement.md` — global ledger write-back

**Files:**
- Modify: `dev-workflow/commands/implement.md`

**Interfaces:**
- Consumes: the global ledger path and namespaced ids (Task 1); the UI plan produced by `/dev:plan` (Task 3).
- Produces: ledger write-back to `product/design/status.json` that `/dev:figma-refresh-plan` and `/dev:figma-accept` read.

- [ ] **Step 1: Repoint the write-back path (line 201).**

Replace:

```markdown
If the phase implemented Figma-sourced pieces tracked in
`product/design/<plan-slug>/status.json`, update each implemented piece before
```

with:

```markdown
If the phase implemented Figma-sourced pieces tracked in the global ledger
`product/design/status.json`, update each implemented piece (by its
`<source>:…` id) before
```

- [ ] **Step 2: Repoint the staged file (line 205).**

Replace `Stage `status.json` with the phase commit.` with `Stage `product/design/status.json` with the phase commit.`

- [ ] **Step 3: Fix the codeTarget-reuse note (line 57).**

Replace `(from Code Connect or the plan's mapping in `status.json`)` with `(from Code Connect or the plan's mapping in `product/design/status.json`)`.

- [ ] **Step 4: Fix the "no ledger" stop condition (line 402).**

Replace the table cell text `Frontend phase is Figma-sourced but no design files / ledger exist (`product/design/<plan-slug>/status.json` missing — `figma-refresh-plan` was never run)` with `Frontend phase is Figma-sourced but the global ledger `product/design/status.json` has no matching pieces (`figma-refresh-plan` was never run for that source)`. Keep the existing "full stop" behavior text.

- [ ] **Step 5: Verify Task 4 invariants.**

Run:

```bash
grep -n "plan-slug" dev-workflow/commands/implement.md
```

Expected: **no output**.

Run:

```bash
grep -n "product/design/status.json" dev-workflow/commands/implement.md
```

Expected: matches present (write-back, codeTarget, stop condition).

- [ ] **Step 6: Commit.**

```bash
git add dev-workflow/commands/implement.md
git commit -m "implement: write ledger back to the global product/design/status.json"
```

---

### Task 5: `figma-accept.md` — global ledger, namespaced targeting

**Files:**
- Modify: `dev-workflow/commands/figma-accept.md`

**Interfaces:**
- Consumes: the global ledger path and namespaced ids (Task 1); the write-back records (Task 4).
- Produces: acceptance fields written to the global ledger.

- [ ] **Step 1: Repoint the ledger-location prose (line 13).**

Replace `Each piece in `product/design/<plan-slug>/status.json` carries, alongside its` with `Each piece in the global ledger `product/design/status.json` carries, alongside its`.

- [ ] **Step 2: Rewrite the Input "determine the ledger" line (line 45).**

Replace:

```markdown
Determine the ledger from the current plan (its `product/design/<plan-slug>/`).
Then determine which pieces to accept:
```

with:

```markdown
Read the global ledger `product/design/status.json`. Then determine which pieces
to accept (pieces are addressed by their `<source>:…` id):
```

- [ ] **Step 3: Repoint the read/write-back steps (lines 62, 70).**

Replace `1. Read `status.json`. Build the list of target pieces from the input.` with `1. Read `product/design/status.json`. Build the list of target pieces from the input.` and `3. Write `status.json` back.` with `3. Write `product/design/status.json` back.`

- [ ] **Step 4: Fix the "no ledger" row (line 83).**

Replace the cell `No ledger / no Figma-sourced design | Report "no design ledger to accept against." Stop.` with `No global ledger / no Figma-sourced pieces | Report "no design ledger to accept against." Stop.`

- [ ] **Step 5: Verify Task 5 invariants.**

Run:

```bash
grep -n "plan-slug" dev-workflow/commands/figma-accept.md
```

Expected: **no output**.

Run:

```bash
grep -n "product/design/status.json\|<source>:" dev-workflow/commands/figma-accept.md
```

Expected: matches present.

- [ ] **Step 6: Commit.**

```bash
git add dev-workflow/commands/figma-accept.md
git commit -m "figma-accept: read/write the global ledger, address pieces by namespaced id"
```

---

### Task 6: `README.md` — global ledger overview, storage, roles, lifecycle

**Files:**
- Modify: `dev-workflow/README.md`

**Interfaces:**
- Consumes: everything from Tasks 1–5.
- Produces: user-facing documentation consistent with the new behavior.

- [ ] **Step 1: Read the current ledger/lifecycle prose to anchor the edits.**

Run:

```bash
grep -n "status ledger\|status.json\|figma-refresh-plan\|UI plan\|product/design" dev-workflow/README.md
```

Read each matched line's surrounding paragraph before editing.

- [ ] **Step 2: Rewrite the ledger-description bullet (line 73).**

Replace the bullet that begins `Maintains a per-piece **status ledger** (`product/design/<plan-slug>/status.json`)` so it describes a single global ledger at `product/design/status.json` with per-source snapshots under `product/design/sources/<source>/`, source-namespaced piece ids, and shared `lib:*` design-system pieces. Keep the deterministic-hash sentence intact (hashing is out of scope).

- [ ] **Step 3: Rewrite the UI-plan bullet (line 77).**

Replace the bullet `Writes the **UI plan** from the ledger to `product/plans/todo/` ...` so UI-plan writing is attributed to `/dev:plan` (selective, conversational), and `figma-refresh-plan` is described as ledger-update + open-overview only.

- [ ] **Step 4: Rewrite the "Order" paragraph (line 79).**

Replace the `**Order: `/plan` → `/figma-refresh-plan` → `/implement`.**` paragraph with the two-stage-per-source order: `/dev:plan` (mapping) → `/dev:figma-refresh-plan <source>` (pull + global ledger) → `/dev:plan` (select + UI plan) → `/dev:implement`.

- [ ] **Step 5: Fix the adoption paragraph (line 81).**

Update the `Adopting an in-progress project` paragraph so seeding lands in the global ledger and the UI remediation plan is written by the follow-up `/dev:plan` pass, not by `figma-refresh-plan`.

- [ ] **Step 6: Fix the acceptance paragraph (line 85).**

Update any `product/design/<plan-slug>/` reference to `product/design/status.json`.

- [ ] **Step 7: Verify Task 6 invariants.**

Run:

```bash
grep -n "plan-slug" dev-workflow/README.md
```

Expected: **no output**.

Run:

```bash
grep -n "product/design/status.json\|product/design/sources/" dev-workflow/README.md
```

Expected: matches present.

- [ ] **Step 8: Commit.**

```bash
git add dev-workflow/README.md
git commit -m "README: describe the global design ledger and the /plan-writes-UI-plan flow"
```

---

### Task 7: CHANGELOG, version bump, and final cross-file invariant check

**Files:**
- Modify: `dev-workflow/CHANGELOG.md`
- Modify: `dev-workflow/.claude-plugin/plugin.json`

**Interfaces:**
- Consumes: all prior tasks.
- Produces: the released version and the guarantee that no stale artifact string survives anywhere.

- [ ] **Step 1: Add the CHANGELOG entry.**

Prepend a new top section to `dev-workflow/CHANGELOG.md` (above `## 2.12.0`):

```markdown
## 2.13.0 — Global design ledger

**Problem fixed:** the design ledger was one-per-plan
(`product/design/<plan-slug>/status.json`), so shared design-system pieces
(tokens/primitives/components) were re-tracked per plan and drift in a shared
component never propagated across plans. The ledger is really the project's
global implementation roadmap, accidentally partitioned along the disposable
plan artifact.

**Changes:**
- One **global** ledger at `product/design/status.json`, keyed by
  source-namespaced pieces (`<source>:<piece-path>`, e.g. `lib:button.primitive`,
  `login:view.mobile`). Shared `lib:*` design-system pieces exist once.
- Figma snapshots move to `product/design/sources/<source>/`
  (`source.md` / `context.md` / `tokens.json`); per-source metadata
  (`figmaFileKey` / `fileVersion` / `fetchedAt`) lives in `source.md`, not the
  ledger.
- `/dev:figma-refresh-plan <source>` now **only** pulls one source and updates
  the global ledger, printing an open-piece overview — it no longer writes a UI
  plan.
- `/dev:plan` now writes the UI plan, selecting open ledger pieces
  conversationally and comparing code against the pinned snapshot. Two-stage per
  source: `/plan` (mapping) → `figma-refresh-plan <source>` → `/plan` (select +
  UI plan) → `/implement`.
- One-time automatic migration of legacy per-plan ledgers into the global
  ledger (relocation, not drift: baselines and acceptances carried forward).

**Out of scope:** the deterministic hash (`ledger-hash.mjs`), status flags, the
soft dependency cascade, and acceptance semantics are unchanged.
```

- [ ] **Step 2: Bump the plugin version.**

In `dev-workflow/.claude-plugin/plugin.json`, change `"version": "2.12.0"` to `"version": "2.13.0"`.

- [ ] **Step 3: Final cross-file invariant check.**

Run:

```bash
grep -rn "plan-slug" dev-workflow/ && echo "FOUND STALE — FIX" || echo "clean: no plan-slug references"
```

Expected: `clean: no plan-slug references`.

Run:

```bash
grep -rln "product/design/status.json" dev-workflow/commands/ dev-workflow/README.md
```

Expected: `figma-refresh-plan.md`, `plan.md`, `implement.md`, `figma-accept.md`, `README.md` all listed.

- [ ] **Step 4: Commit.**

```bash
git add dev-workflow/CHANGELOG.md dev-workflow/.claude-plugin/plugin.json
git commit -m "Release 2.13.0: global design ledger"
```

---

## Self-Review notes

- **Spec coverage:** storage layout → Task 1 (steps 5, 9) + Task 6; ledger schema/identity → Task 1 (steps 6, 7); command roles → Tasks 1, 3, 4, 5; "what's open is a projection, no file" → Task 1 (step 8); two stock-takings → Tasks 1 & 3; lifecycle → Task 1 (step 2), Task 3 (step 1), Task 6 (step 4); migration → Task 2; out-of-scope preserved (hash/acceptance untouched) → constraints + Task 7 CHANGELOG.
- **Type/string consistency:** all tasks use the Global Constraints strings verbatim (`product/design/status.json`, `product/design/sources/<source>/`, `<source>:<piece-path>`).
- **No placeholders:** every edit shows the exact old and new text or a precise line-anchored rule; verification steps are concrete `grep` commands with expected output.
