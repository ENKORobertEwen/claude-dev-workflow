# Changelog — dev plugin

## 2.16.0 — Design-fidelity hardening (learnings from the quote-block audit pilot)

**Problem fixed:** a Figma-sourced implementation passed all checks and the UI
review, yet visible deviations from the Figma frame survived to human review.
Root causes were systemic in the command design: (1) the implementer classified
a visible deviation as "reported" through a too-broad escape hatch; (2) it cited
a test it had itself written earlier as the reason not to adopt the Figma asset;
(3) the phase prompt mentioned the downstream human sign-off, so deferring was
rational; (4) the UI reviewer was told to ignore the implementer's
"deviation-reported" list — the auditor decided what its own reviewer got to
see; (5) no plan reviewer ever looked at the design source, so a plan-level
test-vs-frame contradiction went unseen.

**Changes:**
- **`/dev:plan` gains Reviewer 6: Design Fidelity** (design-sourced frontend
  plans only). It uses the Figma MCP (`get_screenshot`, `get_design_context`,
  `get_metadata`) to check the plan against the actually referenced nodes:
  verify every design-derived claim, find test-vs-design contradictions,
  enumerate completeness from the frame outward, and audit any
  classification/exception rule for loopholes that let a visible deviation
  survive.
- **`/dev:plan` ATDD-coverage reviewer:** acceptance tests must not pin
  implementation details that constrain design fidelity (exact text-node
  structure, concrete glyph characters, text vs. vector asset). Tests pin
  behavior and structural invariants; design truth lives in Figma.
- **`/dev:implement` implementation sub-agent (frontend mode):** the phase
  prompt is filtered — no mention of downstream reviews, sign-off, or
  `figma-accept` reaches the implementer. It is told it is the final authority
  ("a deviation you do not fix will not be fixed by anyone else"), the
  token-mismatch case is the ONLY reportable deviation, and a test — even a
  self-written one — never blocks a fidelity fix: the test is changed, not the
  fix dropped.
- **`/dev:implement` UI review sub-agent runs blind:** it never sees the
  implementer's report or any "known/accepted" deviation list. For
  Figma-sourced phases it pulls reference screenshots of the mapped nodes via
  the Figma MCP and reports EVERY visible deviation. Whether a deviation is
  acceptable is decided exclusively by the human at sign-off — never by the
  agent that produced it.

## 2.15.0 — Host-aware PR creation (GitHub + Azure DevOps)

**Problem fixed:** `/dev:implement` step 6 was "Create Pull Request (GitHub
only)" — hardcoded to `GITHUB_TOKEN` + the GitHub REST API. On any other host
(e.g. Azure DevOps) it always fell into the "push + create the PR manually"
branch, and `/dev:figma-accept --from-pr` (GitHub-only) could not sync
acceptance at all. Nothing broke, but there was no automation off GitHub.

**Changes:**
- **`/dev:implement` step 6 is now host-aware.** It detects the host from the
  `origin` remote and creates the PR on the matching platform:
  - **GitHub** — `gh pr create` (CLI handles auth), falling back to the
    `GITHUB_TOKEN` REST API.
  - **Azure DevOps** — `az repos pr create --detect true` (reads the org from
    the git remote; project/repo from git config); auth via
    `AZURE_DEVOPS_EXT_PAT` or `az login`.
  - **Unknown host / missing CLI / no credential** — pushes the branch and
    hands off to a manual PR, naming the credential that would enable
    automation. The PR body is identical across hosts.
- **`/dev:figma-accept --from-pr` is host-aware** too: GitHub via
  `gh pr view --json body` (or REST), Azure DevOps via
  `az repos pr show --id … --detect true`; falls back to explicit/interactive
  when the host CLI/token is unavailable.

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
