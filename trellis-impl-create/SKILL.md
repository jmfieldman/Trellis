---
name: trellis-impl-create
description: Create a new Implementation Plan [$0=source-design-plan] [$1=output-impl-path]
disable-model-invocation: true
---

# Design Plan → Implementation Plan — Bootstrap Skill

Skill prompt for an LLM agent. Activated when the user wants to take a finished (or substantially-resolved) **design plan** and produce the initial scaffolding of an **implementation plan** — i.e., Round 1 of the implementation plan's iteration cycle.

This skill stops at the end of Round 1. Subsequent rounds (fleshing out individual sprints, locking sprint-level decisions, decomposing sprints into steps) follow the round-by-round mechanics in [implementation-plan.md](./implementation-plan.md). After the scaffold lands, hand the conversation back to the user for Round 2.

---

## What this skill produces

A new directory at the user-supplied path, containing:

- `overview.md` — populated framing, philosophy, locked feature-wide decisions, sprint roster + dependency graph, initial Open Questions, and a `Round 1` Status entry.
- `progress.md` — master checklist scaffolded with one section per sprint (steps left blank or as best-effort placeholders).
- One stub file per sprint — `01-<topic>.md`, `02-<topic>.md`, … — each containing at minimum: Goal, Prerequisites, Deliverables (best-effort first cut), Out of scope, and an initial sprint-level Open Questions list.

The implementation plan is **not** complete after Round 1. Step-level detail, Architecture notes, API surfaces, locked sprint decisions, and Acceptance checklists land in subsequent rounds.

---

## Required inputs

The design plan is located here: $0

The implementation plan should be created in the directory: $1

Before starting, the agent must have:

1. **The design plan.** Read end to end. The agent must know its vocabulary, foundational decisions (numbered if Shape A; prose if Shape B), Scope / Non-Goals, Schema, Lifecycle sections, API surface, Cross-service contracts, Privacy/Auth, Open questions, and Decisions log.
2. **The output path** for the implementation plan directory. Create this directory if it doesn't exist.
3. **The implementation plan specification.** Read [implementation-plan.md](./implementation-plan.md) before producing any files — the document anatomy, file naming, and section ordering rules in that spec are mandatory.
4. **Project context.** Skim the project's `CLAUDE.md`, `AGENTS.md` (or equivalent) so the locked decisions in `overview.md` reflect actual project conventions (test framework, schema-edit authorization, migration policy, idempotency keys, logging, etc.). Don't restate the conventions section verbatim — pin the things every sprint will inherit and link to the project doc for the rest.
5. **(If they exist) one or more analogue implementation plans in the repo or adjacent repos.** Borrowing structure from a precedent is encouraged — naming conventions, locked-decisions tables, sprint-roster format. Cite the analogue in your Round 1 Status entry so the user can see the lineage.

If any of these are missing, **stop and ask the user** before producing files. Don't guess at the design plan's intent.

**Architecture is inherited, not prescribed.** This skill works for any project type — backend service, frontend app, CLI, mobile client, library, infrastructure module, data pipeline. The structural mechanics (sprint roster, dependency graph, locked decisions, sprint stubs, master progress checklist) hold in every case. The *content* of those structures must reflect the project's actual architecture, not a generic template. Whenever this skill cites a backend-service example (schemas, repositories, managers, resources, OpenAPI specs, transactions), translate the example to its equivalent in the target project. If there's no equivalent, drop the section.

**Additional Context**: The user may include additional instructions with the skill invocation. Those should be integrated into these instructions, and take precedent.

---

## The Round 1 workflow

### Step A — Read everything, then sketch slicing

1. Read the design plan end to end.
2. Read [implementation-plan.md](./implementation-plan.md) end to end. Pay particular attention to the "Architecture is inherited, not prescribed" section — it governs how examples in this skill are interpreted.
3. **Read the project's `CLAUDE.md` and any analogue implementation plans the user pointed to** (or that you find adjacent to the design plan). **Extract the project's architecture from these sources** before any planning:
   - The layering the codebase uses (e.g., schemas / repositories / managers / resources for a backend service; design tokens / atoms / molecules / organisms for a UI library; core engine / subcommand handlers / CLI surface for a command-line tool; reducer / selector / view for a state-machine app; module / submodule / fixture for an infra-as-code project).
   - The directory layout conventions, file-naming conventions, casing.
   - The test layering and what each layer covers.
   - The build / spec / generation workflow.
   - The observability and error-handling conventions.
   - The known-footgun set surfaced by prior plans, code-review threads, or post-mortems.

   The implementation plan you're about to scaffold inherits this architecture. **It is not your job to invent one.** If the project doesn't have an obvious convention for some axis (e.g., no public API surface; no schema), the implementation plan simply doesn't have that section.

