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

`$ARGUMENTS` is a **work item ID**. It is context, not a task list: it tells you which piece
of work the session belongs to and where the Tasks you create must hang.

Resolve it to a parent before doing anything else — **load the work item and read its type**,
do not assume it from the number:

- **Product Backlog Item or Bug** → this is the parent. Use it directly.
- **Feature** → a Feature is too coarse to hang Tasks off. **Create a Product Backlog Item
  under it** for this round of work, and use that PBI as the parent. Propose title and
  description to the user and get approval before creating it — the PBI is the record of what
  this session was for, so its wording matters. Title it after the work, not after the
  session (e.g. "Beitragsübersicht auf das Design-System angleichen").
- **Anything else** (Epic, Task, …) → say why it does not work as a parent and ask for the
  right one.
- **No ID given** → ask which work item the session belongs to. Do not start without a
  parent; untracked Tasks defeat the purpose of this command.

State clearly which work item ended up as the parent before the first report comes in.

## Setup (before the first report)

1. **Resolve and load the parent** as described under *Input* — including creating the PBI
   first if a Feature was given. Read its title, description and acceptance criteria so you
   know what the session is about. Confirm in one line which work item is now the parent.

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

## Execution model: two roles, one writer

You are the **orchestrator**. You stay with the user at all times — you must never be the one
stuck inside a long build so that their next report has to wait. Implementation runs in a
sub-agent.

**You do yourself, inline:** everything work-item related (create, set state, link) and every
exchange with the user. A work item state change is a single fast call, and the board is the
record of this session — it must be right at the moment it is set, not repaired later by a
helper that may fail. Do **not** spawn an agent for it.

**The implementation agent does:** the code change and its verification.

### One long-lived implementation agent

Spawn **one** implementation agent for the session and **keep it alive across tasks**
(continue it rather than starting a new one). The items in this mode are small corrections,
and re-reading the design reference, the styling contract and the primitives inventory for
every two-line fix costs more than the fix.

> This deliberately deviates from `/dev:implement`, which mandates a fresh agent per phase.
> That rule protects long, independent implementation phases from context bleed. Here the
> opposite is wanted: the accumulated knowledge of the design and of what was already changed
> is the point. The deviation is scoped to this command.

**Restart it if it drifts** — if it starts contradicting earlier findings, re-solving settled
questions, or its reports get vague, drop it and start a fresh one with a short summary of
what has been decided so far. A stale agent is worse than a cold one.

**Never run two implementation agents at once.** They would write to the same working tree —
in particular the shared theme/base files — and to a single dev server, which makes both file
state and build results unattributable.

### The queue

When a report arrives while an item is still open:

1. **Create its Task immediately** and leave it in `To Do`. The user sees it on the board at
   once, which is what keeps them unblocked.
2. **Do not dispatch it** until the current item is reported done.
3. Then set it to `In Progress` and hand it to the agent.

Tell the user briefly that you queued it. Never silently drop a report because you were busy.

### Evidence, not assurances

The implementation agent must return, for every task: **which files it changed**, the
**output of the checks it ran**, and confirmation that the **latest** build is clean. A claim
of "done" without that is not a result.

You are accountable for what you report to the user, not the agent. Spot-check its claims —
at minimum that the files it names actually changed and that the last build really is green.
Relaying an unverified report is the failure this rule exists to prevent.

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

### 3. Hand it to the implementation agent

Dispatch the task with a **context package** — do not make the agent re-derive what you
already know: the target value you looked up in step 2, the tier rule that applies, the
relevant part of the styling contract, and which primitives already exist. On later tasks
this shrinks to the delta, since the agent still holds the rest.

The agent fixes the **cause**, not the symptom, and puts the change where the project's
styling contract says it belongs:

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

The agent runs the project's fast checks for the area it touched (lint, unit tests, any
style/architecture guards) and confirms the app rebuilt cleanly — and returns that output as
evidence. **Never report an item as done on the strength of an edit having been made.** If a
check is red, it gets fixed or the item is reported as not done.

Note that a dev server log contains the errors of *earlier* builds too — confirm the **latest**
build is clean, not merely that a clean build appears somewhere in the log.

### 5. Hand the Task over to test and report

Set the Task to **`Ready - In Test`** — that state is the signal the user watches for: it
means "implemented, verified, committed — you can test this in the browser now". Do **not**
set `Done`: closing the Task is the user's move, after their own test passes.

