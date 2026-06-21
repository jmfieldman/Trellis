---
name: impl-create
description: Create a new Implementation Plan
argument-hint: <root>
disable-model-invocation: true
---

# Design Plan → Implementation Plan — Bootstrap Skill

Skill prompt for an LLM agent. Activated when the user wants to take a finished (or substantially-resolved) **design plan** and produce the initial scaffolding of an **implementation plan** — i.e., Round 1 of the implementation plan's iteration cycle.

This skill stops at the end of Round 1. Subsequent rounds (fleshing out individual sprints, locking sprint-level decisions, decomposing sprints into steps) follow the round-by-round mechanics in [implementation-plan.md](../specs/implementation-plan.md). After the scaffold lands, hand the conversation back to the user for Round 2.

---

## What this skill produces

Implementation-plan files directly in `<root>`, alongside `design.md`:

- `overview.md` — populated framing, philosophy, locked feature-wide decisions, sprint roster + dependency graph, initial Open Questions. **No Decisions log or Status section inside `overview.md`** — those live directly in the feature root as `decisions.md` and `status.md`.
- `decisions.md` — the plan-level (cross-cutting) Decisions log. Starts empty or with `(R1)`-tagged entries capturing the slicing decisions agreed in Round 1.
- `status.md` — the plan-level round-by-round audit trail. Starts with a single `**Round 1**: scaffolding + sprint roster + open questions enumerated. _Next:_ …` entry.
- `progress.md` — master checklist scaffolded with one section per sprint (steps left blank or as best-effort placeholders).
- One stub file per sprint — `01-<topic>.md`, `02-<topic>.md`, … — each containing at minimum: Goal, Prerequisites, Deliverables (best-effort first cut), Out of scope, and an initial sprint-level Open Questions list.

**Do not create `post-mortem.md` at Round 1.** That file is created lazily — the first sprint to ship creates it in the same PR that checks off its final step. Scaffolding an empty `post-mortem.md` in Round 1 is noise. See `implementation-plan.md` § "`post-mortem.md` anatomy" for the format.

The implementation plan is **not** complete after Round 1. Step-level detail, Architecture notes, API surfaces, locked sprint decisions, and Acceptance checklists land in subsequent rounds.

---

## Required inputs

One path is needed, extracted from the user's natural-language invocation:

- `<root>` — the feature root directory. The agent consumes the design plan at `<root>/design.md`. If the user gave a path ending in `design.md`, treat its parent directory as `<root>`.

If `<root>` is missing, ask for it and stop. If `<root>/design.md` does not exist, stop and report the missing file.

Before starting, the agent must have:

1. **The design plan.** Read end to end. The agent must know its vocabulary, foundational decisions (numbered if Shape A; prose if Shape B), Scope / Non-Goals, Schema, Lifecycle sections, API surface, Cross-service contracts, Privacy/Auth, Open questions, and Decisions log.
2. **The output path** for the implementation plan files. Create `<root>` if it doesn't exist.
3. **The implementation plan specification.** Read [implementation-plan.md](../specs/implementation-plan.md) before producing any files — the document anatomy, file naming, and section ordering rules in that spec are mandatory.
4. **Project context.** Skim the project's `CLAUDE.md`, `AGENTS.md` (or equivalent) so the locked decisions in `overview.md` reflect actual project conventions (test framework, schema-edit authorization, migration policy, idempotency keys, logging, etc.). Don't restate the conventions section verbatim — pin the things every sprint will inherit and link to the project doc for the rest.
5. **(If they exist) one or more analogue implementation plans in the repo or adjacent repos.** Borrowing structure from a precedent is encouraged — naming conventions, locked-decisions tables, sprint-roster format. Cite the analogue in your Round 1 entry in `status.md` so the user can see the lineage.

If any of these are missing, **stop and ask the user** before producing files. Don't guess at the design plan's intent.