4. **Sketch a sprint roster** privately (in scratch / planning context). Begin from the natural seam-lines the *feature* decomposes along *given this project's architecture*. The decomposition is project-specific:
   - Backend service: typically **schema → repositories → managers → resources / endpoints → cross-service contracts → workers / async → UX / surfaces → hardening**.
   - Frontend app: typically **design tokens / primitives → atoms → molecules → screens → flows / state → polish**.
   - CLI: typically **core engine → subcommand routing → individual commands → completion / docs → distribution**.
   - Infrastructure: typically **state / providers → core resources → composed modules → environment wiring → observability → runbooks**.
   - Library: typically **core API → adapters → integration helpers → published types / docs → release polish**.

   Adopt whatever sequence the project already implies. Deviations from that sequence should be deliberate and called out in `overview.md`'s sprint-order rationale. Aim for sprints that:
   - Each ship a coherent, independently-testable deliverable.
   - Each leave the codebase buildable, type-checking, and test-passing.
   - Each fit within roughly one to two weeks of focused work for a single engineer.
   - Form a directed graph where each sprint depends only on prior sprints (cycles are a slicing bug).
   - Are sequenced so foundational layers (schema, repositories) land before logic (managers), which lands before surfaces (resources, endpoints, UX).

   Don't over-optimize for equal sizes — sprints are equal in **shippability**, not in scope.

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

### Step C — Scaffold the directory

Once slicing is agreed, create the directory and the three required artifact types.

#### `overview.md`

Populate every section the implementation plan spec lists, scaled to what's actually known in Round 1:

- **Title & framing block** — link back to the design plan.
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
- **Decisions log** — start empty, or with `(R1)`-tagged entries that capture the slicing decisions agreed in Step B.
- **Status** — one entry: `**Round 1**: scaffolding + sprint roster + open questions enumerated.`
- **What this plan does *not* try to do** — short list at overview level, distinct from per-sprint Out of scope.
- **How to read each sprint** — the standard one-paragraph blurb pointing at the sprint anatomy.

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

Before sending the user the hand-off message, walk the directory you just produced and verify:

- **Each sprint ships something observable.** No sprint's Deliverables list is empty or aspirational ("we'll figure it out").
- **Out-of-scope is forward-linked.** Every "deferred to Sprint NN" entry corresponds to an actual sprint NN whose Deliverables list owns the item.
- **Prerequisites have a home.** Every prerequisite a sprint names corresponds to a deliverable in a prior sprint, a peer-service surface that exists, or an audit-or-build branch flagged in the stub.
- **The dependency chart is honest.** No cycles. No edges that aren't justified in the sprint-order rationale.
- **Cross-references resolve.** Every `Sprint NN` reference points at an actual file; no dangling links between stubs.
- **Every "we'll do X in Sprint NN" matches an entry** in Sprint NN's Deliverables list. Promises and obligations are paired.

Fix the issues you find before handing back to the user. A scaffolded plan with broken cross-references is harder to repair in Round 2 than to get right now.

### Step D — Hand back to the user

After all files are written, summarize what you produced (overview + N stubs + progress) and propose what Round 2 should focus on. The recommended Round 2 default is "flesh out Sprint 01 to execution-ready" — the foundational sprint usually has the highest density of unresolved sprint-level decisions, and locking it informs later sprints.

Stop. Don't auto-progress to Round 2.

---

## Sizing heuristics for sprint slicing

Use these as guard-rails when sketching the roster. Heuristic, not strict.

**Typical shape.** 5–12 sprints per feature; 5–10 steps per sprint. A step is roughly half a day of focused work; a sprint is roughly one to two weeks. Real plans drift from these numbers and that's fine — they're calibration anchors, not targets. If the roster comes out at 3 sprints, the slicing is probably too coarse; if it comes out at 20, the slicing is probably too fine (or some sprints should be folded).

