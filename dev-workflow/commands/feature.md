---
description: Refine a feature on the business level and store it as work items (Feature + PBIs with clean descriptions and acceptance criteria) — the input /dev:plan consumes
argument-hint: [feature idea | work item ID to refine]
model: claude-fable-5
---

# Feature Command

**Model guard:** This command is designed to run on **Fable 5** (`claude-fable-5`) — the
refinement conversation itself is the deliverable. At the start, check which model you are
running as (your system prompt states it). If it is not Fable 5, tell the user before doing
anything else and recommend switching via `/model claude-fable-5`; continue only if they
decline.

You are refining a feature on the **business level** and storing the result as work items in
the project's work item tracker (Azure DevOps unless the project docs say otherwise):

- **One Feature work item** — business-level description (value, scope, non-goals) plus
  acceptance criteria.
- **Product Backlog Items under it, written as user stories** — each with a clean description
  and individually testable acceptance criteria.

The result must be **decision-complete on the business level**: when `/dev:plan` later picks
these work items up, it should only have to make *technical* decisions (architecture,
libraries, data models, phasing). Every business question — scope, roles, rules, edge-case
policies, gates, non-goals — is resolved here, in conversation with the user.

This command does NOT write a development plan and does NOT change any code. It ends with
work items and a hand-off to `/dev:plan`.

## Input

Either:

- **A feature idea** — free text describing what the user wants. You develop it from scratch.
- **An existing Feature work item ID** — you load it, treat its current description as the
  starting point, and refine it: sharpen the description/criteria and create (or complete)
  the PBI breakdown underneath. Refinement is non-destructive until approval: existing
  field content is only replaced after the user approves the new version in step 6.

## Process

### 1. Load Project Context

1. Read the project's `CLAUDE.md` for orientation — including any **mandatory gates** it
   defines for planning/implementation (e.g. compatibility or cutover gates). Those gates
   apply to this command too: the business consequences of a gate (what must not break, what
   needs a manual approval) are business decisions and belong in the work items.
