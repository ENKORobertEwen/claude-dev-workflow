# Changelog — dev plugin

## 2.24.0 — ui-design-fix-session: abgestufte Verifikation fuer Commits waehrend der Session

**Problem fixed:** Der Command forderte Zwischen-Commits, liess aber offen, welches Gate dafuer
gilt — also galt implizit das volle Projekt-Gate. Ein voller Lauf je Commit ist teuer genug,
dass Sessions das Committen aufschieben und einen grossen ungepruefen Arbeitsbaum aufstauen.
Genau das Risiko, das die Regel verhindern soll. Beobachtet in einer Session, die acht fertige,
je einzeln gruene Aufgaben zu einem einzigen uncommitteten Klumpen anwachsen liess.

**Changes:**
- Neuer Abschnitt „The verification gate for this mode" mit drei Stufen: waehrend der Iteration
  nichts, beim **lokalen Commit** das schmale Ziel des beruehrten Bereichs plus die Guards, die
  es nicht mitlaeuft, beim **Push/PR/Session-Ende** das volle Gate.
- Die Lockerung gilt **ausdruecklich nur in diesem Modus** und aendert die Projektregel nicht.
- Begruendung mit dokumentiert: das schmale Ziel bleibt Pflicht, weil eine Historie aus halb
  kaputten Commits nicht mehr bisect-bar ist — und genau dann braucht man sie.
- Hinweis auf die bereits bestehende Ausnahme fuer reine Dokumentations-/Spec-Pfade.

## 2.23.0 — ui-design-fix-session: Orchestrator bleibt beim Nutzer, ein langlebiger Umsetzungs-Agent

**Problem fixed:** Der Hauptagent steckte waehrend Umsetzung und Verifikation fest; neue
Meldungen des Nutzers landeten mitten in laufenden Werkzeugaufrufen. Ausserdem war offen, wer
was ausfuehrt.

**Changes:**
- Rollentrennung: der Hauptagent **orchestriert und bleibt beim Nutzer**; die Umsetzung samt
  Verifikation laeuft in einem Sub-Agenten.
- Work-Item-Aufrufe macht der Hauptagent **inline** — bewusst *kein* eigener DevOps-Agent: ein
  Statuswechsel ist ein schneller Aufruf, das Board ist die Aufzeichnung der Sitzung und darf
  nicht von einem zusaetzlichen Fehlerpfad abhaengen.
- **Ein langlebiger** Umsetzungs-Agent statt eines frischen je Task, mit ausdruecklich als
  solcher markierter Abweichung von `/dev:implement`: hier ist der angesammelte Kontext der
  Zweck, nicht das Risiko. Neustart-Regel, falls er driftet.
- **Warteschlange:** Meldungen waehrend laufender Arbeit werden **sofort als Task angelegt**
  (Sichtbarkeit auf dem Board), aber erst nach Abschluss des offenen Punktes beauftragt. Nie
  zwei Umsetzungs-Agenten gleichzeitig — gemeinsamer Arbeitsbaum, ein Dev-Server.
- **Belegpflicht:** der Agent liefert geaenderte Dateien, Pruefausgaben und den letzten
  Build-Status zurueck; der Hauptagent prueft stichprobenartig nach, statt nur weiterzureichen.
- Kontextpaket beim Beauftragen, damit der Agent Ermitteltes nicht neu herleitet.

## 2.22.1 — /dev:ui-design-fix-session: Eltern-Work-Item wird aufgeloest statt angenommen

**Problem fixed:** Das Command setzte voraus, dass die uebergebene ID bereits ein taugliches
Elternteil ist. Bei einem Feature waeren die Tasks direkt darunter gelandet — zu grob, und die
Sitzung haette keinen eigenen Traeger gehabt.

**Changes:**
- Der Typ des uebergebenen Work Items wird **geladen und geprueft**, nicht aus der Nummer
  geschlossen: PBI/Bug werden direkt verwendet; bei einem **Feature** wird zuerst ein passendes
  **Product Backlog Item** darunter angelegt (Titel/Beschreibung vorschlagen, Freigabe abwarten)
  und dieses als Elternteil genutzt; andere Typen werden mit Begruendung zurueckgewiesen.
- Ohne ID wird nach dem Work Item gefragt.
- Das ermittelte Elternteil wird vor der ersten Meldung ausdruecklich genannt.

