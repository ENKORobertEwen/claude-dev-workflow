# Changelog — dev plugin

## 2.14.0 — Stop the drift cascade at the token abstraction boundary

**Problem fixed:** the soft cascade in `/dev:figma-refresh-plan` flipped every
(transitive) `dependsOn` consumer of a `to-implement` piece to `to-review`.
Because token pieces hash over their values, a pure design-variable value change
(e.g. a global font size) flipped the token piece and cascaded onto every
consuming control/block/view. In a token-referencing codebase (`var(--token)`)
consumers inherit the new value for free and need no code change — so the
cascade produced only red noise and made the ledger unusable. It ignored the
abstraction boundary the variables establish.

**Principle:** a consumer becomes `to-review` only when a change could force it
to change *code* — structure/interface/binding, not value. Values flow through
the variable.

**Changes:**
- **Cascade:** `token`-level pieces are no longer a cascade seed. A token
  **value** change moves only the token piece (`to-implement`, regenerate the
  token layer); consumers stay `implemented`. Component→component edges still
  cascade — a dependency's *structural* change is the only thing that makes it
  `to-implement` in the first place. (Doc-only; no hash change for this part.)
- **Hash basis (`hashSpec` v1 → v2):** the per-node descriptor gains a `tokens`
  field — the sorted set of design-variable *identifiers* the node BINDS
  (identifiers only, never values, sourced from `get_design_context`). This
  makes a token **rebinding** (a consumer now references a *different* token) a
  direct structural drift, while a token **value** change stays invisible in the
  consumer's hash. The reference `ledger-hash.mjs` is field-agnostic and
  unchanged; only the descriptor and the worked example's SHA move.

**Migration (automatic, safe):** existing ledgers are `hashSpecVersion` 1 ≠ 2,
so the next `/dev:figma-refresh-plan` routes them through the existing step-4.4
re-baseline — baselines and acceptances carried forward, no false drift.

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

## 2.12.0 — Deterministic design-ledger hash

**Problem fixed:** `/dev:figma-refresh-plan`'s drift detection used a per-piece
SHA-256 specified only in prose, so every run was a model re-interpreting the
canonicalization — two runs produced different hashes for the *same* unchanged
design. The next refresh would then mark every implemented piece as false drift
and write a bogus rework plan, destroying the baseline. The hash input was also
inconsistently specified against `get_design_context`, whose output is unstable.

**Changes:**
- Hashing is now produced by a committed reference script
  (`product/design/bin/ledger-hash.mjs`), materialized into the project on first
  run — the single source of truth. Identical design ⇒ identical hash across
  sessions, runs, and MCP response ordering (sorted keys, sorted arrays,
  fixed-precision sizes).
- Input source pinned to **`get_metadata`** (id, name, type, size, variant
  structure) — explicitly NOT `get_design_context` (screenshots, generated code,
  asset URLs, x/y positions vary run-to-run). Multi-node pieces and DTCG token
  pieces have byte-exact rules; worked example included in the command doc.
- Algorithm versioned via `hashSpecVersion` / `hashSpec` in `status.json`.

**Migration behavior for existing ledgers (automatic, safe):**
- A ledger with no `hashSpecVersion`, or one that differs from the current spec
  (including v2.10.0 ledgers seeded with the old unrecoverable prose method,
  `adoptionSeed: true`, or null hashes), is **re-baselined**, never treated as
  drift: the next `/dev:figma-refresh-plan` recomputes every hash with the new
  script, sets `lastImplementedHash = currentHash` for implemented pieces, sets
  `hashSpecVersion`, commits "migrated", and writes **no** rework plan. Re-run
  the command afterward for normal drift detection.
- **Acceptance is carried forward:** a piece accepted under the old spec
  (`acceptedHash == lastImplementedHash`) has its `acceptedHash` migrated to the
  new hash, so testers do not re-accept unchanged UI for a purely internal algo
  change. Genuine design changes still reopen acceptance on normal runs.
- Requires Node.js for the reference script; the command stops with a clear
  message if `node` is unavailable rather than improvising hashes.

## 2.11.0 — Lifecycle order corrected
`/dev:plan` → `/dev:figma-refresh-plan` → `/dev:implement`. `figma-refresh-plan`
runs after plan (it needs the mapping plan resolves) and is the single source of
UI plans (first build and rework alike).

## 2.10.0 — Human acceptance
Added `/dev:figma-accept` and an acceptance dimension in the ledger
(`acceptedHash`/`By`/`At`), soft (never blocks merges/plan completion).

## 2.9.0 — Figma design ledger
Added `/dev:figma-refresh-plan`: pull Figma design, maintain a layered
per-piece status ledger, write UI plans; plus design-to-code guardrails in
`/dev:implement` and the visual-review loop.