Then reply in exactly three parts:

- **Umgesetzt** — what changed, and in which files/tier.
- **Nicht umgesetzt** — anything from the report you did not do, with the reason.
- **Braucht deine Entscheidung** — open questions, with a recommendation.

Keep it short. Any consequence the user would not expect — a changed default, a behaviour
change, an accessibility trade-off — belongs in the report even when it was the right call.

Then wait for the next report. **Do not start the next item while one is open**: the user
verifies each change in the browser, and overlapping changes make a regression unattributable.
If they send several messages in a row, finish the current one, report it, then take the next.

## Failed tests come back: the re-test loop

The user tests each `Ready - In Test` Task in their own browser. Two outcomes:

- **Pass** → the user sets the Task to `Done` themselves. Nothing for you to do.
- **Fail** → the user moves the Task back to **`In Progress`** and typically leaves a
  **comment** on the work item saying what is still wrong.

A Task in `In Progress` that you did not just dispatch is therefore a **returned item** —
and it outranks the queue: fix it before starting the next new report. A fix the user is
waiting to re-test beats a deviation nobody has looked at yet.

**Check for returned items at every natural pause**: after handing an item over, before
dispatching the next queued one, and when the user goes quiet for a while. Query the
parent's child Tasks for state `In Progress`, excluding the one currently dispatched. For
each hit, read the work item's **comments** (the comments API — the description will not
contain the feedback). Comment missing or unclear → ask the user instead of guessing.

A returned item runs through the same loop as a fresh report: dispatch to the
implementation agent (the user's comment plus the original context package is the brief),
verify, commit as a **follow-up commit referencing the same Task**, set it back to
`Ready - In Test`, and report what changed this time.

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

1. Confirm no Task is left in `In Progress` — a returned item still open means the session
   is not finished. Tasks in **`Ready - In Test`** may remain: the user closes them as
   their tests pass, possibly after the session ends. List them explicitly in the summary.
2. Run the project's **full verification gate** (e.g. `./do check`) and get it green. If it is
   red, fix it or report precisely what is red — do not commit over it.
3. Commit anything still uncommitted and integrate per the project's branching rules,
   referencing the parent work item.
4. **Push, then wire the commits to their Tasks** — this is the point at which the links
   resolve, and the reason they were not created earlier. For each Task handed over during
   the session, attach its commit to the work item the way the project documents it, and follow
   whatever the project's work-item lifecycle requires (PR links, states).
5. Summarise: Tasks done, Tasks still in `Ready - In Test`, Tasks parked, and anything
   still open.

**Commit after every finished Task — by default, without asking.** One Task is one commit. Do it
as part of handing the item over, in the same breath as setting the state to `Ready - In Test`: the narrow check
has just passed, the change is small, and its scope is exactly one reported deviation. Waiting
for "a coherent block" is what produces the large uncommitted tree this rule exists to prevent.

Reference the Task in the commit message so the two can be matched later. Fold a Task's commit
into the previous one only when the item turned out to be a correction of the item before it.

**Do not link commits to work items during the session.** A "fixed in commit" link is a
reference to a commit object *in the server repository*; while the commit is local only, that
object does not exist there. The platform will not validate the SHA, so the link can be created
— and will point at nothing. Wire commits to their Tasks **after pushing**, in one pass, as part
of ending the session. Until then the commit message is the link.

### The verification gate for this mode

This mode overrides the project's usual "full gate before every commit" rule — **for local
commits made during the session only**. It does not change the rule anywhere else.

| Point | What must pass |
|---|---|
| While iterating on an item | nothing — run whatever gives you fast feedback |
| **Local commit during the session** | the **narrow target** covering the area you touched (e.g. `./do test webmanager`), plus any style/architecture guards that the narrow target does not run |
| **Pushing, opening a PR, or ending the session** | the project's **full gate** (e.g. `./do check`), green |

The reason for the relaxation is observed behaviour, not convenience: when every intermediate
commit costs a full gate run, sessions stop committing and accumulate a large uncommitted tree
— which is the exact risk the rule exists to prevent. A commit that a fast targeted check
passed is worth far more than a commit that never happened.

The reason the narrow check is still **required** is bisectability: a history where half the
commits are broken cannot be searched later, and that is precisely when you need it.

Commits touching **only** check-exempt paths (the project's documentation/spec directories)
need no run at all — the project already carves this out; follow its declaration rather than
inventing one.