## 2.22.0 — Neues Command /dev:ui-design-fix-session

**Problem fixed:** Interaktive Design-Feinschliff-Runden liefen bisher formlos im Chat: der
Nutzer meldete Abweichungen, die Umsetzung wurde in Prosa zurückgemeldet, und offene Punkte
gingen zwischen den Meldungen verloren. Es gab keine Spur davon, was gemeldet, umgesetzt oder
zurückgestellt wurde — und keine Live-Sicht für den Nutzer.

**Changes:**
- Neues `commands/ui-design-fix-session.md`: Sitzung gegen ein Eltern-Work-Item (PBI/Bug);
  jede gemeldete Abweichung wird **ein Task-Work-Item** darunter, mit Zustandsführung
  `To Do → In Progress → Done` in Echtzeit, damit der Fortschritt auf dem Board sichtbar ist
  statt im Chat.
- Setup-Gate: erst Eltern-Work-Item und Styling-Kontrakt lesen, dann **nach Go des Nutzers**
  die App lokal starten — die Prüfschleife des Nutzers ist der Kern des Modus.
- Ebenen-Disziplin verbindlich: Ursache statt Symptom, Werte in die Token-Ebene, Bedienelemente
  in gemeinsame Primitives. Ausdrückliche Regel gegen doppeltes Stylen desselben Controls auf
  mehreren Seiten.
- Verifikationspflicht vor jeder Fertigmeldung, inkl. Hinweis, dass Dev-Server-Logs auch alte
  Fehler enthalten — der **letzte** Build muss sauber sein.
- Festes dreiteiliges Antwortformat (umgesetzt / nicht umgesetzt / braucht Entscheidung).
- Parkbahn für Nicht-Live-Arbeit (v. a. Backend-Änderungen mit Neustart): eigener Task,
  gekennzeichnet, bleibt in `To Do` und wird nicht nebenbei erledigt.
- Harte Regeln: kein Browser-Automatismus während der Sitzung (nur auf Wunsch als letzte
  Instanz am Ende), strikt sequenziell, keine parallelen Sub-Agents auf demselben Arbeitsbaum.
- Zwischencommits statt eines großen ungeprüften Arbeitsbaums am Ende.

## 2.21.0 — Neues Command /dev:feature (fachliches Refinement → Work Items)

**Problem fixed:** Between "idea" and `/dev:plan` there was no command: business-level
refinement (scope, rules, grey zones, gates) happened ad hoc in chat, and work items were
created by hand — with varying description/criteria quality. `/dev:plan` then had to
re-ask business questions.

**Changes:**
- New `commands/feature.md`: develops a feature idea (or refines an existing Feature work
  item) on the business level — one question at a time, evidence-based codebase scouting,
  project gates from CLAUDE.md — and stores the result as work items: Feature + PBIs as
  user stories, descriptions and acceptance criteria in the proper fields
  (`Microsoft.VSTS.Common.AcceptanceCriteria` exists on Feature and PBI in Scrum).
  Approval gate before anything is written; read-back verification (hierarchy, iteration,
  encoding) after. Definition of done: `/dev:plan` needs no further business decisions.
- Conventions come from the project's tracker doc (default
  `product/docs/azure-devops-structure.md`); if missing, the command asks org/project and
  seeds the doc. States stay `New` — `Approved`/plan tags remain `/dev:plan`'s job.
- Model pinning like plan.md: Fable 5 (frontmatter + model guard).

## 2.20.0 — Infrastruktur-Dokumentation als Methodik-Bestandteil

**Problem fixed:** Bootstrap scaffolds DDD/ADR/plans but no home for
infrastructure/operations knowledge (environments, hosting, CI/CD, external
services, secret locations). Real projects then invent unstructured places for
it (CustomerWallet: ad-hoc product/docs/) that no command maintains.

**Changes:**
- bootstrap.md: new conversational step 5 (deployment target/environments —
  "still open" is a valid answer), new `product/infra/infrastructure.md`
  starter (sections: Umgebungen, Hosting, CI/CD, Externe Dienste,
  Secrets-Fundorte — never values, Monitoring, Runbooks), CLAUDE.md template
  gets the infra tree, a "Know the Infrastructure" read step and a pre-change
  question; steps renumbered (gap at 5 closed, generation steps 7–11).