**Common sprint archetypes.** Most sprints fall into one of a handful of shapes; recognizing which archetype you're slicing helps name the deliverables and the test surface. The archetypes below are stack-agnostic; the *content* of each in your project depends on the architecture you extracted in Step A:

- **Foundation sprints** — establish primitives the rest of the plan builds on. (Backend: schema + repositories. Frontend: design tokens + base components. CLI: argument parser + core engine. Infra: providers + state + base modules.) No external surface yet; verification is invariant tests.
- **Logic-layer sprints** — non-trivial business rules built on the foundation, with no public surface yet. (Backend: managers. Frontend: hooks / state machines. CLI: command-engine logic.) Fold into the foundation sprint when the layer is thin.
- **Capability sprints** — ship one user-or-system-visible capability end-to-end. (Backend: an endpoint. Frontend: a screen / flow. CLI: a subcommand. Library: a public function or class.) Built on prior foundation and logic. Usually one capability per sprint, sometimes a tightly-coupled pair (e.g., provision + complete).
- **Integration / contract sprints** — wire one component into another (post-commit fan-out into a peer service; a screen into the routing graph; a CLI command into shell completion; a Terraform module into an environment wiring). The risk surface is the seam; integration tests dominate.
- **Helper / predicate sprints** — small, narrowly-scoped landing of a shared utility. Often a single helper + a focused test matrix.
- **Async / background sprints** — workers, jobs, scheduled tasks, queues. One worker per sprint when they're large.
- **Surface / polish sprints** — visible product surface or final ergonomics. Usually late, after the substrate stabilizes.
- **Hardening sprints** — quotas, rate limits, observability, runbook docs, accessibility audits, performance budgets, spec / docs reconciliation. Always last; don't fold into earlier sprints.

**Slicing heuristics** (illustrated with backend examples; translate to your stack):

- **Foundation work** is usually one sprint, sometimes two. If the substrate spans many independent primitives with non-trivial constraints, splitting may be warranted (e.g., one for entity tables, one for engagement / junction tables).
- **Logic-layer work** is typically a sprint of its own when there are non-trivial rules (e.g., supersession, soft-delete cascade, idempotency, multi-step orchestration). When the logic layer is a thin pass-through, fold it into the foundation sprint.
- **Capability work** is sliced by capability seam, not by individual route / function. Group capabilities that share an underlying component or a transactional shape; split when authorization / permission stories diverge meaningfully.
- **Async / background work** is usually its own sprint, sometimes one per worker if they're large.
- **Integration / cross-component work** is its own sprint when it involves more than one peer or when the contract is novel.
- **Surface / UX work** is typically late, after the data model and reads stabilize.
- **Hardening** is the last sprint. Don't fold into earlier sprints — it's load-bearing as a deliberate last-pass.

If a sprint's Deliverables list runs to ~15+ artifacts, it's probably too big — re-slice. If a sprint has ≤ 2 Deliverables and shares a clear seam with a neighbor, it's probably too small — fold.

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
- **Don't impose a backend-shaped layering on a non-backend project.** If the project is a UI library, a CLI, or an infrastructure module, the implementation plan's sprints, sections, and conventions reflect *that* project's architecture. Don't invent a "manager layer" or a "schema sprint" because the precedent plans had one.
- **Don't skip Step B.** Writing files before confirming the slicing with the user means re-shaping every file when slicing changes. The 5-minute confirmation is cheap insurance.
- **Don't create the `00-overview.md` legacy filename.** The canonical name is `overview.md`. (Older plans in the repo may still use `00-overview.md`; new plans don't.)
- **Don't paraphrase the design plan into one giant sprint.** Re-derive sprint structure from outcomes (what ships when), not from the design plan's section list. A roster that mirrors the design plan's chapter headings isn't slicing — it's re-titling, and it loses the entire point of the implementation plan.
- **Don't write "implementation details TBD"** anywhere in a stub. That phrase is a planning failure dressed up as humility. Either lock the call, surface it as an Open Question with rationale, or name the future round / sprint that closes it.

---

## Hand-off message template

After the scaffold is written, message the user along these lines:

> Round 1 scaffold landed at `<path>`:
> - `overview.md` (philosophy, sprint roster, dependency graph, feature-wide locked decisions, open questions)
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
