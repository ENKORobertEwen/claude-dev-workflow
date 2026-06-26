# Changelog — dev plugin

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