- plan.md: new step 5a "Identify Infrastructure Implications" (env/config,
  pipelines, external services, secrets, runbooks → plan phase updating
  product/infra/; existing projects use their documented equivalent), infra
  phase template in the plan format, Reviewer 3 checks it.

## 2.19.0 — Frische Sub-Agents + deterministisches Kontext-Paket + Modell-Pinning

**Problem fixed (1/2 — gekoppelt):** Orchestrators sometimes CONTINUED sub-agents
across phases/cycles (SendMessage) instead of spawning fresh ones — context fills
up, quality degrades, earlier-phase assumptions leak forward, the UI reviewer
stops being blind. But reuse silently compensated a second defect: the hand-off
was "paste ... any relevant context" — pure judgment. Enforcing fresh agents
without fixing the hand-off would regress.

**Changes:**
- implement.md: "Sub-agent lifecycle: fresh, always" — every launch is a NEW
  agent, no continuation ever (not even follow-up questions); artifacts are the
  only interface. Clarified: the running APP may be reused between frontend
  phases, the reviewer agent may not.
- implement.md: deterministic hand-off — non-frontend sub-agents read the plan
  file themselves (only Phase N); frontend sub-agents get an assembled, filtered
  packet (phase block + UI/UX Spec + decisions + registers + explicitly
  referenced sections, chains resolved). Asymmetry rationale documented.
- plan.md: phase-isolation rule — cross-block dependencies only via explicit
  named references (implement resolves exactly these); Reviewer 4 checks it.
- Model pinning: plan.md runs on Fable 5 (frontmatter + model guard; reviewers
  model 'fable'); implement.md frontmatter claude-opus-4-8; implementation/fix/
  UI-review sub-agents 'opus', verification 'haiku'; graceful fallback when a
  model is unavailable.

## 2.18.0 — /dev:implement: sichtbare Taskliste erzwingen

**Problem fixed:** Step 1 said only "Create a task list with all phases" — vague
enough that runs sometimes produced the visible Claude Code task list (task
tools) and sometimes only prose. Phase progress was then invisible.

**Change:** Step 1 now mandates creating the task list via the harness task
tools (TaskCreate / TodoWrite fallback) BEFORE any other work — one task per
phase (`Phase N: <title>`), exactly one `in_progress` at a time, `completed`
immediately after each phase commit, extra tasks for PR/done-move/work-item
wiring. A markdown checklist explicitly does not count. Resume path (step 2)
additionally syncs the list against already-committed phases.

## 2.17.0 — Check-exempt commits: skip provably redundant `./do check` runs

**Problem fixed:** `./do check` is expensive (time), and the verification rule
("must pass before ANY commit, no exceptions") forced a full run even for
commits that never touch a check input — plan documents, ADRs, DDD docs, design
snapshots, the design ledger. Those runs are pure waste: the check is a
function of its inputs, and if no changed file feeds the pipeline, the result
cannot have changed.

**Design constraint:** an exemption must not become an escape hatch (the exact
failure mode 2.16.0 closed for design fidelity). It is therefore defined by
mechanically checkable facts only — path membership of the actual changed
files — never by judgment ("small", "trivial", "doesn't touch code").

**Changes:**
- **`verification-required` skill, new rule 0:** a commit is check-exempt iff
  EVERY staged file (`git diff --cached --name-only`) matches a check-exempt
  glob. Plugin default: `product/**`. Project additions: globs declared in the
  project CLAUDE.md under `### Check-Exempt Paths` (additive only). Fail-closed
  — one non-matching file, a missing declaration, or any doubt → full check.
  Machine-readable files in an exempt commit (`.json`/`.yaml`) must still parse
  (`jq empty`). Exempt commits are announced explicitly ("check-exempt: …").
- **`/dev:implement`:** the orchestrator decides per phase on the actual
  `git status` file list (never on what the phase text claims) and skips the
  verification sub-agent for exempt phases — typically DDD, ADR, and plan-only
  phases. New hardcoded operational decision; the NEVER-rule "skips
  verification" now names the mechanical exemption as its only carve-out.
- **`/dev:plan` step 14:** the plan commit is explicitly covered by the
  exemption (`product/**`) instead of silently ignoring the verification rule.
- **CLAUDE.md template + `/dev:bootstrap`:** the Verification Rule section
  documents the exemption and a new `### Check-Exempt Paths` section lets each
  project declare additional globs; bootstrap fills `(none beyond the default)`
  for fresh projects.

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