**Architecture is inherited, not prescribed.** The full rule lives in [implementation-plan.md § "Architecture is inherited, not prescribed"](../specs/implementation-plan.md#architecture-is-inherited-not-prescribed) — read it before scaffolding. In brief: the structural mechanics (sprint roster, dependency graph, locked decisions, sprint stubs, master progress checklist) hold for any project type, but their *content* must reflect the target project's actual architecture. Translate every backend-service example (schemas, repositories, managers, resources, OpenAPI specs, transactions) to its equivalent in the target stack; where there's no equivalent, drop the section.

**Additional Context**: The user may include additional instructions with the skill invocation. Those should be integrated into these instructions, and take precedent.

---

## The Round 1 workflow

### Step A — Read everything, then sketch slicing

1. Read the design plan end to end.
2. Read [implementation-plan.md](../specs/implementation-plan.md) end to end. Pay particular attention to the "Architecture is inherited, not prescribed" section — it governs how examples in this skill are interpreted.
3. **Read the project's `CLAUDE.md` and any analogue implementation plans the user pointed to** (or that you find adjacent to the design plan). Extract the project's architecture per the checklist in [implementation-plan.md § "Architecture is inherited, not prescribed"](../specs/implementation-plan.md#architecture-is-inherited-not-prescribed) — layering, directory conventions, test layering, build/spec workflow, observability, known-footgun set. The implementation plan you're about to scaffold inherits this architecture; it is not your job to invent one.

4. **Sketch a sprint roster** privately (in scratch / planning context). Begin from the natural seam-lines the *feature* decomposes along *given this project's architecture*. Adopt whatever sequence the project already implies — typically foundation → logic → capability → integration → async → surface → hardening, but the exact shape is stack-specific. Deviations should be deliberate and called out in `overview.md`'s sprint-order rationale.

   See [implementation-plan.md § "Sizing heuristics for sprint slicing"](../specs/implementation-plan.md#sizing-heuristics-for-sprint-slicing) for archetypes (Foundation / Logic-layer / Capability / Integration / Helper / Async / Surface / Hardening), the slicing heuristics (when to split foundation, when to fold a thin logic layer, etc.), and the calibration anchors (5–12 sprints, 5–10 steps/sprint, ≤2 deliverables → fold, 15+ → re-slice). The criteria each sprint must satisfy (coherent, buildable, ~1–2 weeks, DAG, foundational-before-surface) live in that same guide under "Rationale for sprint-sized slices."

5. **Identify the cross-cutting decisions** that should live in `overview.md`'s "Feature-wide locked decisions" table. These are the choices every sprint would otherwise re-decide. The exact list depends on the project — pull them from the design plan's foundational decisions plus the project's conventions. Examples from a backend-service shape (translate to your stack): schema location, default-view filters, soft-delete posture, worker cadence, idempotency-key shape, banned patterns, cross-schema FK rule. Examples from a frontend project: design-token sources, store / cache library, routing strategy, accessibility-test thresholds, banned UI primitives. The principle is the same: pin the choices that, left unpinned, would be re-litigated in every sprint.

6. **Identify what's truly out of scope across the whole plan** (vs. out of scope for a specific sprint). Aggregate into the "Out-of-scope across all sprints" list so individual sprints can stay tight.

### Step B — Confirm slicing with the user before writing files

Don't write any files yet. Surface your sprint roster as a numbered list with one-line outcomes:

```
01. Schema, types, repositories — the four tables land with all indexes …
02. Manager core: groups, supersession, soft-delete cascade — manager …
03. Resource layer + lab-order ingestion — first end-to-end ingestion …
…
```

Plus a sketch of the dependency graph.

Ask the user: does this slicing match their mental model? Are there sprints they would split, fold, or reorder? Are there foundational decisions they want pinned at the overview level that you missed?

**Iterate on the slicing in conversation before writing files.** A bad slicing means re-shaping every file in Round 2; getting it close in Round 1 saves churn.

### Step C — Scaffold files directly in `<root>`

Once slicing is agreed, create the five required artifact types directly in `<root>` (`overview.md`, `decisions.md`, `status.md`, `progress.md`, one stub per sprint). See [implementation-plan.md § "Root layout"](../specs/implementation-plan.md) for the canonical file list.

#### `overview.md`

Populate every section the implementation plan spec lists, scaled to what's actually known in Round 1:

- **Title & framing block** — link back to the design plan at `./design.md`.
- **Philosophy** — adapt the design plan's tone. Common subsections: "The plan is the source of truth," "Tier-0-aware but never tier-0-locked" (or the project's equivalent), "Cross-service contract over internal cleverness," "Test the cross-service seams hardest," "Small, shippable sprints." Crib language from analogue plans where appropriate; tighten to this feature's specifics.
- **Architectural invariants the sprints uphold** — restate from the design plan's foundational decisions and the project's `CLAUDE.md`. Be exhaustive about invariants every sprint must verify when touched. The list is project-specific (backend examples: no cross-schema FKs, no peer calls inside DB transactions, append-only constraints. Frontend examples: no direct DOM manipulation outside the renderer, no `any` in public component props. Inherit, don't invent).
- **Module / directory layout impact** — a code-block tree showing what the codebase looks like at the end of the plan. Note that this is forward-looking; subsequent rounds will tighten file-level detail per sprint.
- **Cross-sprint conventions** — pin one canonical answer per axis the project actually has. Drop axes that don't apply; add axes the project enforces but `impl-plan.md`'s example list doesn't enumerate. (Backend-service example axes: branching, migrations, seed data, OpenAPI sync, tests, logging, errors, idempotency, feature flags. Adapt.)
- **Sprint organization** — Rationale for the sprint order (one paragraph per sprint), Rationale for sprint-sized slices (the manifesto).
- **Sprint roster** — table with columns `# | Sprint (link) | Outcome`. Every link points at a stub file you'll create in this same step.
- **Dependency graph** — ASCII chart + one-liners explaining each edge.
- **Feature-wide locked decisions** — the table you sketched in Step A.
- **Out-of-scope across all sprints** — the list you assembled in Step A.
- **Open questions** — anything from the design plan still marked open that affects sprint shape, plus any new questions surfaced by the slicing discussion in Step B.
- **What this plan does *not* try to do** — short list at overview level, distinct from per-sprint Out of scope.
- **How to read each sprint** — the standard one-paragraph blurb pointing at the sprint anatomy. Mention that cross-cutting decisions live in `decisions.md` and the round-by-round audit trail lives in `status.md`.

> **Do not add a Decisions log or Status section inside `overview.md`.** Those live directly in the feature root as `decisions.md` and `status.md` — see below.

#### `decisions.md`

Create the file. Body starts with the standard framing block (`> Part of the [<Feature> Implementation Plan](./overview.md). Cross-cutting decisions only…`) and either:

- An **empty bullet list** (most common in Round 1), or
- One or more `(R1)`-tagged bullets capturing the cross-cutting slicing decisions agreed in Step B.

See [implementation-plan.md § "`decisions.md` anatomy"](../specs/implementation-plan.md) for the format. Don't fabricate decisions just to populate the file — empty + framing block is a perfectly valid Round 1 state.

#### `status.md`

Create the file. Body starts with the standard framing block (`> Part of the [<Feature> Implementation Plan](./overview.md). Plan-level round-by-round audit trail…`) and exactly one entry:

```
- **Round 1**: scaffolding + sprint roster + open questions enumerated. _Next:_ <one-line recommended Round 2 focus — usually "lock Sprint 01 to execution-ready" or, if a particular Open question gates Sprint 01, the question ID + tag + scope>.
```

The `_Next:_` clause persists the chat hand-off recommendation into the doc itself so a user resuming via `impl-iterate` recovers it without depending on chat history. See [implementation-plan.md § "`status.md` anatomy"](../specs/implementation-plan.md) for the format.

#### Stub sprint files (`NN-<topic>.md`)

Round 1 stubs are deliberately incomplete. Each contains:

- **Title & overview pointer.**
- **Goal** — a single sentence.
- **Prerequisites** — prior sprints + peer-service surfaces (audit-or-build branches noted where you don't yet know whether the helper exists).
- **Deliverables at end of sprint** — best-effort first cut from the design plan's relevant section. It's fine if this list is incomplete; subsequent rounds will tighten.
- **Out of scope** — explicit forward-links to the sprint(s) that own each deferred item.
- **Open questions** — sprint-scoped questions that need answers before this sprint can be locked. Include both questions inherited from the design plan and questions surfaced by the slicing.

Do **not** populate in Round 1:

- **Key design decisions, locked before implementation** — leave the section header and a placeholder line `_To be locked in a later round._`. Don't fabricate decisions.
- **Architecture notes** — same posture; placeholder.
- **API surface** — placeholder.
- **Progress** — leave this section out of the stub. It will be added when the sprint is decomposed into steps. (`progress.md` will list this sprint by name only, no step bullets yet.)
- **Implementation Steps** — leave out entirely.
- **Step Dependency Chart** — leave out.
- **Acceptance checklist** — leave out.

A Round 1 stub is typically ~30–80 lines. If a stub is approaching the length of a finished sprint file, you've over-decided. Pull back.

#### `progress.md`

Initial scaffold:

```markdown
# <Feature> — Progress

> Part of the [<Feature> Implementation Plan](./overview.md).

## Sprint 01 — <title>

_Steps will be enumerated when this sprint is decomposed._

## Sprint 02 — <title>

_Steps will be enumerated when this sprint is decomposed._

…
```

Subsequent rounds replace each placeholder with the actual step checklist as that sprint gets fleshed out.

### Step C.5 — Sanity-check / cross-link before hand-off

Before sending the user the hand-off message, walk the implementation-plan files you just produced directly in `<root>` and verify:

- **Each sprint ships something observable.** No sprint's Deliverables list is empty or aspirational ("we'll figure it out").
- **Out-of-scope is forward-linked.** Every "deferred to Sprint NN" entry corresponds to an actual sprint NN whose Deliverables list owns the item.
- **Prerequisites have a home.** Every prerequisite a sprint names corresponds to a deliverable in a prior sprint, a peer-service surface that exists, or an audit-or-build branch flagged in the stub.
- **The dependency chart is honest.** No cycles. No edges that aren't justified in the sprint-order rationale.
- **Cross-references resolve.** Every `Sprint NN` reference points at an actual file; no dangling links between stubs.
- **Every "we'll do X in Sprint NN" matches an entry** in Sprint NN's Deliverables list. Promises and obligations are paired.

Fix the issues you find before handing back to the user. A scaffolded plan with broken cross-references is harder to repair in Round 2 than to get right now.

### Step D — Hand back to the user

After all files are written, summarize what you produced (overview + decisions + status + progress + N stubs) and propose what Round 2 should focus on. The recommended Round 2 default is "flesh out Sprint 01 to execution-ready" — the foundational sprint usually has the highest density of unresolved sprint-level decisions, and locking it informs later sprints.

Stop. Don't auto-progress to Round 2.

---

## Quality bar for Round 1

When the scaffold is done, the user should be able to:

- Read `overview.md` end to end and have a clear mental model of the build sequence (≤ 15 minutes).
- Skim the sprint stubs and feel confident that no major piece of the design plan has been silently dropped.
- See the Open Questions list and recognize them as the real load-bearing decisions left to make.
- Open `progress.md` and see one section per sprint, all unchecked.

What's explicitly *not* expected after Round 1:

- That any sprint is execution-ready.
- That step-level work is decomposed.
- That every locked decision is captured.
- That the sprint roster won't shift in Round 2 — re-slicing is normal once sprint-level work begins.

---

## Anti-patterns specific to bootstrap

- **Don't fabricate sprint-level locked decisions.** If the design plan didn't decide it, leave it as an Open Question for the relevant sprint. The agent's Round 1 job is to surface, not to decide.
- **Don't pre-write Implementation Steps.** Round 1 stubs do not have steps. Steps are work for the round that actually fleshes out a given sprint.
- **Don't copy the design plan into `overview.md`.** Link, don't recapitulate. The Architectural invariants section restates a few load-bearing rules; everything else stays in the design plan.
- **Don't merge the design plan's Decisions log into the implementation plan's.** The design plan logs what the system is; the implementation plan logs how and in what order it gets built. Cross-reference, don't merge.
- **Don't propose a sprint sequence without surfacing alternatives** when the slicing has competing reasonable shapes. A note in Open Questions: "Sprints 06/07 could be merged or split — leaning split per analogue X; revisit in Round 2 once Sprint 06 sketches concretely."
- **Don't impose a backend-shaped layering on a non-backend project.** Inherit the target project's architecture per [implementation-plan.md § "Architecture is inherited, not prescribed"](../specs/implementation-plan.md#architecture-is-inherited-not-prescribed) — don't invent a "manager layer" or a "schema sprint" because the precedent plans had one.
- **Don't skip Step B.** Writing files before confirming the slicing with the user means re-shaping every file when slicing changes. The 5-minute confirmation is cheap insurance.
- **Only sprint files take the `NN-` numeric prefix.** The overview file is exactly `overview.md` — no number, no `0`, no prefix of any kind. Same for `decisions.md`, `status.md`, `progress.md`, and (later) `post-mortem.md`. The `01-`, `02-`, … `NN-` pattern applies *only* to sprint files.
- **Don't paraphrase the design plan into one giant sprint.** Re-derive sprint structure from outcomes (what ships when), not from the design plan's section list. A roster that mirrors the design plan's chapter headings isn't slicing — it's re-titling, and it loses the entire point of the implementation plan.
- **Don't write "implementation details TBD"** anywhere in a stub. That phrase is a planning failure dressed up as humility. Either lock the call, surface it as an Open Question with rationale, or name the future round / sprint that closes it.
- **Don't pre-create `post-mortem.md`.** It's created lazily when the first sprint ships, not at scaffold time.

---

## Hand-off message template

After the scaffold is written, message the user along these lines:

> Round 1 scaffold landed in `<root>` alongside `design.md`:
> - `overview.md` (philosophy, sprint roster, dependency graph, feature-wide locked decisions, open questions — no Decisions log or Status section)
> - `decisions.md` (plan-level Decisions log — empty / R1 entries only)
> - `status.md` (plan-level Status log — Round 1 entry)
> - `progress.md` (master checklist; per-sprint step lists empty until each sprint is decomposed)
> - `01-<topic>.md` … `NN-<topic>.md` (Round 1 stubs — Goal, Prerequisites, Deliverables, Out of scope, Open questions)
>
> The major open questions surfaced are:
> 1. <question> (affects Sprints <N>, <M>)
> 2. <question> (affects Sprint <N>)
> …
>
> Recommended Round 2 focus: flesh out Sprint 01 to execution-ready. Want to start there, or pick a different focus?

Explain that per-sprint steps are not created until the broader sprint open questions are answered to enough degree that you are confident in your ability to generate accurate sub-steps.

Stop and wait for the user's direction.

---

## Additional Instructions

The **Trellis instruction precedence chain** (`~/.trellis/instructions.md` → `<repo-root>/.trellis/instructions.md` → instructions supplied with this invocation) is applied by `SKILL.md` before this file is dispatched, so it is already in effect; honor it. Canonical statement — the single source shared by every instruction file and subagent brief: [instruction-precedence.md](../specs/instruction-precedence.md).
