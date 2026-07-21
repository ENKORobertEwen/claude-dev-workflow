---
description: Interactive UI polish session — the user reports design deviations one at a time, each becomes a tracked Task work item, you implement it against a running local app
argument-hint: [work item ID of the PBI or Bug this session belongs to]
model: claude-opus-4-8
---

# UI Design Fix Session

An interactive session in which the user compares the running app against a design and
reports deviations, **one message at a time**. Every reported item becomes a **Task work
item** under a parent work item, so the user can watch progress live on the Azure DevOps
board instead of scrolling chat.

This command is for **visual/UI refinement of existing screens**. It is not a planning
command and not a feature command: there is no plan document, no phase structure, no
sub-agent orchestration. It is you, a running app, and a stream of small corrections.

## Input

`$ARGUMENTS` is the **ID of the parent work item** — normally a Product Backlog Item, or a
Bug. It is context, not a task list: it tells you which feature the session belongs to and
where the Tasks you create must hang.

If no ID was given, ask for one before doing anything else. Do not start without a parent —
untracked Tasks defeat the purpose of this command.

## Setup (before the first report)

1. **Load the parent work item.** Read its title, description and acceptance criteria so you
   know what the session is about. Confirm to the user in one line what you loaded.

2. **Read the project's design and styling contract.** At minimum the project `CLAUDE.md` and
   any sub-directory `CLAUDE.md` for the app you are touching, plus whatever design reference
   the project documents (design system, capture, ADR on theming). You must know **where a
   colour, a spacing value or a component style is allowed to live** before you change one.

3. **Ask for the go.** Then, and only then, **start the app locally** using the project's
   documented command (e.g. `./do run <target>`). The user must be able to check every change
   in their own browser immediately — that is the whole point of the session. Run it in the
   background and confirm the URL.

4. State the working rules back to the user in three lines: one item at a time, each becomes a
   Task, backend work gets parked. Then wait.

## The loop

For each thing the user reports:

### 1. Create the Task — before you touch code

Create a **Task** work item, parented to the input work item:

- **Title:** the user's report, condensed to one line.
- **Description:** their wording verbatim, plus the screen/route it concerns.
- **State:** `To Do` on creation.

Then set it to **`In Progress`**. The user is watching the board; the state must reflect
reality at all times, not be corrected in a batch afterwards.

If one message contains several independent deviations, create **one Task per deviation** —
they will be verified separately, so they must be trackable separately.

### 2. Find the target value — do not guess it

Read the design reference for the concrete value (colour, size, spacing, icon path, layout).
If you cannot find it, say so and ask. **Never infer a value from a screenshot** and never
explain a gap with an assumption you have not checked — verify, or say you did not.

### 3. Implement at the right tier

Fix the **cause**, not the symptom, and put the change where the project's styling contract
says it belongs:

- A value used by more than one screen belongs in the **theme/token layer**, never in a page.
- A control that appears on more than one screen belongs in a **shared component/primitive**,
  never re-styled per page.
- Only genuinely page-local layout belongs in the page's own stylesheet.

**Before styling any control, check whether a primitive for it already exists** — and if the
design system defines one and the project does not have it yet, build the primitive instead
of styling the instance. Styling the same control twice on two pages is the failure mode this
rule exists to prevent.

If a fix is a base-layer change (theme, tokens, shared component), say so in your report:
it silently affects every other screen, which is usually intended but must be visible.

### 4. Verify before you report

Run the project's fast checks for the area you touched (lint, unit tests, any style/architecture
guards) and confirm the app rebuilt cleanly. **Do not report an item as done on the strength of
having edited a file.** If a check is red, fix it or report it as not done.

Note that a dev server log contains the errors of *earlier* builds too — confirm the **latest**
build is clean, not merely that a clean build appears somewhere in the log.

### 5. Close the Task and report

Set the Task to **`Done`**, then reply in exactly three parts:

- **Umgesetzt** — what changed, and in which files/tier.
- **Nicht umgesetzt** — anything from the report you did not do, with the reason.
- **Braucht deine Entscheidung** — open questions, with a recommendation.

Keep it short. Any consequence the user would not expect — a changed default, a behaviour
change, an accessibility trade-off — belongs in the report even when it was the right call.

Then wait for the next report. **Do not start the next item while one is open**: the user
verifies each change in the browser, and overlapping changes make a regression unattributable.
If they send several messages in a row, finish the current one, report it, then take the next.

## Parked work: anything you cannot do in this mode

Some findings cannot be fixed in a live UI session — most commonly **backend changes**, which
need a rebuild and restart and would interrupt the user's checking loop.

When you hit one:

1. Create a **Task** under the same parent, titled so it is recognisable as parked
   (e.g. prefix `[Backend]`).
2. Describe the finding, the evidence, and what would have to change.
3. Leave it in **`To Do`** — do not start it, do not fix it in passing.
4. Mention it in your report for that round.

The same applies to anything else that would break the loop: a dependency upgrade, a
migration, a change requiring the app to be restarted in a different configuration.

## Hard rules for this mode

- **No browser automation during the session.** The user checks in their own browser; driving
  a headless browser yourself is slower than their glance and adds nothing. It is reserved for
  a **final pass at the end of the session, only after the user asks for it**.
- **Sequential.** One item open at a time. No parallel sub-agents editing the working tree —
  the shared theme/base files and the single dev server make concurrent edits unattributable.
- **The board is the record.** Every reported item is a Task, and its state matches what you
  are actually doing at that moment.
- **Never claim done without a check that passed.** Reporting an unverified fix costs more
  trust than saying "this one is not finished yet".

## Ending the session

When the user signals the end:

1. Confirm no Task is left in `In Progress`.
2. Run the project's **full verification gate** (e.g. `./do check`) and get it green. If it is
   red, fix it or report precisely what is red — do not commit over it.
3. Commit and integrate per the project's branching rules, referencing the parent work item.
4. Summarise: Tasks done, Tasks parked, and anything still open.

**Commit during the session, not only at the end.** A long session accumulates a large
unverified working tree; propose an intermediate commit whenever a coherent block is green.