2. Read the project's work-item-tracker documentation — by default
   `product/docs/azure-devops-structure.md`, or whatever equivalent the project's CLAUDE.md
   names. From it, extract: organization/project URL, the epic list (top-level structure),
   the iteration/labelling conventions for new work items, the work-item state models, and
   the documented API mechanics (auth, encoding pitfalls, verified snippets).
   - **If no such doc exists:** ask the user for organization + project, verify access with
     one read call, and seed the doc (in the project's documented docs location) with what
     you learned — organization/project, hierarchy, conventions. Do not silently work from
     memory.
3. Read `product/DDD/glossary.md` and `product/DDD/context-map.md` (if present) — the work
   items must use the project's ubiquitous language, not invented vocabulary.
4. Skim `product/ADR/` for decisions that constrain the feature's business scope.
5. If refining an existing work item: fetch it via the API **with relations** (parent epic,
   existing children, attachments) and read its current description and acceptance criteria.

### 2. Understand the Feature — Business-Level Brainstorming

Work like a good product owner preparing refinement:

- Ask clarifying questions **one at a time**; prefer selection boxes (`AskUserQuestion`)
  with a recommended option over open-ended questions. Multiple rounds are expected.
- **Scout the codebase where facts beat opinions.** When a business decision depends on the
  current state of the product (which surface exists, what is dead, what already works a
  certain way), establish the facts by reading code/config/docs first and present them as
  evidence with the question — never make the user guess about their own product, and never
  decide from assumption what can be decided from evidence.
- Typical questions to resolve: What problem does this solve, and for whom? What is in
  scope, what is explicitly out (non-goals)? Which roles/actors are affected? What are the
  business rules, including edge cases? Where are grey zones, and what is the explicit rule
  for them (e.g. "when in doubt, keep it")? Are there sequencing constraints or manual
  approval gates? What does "done" look like, observably?
- Check the feature against the project's mandatory gates from step 1 and record the verdict
  and its business consequences (e.g. "destructive step only after X, manual approval by Y").

**Boundary discipline — business vs. technical:** this command decides the business level
only. Technical choices (architecture, endpoints, schemas, libraries, phasing) belong to
`/dev:plan`. But business constraints that *bind* later technical work (compatibility
promises, gates, sequencing, data-retention rules) are decided here and written into the
work items. If the user starts making a technical decision, capture it as a constraint note
for `/dev:plan` rather than baking implementation detail into story texts.

### 3. Propose the Breakdown

When the picture is clear, propose — in chat, not yet in the tracker:

1. **The Feature** — title plus business-level description: the value ("why"), the scope
   ("what"), explicit non-goals, agreed decisions (including gate verdicts), and
   feature-level acceptance criteria (the observable outcomes that make the feature "done").
2. **The PBI cut** — each PBI as a user story slice with a working title. Aim for slices
   that each deliver observable value or a verifiable intermediate state, are independently
   plannable, and together cover the feature. Name dependencies between PBIs explicitly.
3. **Parent epic** — propose which epic the Feature belongs under (from the doc's epic
   list) and say why.

Iterate with the user until the cut is agreed.

### 4. Write the Story Texts

For every PBI, write:

- **Description** (user-story form):
  - First line: "Als *Rolle* möchte ich *Ziel*, damit *Nutzen*." (in the project's work-item
    language — match the language of the existing work items; user-story form regardless of
    language).
  - Then: business context and the agreed decisions this story embodies — enough that a
    reader understands the story without the chat history.
  - Constraint notes for `/dev:plan` where applicable (gates, compatibility promises,
    sequencing).
- **Acceptance criteria** (separate field, see step 6): a numbered list of individually
  testable statements. Use "Wenn X, dann Y" / "When X, then Y" phrasing where it fits;
  plain verifiable assertions where it doesn't. Each criterion must be checkable in
  isolation — no "works correctly", no "as discussed".

For the Feature itself, write the description from step 3 plus feature-level acceptance
criteria (typically: the sum of its PBIs' outcomes, any cross-cutting outcomes, and the
gates' observable conditions).

**Quality bar (self-check before presenting):**

- No placeholder ("TBD", "tbd.", "noch zu klären") anywhere.
- Every criterion individually testable; two readers would judge pass/fail the same way.
- Business language throughout — code identifiers only where they *are* the business fact
  (e.g. naming a legacy surface that is being removed).
- Grey zones have explicit rules, not silence.
- `/dev:plan` could start from these texts without asking the user a single business
  question. That is the definition of done for this command.

### 5. Approval Gate

Present the complete result in chat: Feature text, every PBI with description and acceptance
criteria, parent epic, and — when refining — a clear before/after statement of what will be
replaced. Then use `AskUserQuestion` with two options:

- **"Anlegen" / "Create (Recommended)"** — proceed to step 6.
- **"Ich habe Änderungen" / "I have changes"** — incorporate feedback, present again.

Do NOT write anything to the tracker before this approval.

### 6. Write the Work Items

Follow the API mechanics documented in the project's tracker doc. For Azure DevOps the
proven mechanics are:

- **Never pass non-ASCII text as inline CLI arguments** (`az.exe` mangles them). Build a
  **JSON-Patch document as a UTF-8 file** — generate it with a script (e.g. `python3` +
  `json.dump`) rather than hand-written JSON, so quoting/escaping cannot break — and send it
  with `curl` + bearer token:
  `TOK=$(az account get-access-token --resource 499b84ac-1321-427f-aa17-267ca6975798 --query accessToken -o tsv | tr -d '\r')`
  then `curl -s -X POST -H "Authorization: Bearer $TOK" -H "Content-Type: application/json-patch+json" --data-binary @patch.json "https://dev.azure.com/{org}/{project}/_apis/wit/workitems/\$FEATURE_TYPE?api-version=7.1"`
  (URL-encode the type: `$Feature`, `$Product%20Backlog%20Item`).
- **Fields:**
  - `System.Title`
  - `System.Description` — HTML (`<p>`, `<ul>`, `<b>`)
  - `Microsoft.VSTS.Common.AcceptanceCriteria` — HTML numbered list. This field exists on
    Feature AND Product Backlog Item in the Scrum process; if the project's process lacks it
    on a type, append an "Akzeptanzkriterien" section to the description instead.
  - `System.IterationPath` / `System.AreaPath` / tags — per the project's documented
    conventions (e.g. CustomerWallet: iteration `CustomerWallet\Release 3.0`, **no** line
    tags, **no** plan tag — plan tags are `/dev:plan`'s job).
- **Hierarchy:** parent links via a `System.LinkTypes.Hierarchy-Reverse` relation on the
  child (PBI → Feature, Feature → Epic).
- **State:** leave new items in the initial state (`New`). `Approved` is set by `/dev:plan`
  when a plan covers the item — not by this command.
- **Refining existing items:** update `System.Description` / acceptance criteria with
  `op: replace` (or `add` if the field was empty). **Never** rewrite `System.Tags` casually —
  `op: add` on tags merges old and new; removals need `replace` plus verification.
- Create the Feature first, then the PBIs with their parent link, so the hierarchy is right
  from the start.

### 7. Verify and Report

Read every created/updated item back via GET (curl + bearer, never `az` stdout for
content) and check:

- parent link correct (PBIs → Feature, Feature → Epic),
- iteration path per convention,
- description and acceptance criteria non-empty and **umlauts/special characters intact**,
- state as intended.

Then report to the user: every item as `#ID Title` with its browser URL
(`https://dev.azure.com/{org}/{project}/_workitems/edit/{id}`), and close with the hand-off:
the work items are ready as input for `/dev:plan`.

## Critical Rules

1. **Business level only.** This command produces refined work items — no development plan,
   no code changes, no ADRs, no DDD edits. Those happen in `/dev:plan` / `/dev:implement`.
2. **Decision-complete for `/dev:plan`.** The definition of done: `/dev:plan` needs no
   further business decisions from the user. Every scope boundary, rule, grey zone, and gate
   is written into the work items.
3. **Evidence before assumption.** When the current product state matters to a business
   decision, read the code/docs and present facts — don't let the conversation run on guesses.
4. **One question at a time, selection boxes preferred.** Like `/dev:plan`, the conversation
   is the tool; don't overwhelm the user.
5. **Nothing is written before approval.** The full breakdown (Feature + all PBIs + criteria)
   is approved in chat first. Refinement replaces existing content only after approval.
6. **Follow the project's documented tracker conventions** (iteration, tags, states, API
   mechanics) — and seed that documentation if it doesn't exist yet. No convention decisions
   from memory.
7. **Verify what you wrote.** Every item is read back and checked (hierarchy, fields,
   encoding) before you report success.
8. **Work items speak the project's language** — the language the existing work items use
   (CustomerWallet: German), in the project's ubiquitous language from the DDD glossary.
