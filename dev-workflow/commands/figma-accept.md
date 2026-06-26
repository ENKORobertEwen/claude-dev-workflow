# Figma Accept Command

Record **human acceptance** of implemented UI pieces in the design ledger.
Building a piece and passing the automated visual review is not the same as a
tester signing off — this command captures the sign-off.

Acceptance is **soft**: it never blocks anything. A PR can merge and a plan can
move to `done/` with pieces still awaiting acceptance; they simply stay visible
as `awaiting-acceptance` until a tester accepts them here.

## The acceptance model

Each piece in `product/design/<plan-slug>/status.json` carries, alongside its
implementation fields, an acceptance record:

- `acceptedHash` — the implemented hash a tester signed off on
- `acceptedBy` — who accepted
- `acceptedAt` — when

A piece is **accepted** only when `acceptedHash == lastImplementedHash` (the
tester reviewed exactly what is built). Derived states:

- `lastImplementedHash != currentHash` (or null) → not built yet:
  `to-implement` / `to-review`
- built, but `acceptedHash != lastImplementedHash` → **awaiting-acceptance**
- built, and `acceptedHash == lastImplementedHash` → **accepted** (truly done)

Because acceptance is tied to the implemented hash, any later design change that
triggers a rebuild produces a new hash — so the piece automatically returns to
`awaiting-acceptance` and must be re-accepted. The tester never silently signs
off on a stale version.

**Hash-spec migrations do not reopen acceptance.** Hashes are produced solely by
`/dev:figma-refresh-plan` via its committed `ledger-hash.mjs` reference script.
If that script's algorithm is versioned up, the stored hash representation
changes without any real design change; `/dev:figma-refresh-plan` re-baselines
the ledger and **carries accepted pieces forward** (migrates `acceptedHash`
alongside `lastImplementedHash`) so testers are not asked to re-accept unchanged
UI for a purely internal reason. Only a genuine design change reopens acceptance.
This command never computes or migrates hashes itself — it only copies
`lastImplementedHash` into `acceptedHash` on sign-off.

## Input

Determine the ledger from the current plan (its `product/design/<plan-slug>/`).
Then determine which pieces to accept:

1. **Explicit pieces** — `/dev:figma-accept <pieceId> [<pieceId> …]`: accept
   exactly these.
2. **From a PR** — `/dev:figma-accept --from-pr <number>`: read the PR's
   acceptance checklist (see `/dev:implement`'s PR body) and accept every piece
   whose box is checked. Requires `GITHUB_TOKEN` / `gh`.
3. **Interactive** — `/dev:figma-accept` with no args: list all
   `awaiting-acceptance` pieces with their `codeTarget` and the live URL, and
   ask the tester which to accept. (If fully headless, accept none and just
   report the list.)
4. **All** — `/dev:figma-accept --all`: accept every `awaiting-acceptance`
   piece. Use only when the tester has reviewed everything.

## Process

1. Read `status.json`. Build the list of target pieces from the input.
2. For each target piece:
   - If it is not built (`lastImplementedHash` is null or `!= currentHash`),
     **skip it** and warn — you cannot accept a piece that isn't implemented
     against the current design.
   - Otherwise set `acceptedHash = lastImplementedHash`, `acceptedBy` (the git
     user, or a name passed by the caller), and `acceptedAt` (current
     timestamp).
3. Write `status.json` back.
4. Commit: `Accept design pieces: <ids or summary>`.
5. Report: how many pieces are now `accepted`, and how many remain
   `awaiting-acceptance` / `to-implement` / `to-review`.

## Hardcoded Operational Decisions

| Scenario | Behavior |
|----------|----------|
| Piece not yet built | Skip + warn. Cannot accept an unbuilt piece. |
| Piece already accepted at the current hash | No-op for that piece; note it. |
| `--from-pr` but no `GITHUB_TOKEN` / `gh` | Report that PR sync needs a token; fall back to explicit/interactive. |
| No `awaiting-acceptance` pieces | Report "nothing awaiting acceptance." No commit. |
| No ledger / no Figma-sourced design | Report "no design ledger to accept against." Stop. |
| Nothing actually changed | No commit (no no-op commits). |

## Critical Rules

1. **Acceptance is soft.** This command never blocks merges, plan completion, or
   anything else — it only records sign-off.
2. **Accept only what's built.** `acceptedHash` is always set to
   `lastImplementedHash`; never accept a piece that isn't implemented against
   the current design.
3. **Acceptance is hash-bound.** A rebuild after a design change re-opens
   acceptance automatically — this command does not pre-accept future states.
4. **No no-op commits.**
