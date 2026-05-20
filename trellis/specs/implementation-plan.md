# Implementation Plan Documents — Authoring Guide

Instructions for an LLM agent on producing **implementation plan documents**. This guide is the agent-facing brief for collaborating with a human user to produce one of these document sets from scratch — or to drive an existing one forward through another planning round.

The companion document, [design-plan.md](./design-plan.md), describes how upstream **design plans** are produced. An implementation plan is the next layer down: it consumes a finished (or in-progress) design plan and turns its decisions into concrete, sequenced engineering work.

---

## What these documents are

An **implementation plan** is a directory of markdown files describing how a design plan will actually be built. It is the substrate a junior engineer (or an agent) consumes to ship the feature with minimal intervention — every architectural decision should already be made, every non-obvious pattern should already be explained, and every gotcha should already be flagged.

The motto: **optimize for so much rigor up front that implementation has nothing left to debate.** When a step requires the implementer to make a decision the doc didn't make, that's a planning failure, not a feature. Lock the decision, surface it as an Open Question, or name the future sprint that closes it — never punt the call into the code.

An implementation plan IS:
- A directory of files (not a single document) — one **overview**, N **sprint plans**, and a master **progress checklist**.
- A high-fidelity translation of the design plan's decisions into ordered work units.
- A sprint-by-sprint sequencing where each sprint depends only on prior sprints.
- A step-by-step breakdown within each sprint, where each step is a self-contained semantic unit of work and steps form a small dependency graph.
- A living artifact — open questions, decisions log, and per-sprint deviation logs evolve as work progresses.

An implementation plan is NOT:
- A redesign. The upstream design plan is the source of truth for *what* the system is. The implementation plan is *how* and *in what order* it gets built. When a sprint surfaces a question the design plan didn't answer, the answer is logged into the design plan first (or as an amendment), then the sprint proceeds.
- A finished/frozen reference. Sprints get re-scoped, steps get folded together, deviations get recorded mid-implementation. A sprint doc that no longer matches the code is worse than no sprint doc.
- A user-facing product brief. It speaks engineering vocabulary and assumes the reader has the design plan loaded.
- A sprint *schedule*. The work is sequenced by dependency, not by calendar. "Sprint" here means "PR-sized coherent slice," not "two-week ceremony."

The reader you are writing for: a junior engineer with no prior context on this design, joining the project today, who needs to ship the feature one sprint at a time. They have the design plan available; they should rarely need to consult it after picking up a sprint, but it's the tiebreaker on any conflict.

---

## Architecture is inherited, not prescribed

This guide describes the **structure** of an implementation plan — what files exist, what sections they contain, how rounds advance the work. It does **not** prescribe an architecture for the system being implemented. Architecture comes from two upstream sources:

1. **The design plan being implemented.** What the system is, what its components are, what contracts they expose.
2. **The project's existing conventions.** Captured in `CLAUDE.md`, `AGENTS.md` (or the project's equivalent), `CONTRIBUTING.md`, similar prior implementation plans in the repo, and what the existing code actually does.

Before scaffolding or fleshing out a plan, the agent reads both. The patterns the implementation plan adopts — naming, layering, test posture, observability hooks, build workflow, deployment surface, what counts as "shipped" — are inherited from the project. They are not invented by the planner.

Concretely, extract from those sources:

- The layering the codebase uses (e.g., schemas / repositories / managers / resources for a backend service; design tokens / atoms / molecules / organisms for a UI library; core engine / subcommand handlers / CLI surface for a command-line tool; reducer / selector / view for a state-machine app; module / submodule / fixture for an infra-as-code project).
- The directory layout conventions, file-naming conventions, casing.
- The test layering and what each layer covers.
- The build / spec / generation workflow.
- The observability and error-handling conventions.
- The known-footgun set surfaced by prior plans, code-review threads, or post-mortems.

This extraction is load-bearing in every round, not just at scaffold time: re-slicing mid-stream, locking a sprint's Key Design Decisions, or adding a new sprint all re-derive against the same conventions.

Examples throughout this spec lean on a backend-service shape (schemas, repositories, managers, resources, HTTP endpoints, OpenAPI specs, database migrations, transactions, signed URLs) because the precedent plans this guide was built from were backend services. **Treat those as illustrative, not prescriptive.** When implementing for a different kind of project — a frontend app, a CLI, a mobile client, a library, an infrastructure module, a data pipeline — translate every backend-specific example into its equivalent in that stack:

- A schema sprint becomes a design-tokens sprint, or a CLI-engine sprint, or a Terraform-module sprint.
- A "manager throws domain error / resource maps to HTTP" boundary becomes whatever layering the project uses.
- An OpenAPI sync becomes an SDK type-export sync, a GraphQL SDL update, a CLI man-page regeneration, or no spec at all.
- A "no peer-service calls inside a transaction" invariant becomes whatever the project's analogous "don't do X across the Y boundary" rules are.

The structural advice in this guide (Goal → Prerequisites → Deliverables → Locked Decisions → Architecture notes → Steps → Verification; per-sprint Progress mirrored in a top-level `progress.md`; living-during-execution Deviations log) holds in every case. The *content* of those sections must reflect the actual project the agent is working in.

If the project lacks an obvious analogue for a section the spec describes (e.g., no public API surface; no schema; no async workers), drop the section. If the project has surfaces this spec doesn't enumerate, add them. **Adapt to the codebase. Don't bend the codebase to fit the examples in this spec.**

---

## How the document set is produced

### The iteration model

Like design plans, implementation plans are produced through repeated conversation rounds with a human collaborator. Each round resolves a small number of open questions, updates the affected files, and (where helpful) appends an entry to the plan-level `decisions.md` / `status.md` files or to a sprint-scoped log inside a sprint file. The agent should never try to "finish" the plan in one pass — the rounds keep cognitive load low and make trade-offs visible.

**Where the logs live.** Plan-level (cross-cutting) decisions and the round-by-round audit trail live in their **own top-level files** in the plan directory: `decisions.md` and `status.md`. They are **not** sections inside `overview.md`. Each sprint file may additionally carry its own sprint-scoped Decisions log section (for calls that affect only that sprint) and an optional sprint-scoped Status / Feedback-incorporated section (for round entries that materially changed just that sprint). The split is deliberate: `decisions.md` and `status.md` are the plan's single-page indexes for "what got decided across the whole feature" and "how did this plan get here"; per-sprint logs are the local record inside the sprint that owns them.

A round generally goes:

1. **Read the entire plan**, every file in the directory, before saying anything. The overview frames the philosophy; each sprint encodes a slice; the progress file shows what's been completed. Don't rely on prior conversation context.
2. **Triage Open Questions** — pick 1 to ~5 questions that can plausibly be resolved in this round without spawning many more.
3. **Surface alternatives clearly** to the user — present 2–3 viable options for each question with trade-offs, recommend one, and let the user pick or redirect.
4. **Update the affected files** for each resolution:
   - Rewrite the affected sprint section (or overview section) to reflect the chosen approach.
   - Add a tagged bullet to the relevant **Decisions log** — cross-cutting calls go in the plan-level **`decisions.md`**; sprint-specific calls go in the affected sprint file's Decisions log section.
   - Remove the resolved entry from **Open questions**.
   - Add any newly-surfaced sub-questions to Open questions for next round.
5. **Re-shape sprint boundaries when warranted.** A resolution may show that a sprint is too large (split it), too small (fold it into a neighbor), or sequenced wrong (swap order). Sprint files are renamed and the index updated; the progress file is regenerated.
6. **Purge obsolete prose.** When a decision invalidates earlier wording, delete the old wording — don't leave both versions side-by-side.
7. **Append a Status entry** for the round summarizing what changed — to the plan-level **`status.md`**, and (when a single sprint was materially changed) optionally to that sprint file's sprint-scoped Status / Feedback-incorporated section.
8. **Ensure internal consistency.** Rewriting one sprint may leak inconsistency into adjacent sprints (a new dependency, a deferred concern, a renamed helper). Either fix the consequence in the same round or add a note to Open questions for next round.
9. **Emit a completeness assessment to chat** at the end of the round (see § "Round-end completeness assessment"). This is the agent's recommendation on how close the directory is to "the next sprint can be picked up cold and shipped," what's still load-bearing-open, and which nits the user may want to resolve before handing the plan off to an implementer.

### Round 1 — scaffolding

Round 1 is initiated by a human pointing the agent at a design plan and (usually) sketching the high-level slicing they have in mind. The first round produces the skeleton, not the fully-detailed plan:

- The implementation plan **directory** is created.
- `overview.md` is drafted — title, framing, link back to the source design plan, philosophy, locked feature-wide decisions, sprint roster (titles + one-line outcomes), dependency graph. **No Decisions log or Status section inside `overview.md`** — those live in `decisions.md` and `status.md`.
- `decisions.md` is initialized — usually empty, or with `(R1)`-tagged entries capturing the slicing decisions agreed in Round 1.
- `status.md` is initialized with a single Round 1 entry: `**Round 1**: scaffolding + sprint roster + open questions enumerated. _Next:_ <recommended Round 2 focus>.`
- One stub file per sprint is created (`01-<topic>.md`, `02-<topic>.md`, …) with at minimum the Goal, Prerequisites, and an initial Open Questions list scoped to that sprint.
- `progress.md` is initialized with a master checklist where every sprint shows its sprint-level title and (if known) its tentative steps. Each item starts unchecked.

Detailed step breakdowns within each sprint mostly come in later rounds. Don't pre-populate them with assumptions; surface unresolved shape as Open questions.

### Subsequent rounds — fleshing out sprints

Each subsequent round picks one (or a small number) of sprint files and pushes it from "stub" to "ready-to-execute":

- Lock the sprint's Key Design Decisions table.
- Add Architecture notes ("Why X" subsections) for non-obvious choices.
- Sketch the API surface, request/response shapes, and error tables (when applicable).
- Decompose the sprint into ordered steps with concrete Actions / Deliverables / Verification.
- Add a Step Dependency Chart and an Acceptance Checklist.
- Update the master `progress.md` with the new step list for that sprint.

A sprint is "ready to hand off" when a junior engineer could pick it up cold and execute it without consulting the design plan for anything load-bearing.

### Supersession

When a later round invalidates a prior decision (cross-sprint or within a sprint):

1. Update the affected file(s) to reflect the new design.
2. Either edit the existing Decisions-log bullet (preserving its round tag) or add a new one tagged with the current round. Cross-cutting calls live in `decisions.md`; sprint-scoped calls live in the affected sprint file's Decisions log section.
3. Note the supersession explicitly in `status.md`: "Round 5: re-sliced Sprint 06; cross-cutting helper now lands in Sprint 04 instead. R3's helper-in-Sprint-06 decision superseded." If the supersession is sprint-scoped and that sprint has its own Status section, also note it there.
4. **Purge stale wording** from all files where it appears. Old paragraphs left in place become land mines.
5. **Update `progress.md`** if step ordering or step count changed.

### Living during execution: Deviations applied during implementation

Sprint files are also living during execution, not just during planning. When the *implementation* of a step diverges from what the sprint doc said — a column rename, a different idempotency-recovery shape, a swapped error-mapping layer, an extra constraint discovered during coding — the **sprint doc is updated in the same PR** that ships the divergence. There are two complementary surfaces for this:

- **Deviations applied during implementation** — a section near the top of the sprint doc listing the deltas between the original plan and what shipped. The semantic contract from the design plan stays preserved; only the implementation shape is allowed to shift.
- **Per-step Deviations / Post Mortem** — short subsections inline with the affected step, capturing detail too granular for the top-of-doc list (test count, parameter rename, regex tweak, etc.).

A sprint doc that no longer matches the code actively misleads the next reviewer. The PR that introduces the divergence owns the doc update.

### Honest framing for v1 / Tier 0 compromises

Like design plans, implementation plans ship compromises. Label them in place:
- `v1 ships with X — semantics improve once Y lands`.
- `Tier 0 trade-off: …`.
- `Acceptable at this scale; revisit if Z becomes hot.`
- `Schema-additive when we want it.`
- `Defers the chat fan-out to Sprint 11; Sprint 07 ships locally-correct behavior with a known cross-service gap.`

A reader should never have to guess whether a piece of a sprint is the long-term answer or a deliberate stop-gap.

---

## Sizing heuristics for sprint slicing

Use these as guard-rails when sketching the initial roster — and when re-slicing mid-stream. Heuristic, not strict.

**Typical shape.** 5–12 sprints per feature; 5–10 steps per sprint. A step is roughly half a day of focused work; a sprint is roughly one to two weeks. Real plans drift from these numbers and that's fine — they're calibration anchors, not targets. If the roster comes out at 3 sprints, the slicing is probably too coarse; if it comes out at 20, the slicing is probably too fine (or some sprints should be folded).

**Common sprint archetypes.** Most sprints fall into one of a handful of shapes; recognizing which archetype you're slicing helps name the deliverables and the test surface. The archetypes below are stack-agnostic; the *content* of each in your project depends on the architecture extracted from the project's conventions:

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

**Too big / too small.** If a sprint's Deliverables list runs to ~15+ artifacts, it's probably too big — re-slice. If a sprint has ≤ 2 Deliverables and shares a clear seam with a neighbor, it's probably too small — fold.

**During iteration.** Re-slicing is normal once sprint-level work begins. When a round surfaces that a sprint is too large, too small, or sequenced wrong, apply these heuristics and follow the supersession + `progress.md` regeneration steps in "How the document set is produced."

---

## Directory layout

The implementation plan lives in a single directory. The canonical convention is `<root>/impl/`, a sibling of the feature's `<root>/design.md`. Users can override the location, but `<root>/impl/` is what the trellis skill creates by default:

```
<root>/
├── design.md              source-of-truth design plan
└── impl/                  canonical implementation plan directory
    ├── overview.md
    ├── decisions.md
    ├── status.md
    ├── progress.md
    ├── post-mortem.md     # created lazily; first appears when the first sprint ships
    ├── 01-<topic>.md
    ├── 02-<topic>.md
    ├── …
    └── NN-<topic>.md
```

Rules:
- `overview.md` is required. It's the entry point — `decisions.md`, `status.md`, `progress.md`, and every sprint file links back to it. **`overview.md` does not contain a Decisions log or a Status section** — those live in `decisions.md` and `status.md` respectively. This is non-negotiable: a Decisions log inside `overview.md` is a planning bug.
- `decisions.md` is required. It is the plan-level (cross-cutting) Decisions log — the single page a returning collaborator scans to recover "what got decided across the whole feature." Sprint-scoped Decisions logs continue to live inside their sprint files; `decisions.md` holds only the cross-cutting calls. See "`decisions.md` anatomy" for the format.
- `status.md` is required. It is the plan-level round-by-round audit trail — the doc's "git log." Sprint-scoped Status / Feedback-incorporated entries continue to live inside their sprint files; `status.md` holds the plan-wide narrative. See "`status.md` anatomy" for the format.
- `progress.md` is required. It is the master checklist that humans scan to see "what's done."
- `post-mortem.md` is created the first time a sprint's final step is checked off — not at scaffold time. It accrues a distilled, sprint-keyed entry per shipped sprint. See "`post-mortem.md` anatomy" for the format.
- Sprint files are zero-padded numerically prefixed (`01-`, `02-`, …) so directory order matches execution order. Re-numbering on a re-slice is fine.
- Sprint file names are `<NN>-<short-kebab-topic>.md` — short enough that the index table reads cleanly, descriptive enough that a `git log` line is meaningful.
- One file per sprint. Don't split a sprint across two files; if a sprint is too large to fit comfortably in one file, that's a re-slice signal.
- **The overview file is `overview.md` — it is not numbered, not prefixed, not a sprint.** Only sprint files take the `NN-` prefix. Numbered prefixes on `overview.md`, `decisions.md`, `status.md`, `progress.md`, or `post-mortem.md` are wrong.

---

## `overview.md` anatomy

Drop sections that don't apply. Add domain-specific sections where the work demands them. The list below is the canonical superset.

### 1. Title & framing block

```
# <Feature> — Implementation Plan
```

Followed by 1–3 sentences saying what this directory contains and a clear link back to the source design plan. Under the canonical `<root>/design.md` + `<root>/impl/` layout, that link is `../design.md`:

> Implementation plan for the [Lab Results & Measurements design plan](../design.md). Read that document end-to-end before picking up Sprint 01 — every sprint assumes its vocabulary.

If the user overrode `<impl-plan-dir>` so it's not a sibling of `design.md`, the link points at the actual design-plan path instead of `../design.md`. Either way: the link back to the design plan is **not optional**. The design plan is the upstream source of truth; an implementation plan that doesn't cite it is a red flag.

### 2. Philosophy

A handful of subsections capturing the cross-cutting posture this plan adopts. Common subsections:

- **The plan is the source of truth.** Sprint docs are execution plans, not redesigns. New decisions go back into the design plan (or land as amendment records) before the sprint proceeds.
- **Tier-0-aware, but never tier-0-locked.** The plan ships v1 compromises; v1 must not preclude later tiers.
- **Cross-service contract over internal cleverness.** Restate the relevant boundary rules.
- **Test the cross-service seams hardest.** Where the integration tests live; what they cover.
- **Small, shippable sprints.** Each sprint leaves the codebase buildable, testable, demonstrable. Type checks pass. Test suite passes. No half-finished states between sprints.

These read as a short manifesto — a paragraph or two each. They orient the reader on the moves the plan deliberately did not make.

### 3. Architectural invariants the sprints uphold

A numbered list of cross-cutting invariants every sprint must verify (often in tests) when it touches them. These are restated from the design plan + the project's conventions, but called out here because **every sprint is responsible for not violating them**.

The exact invariants are project-specific. Examples from a backend-service shape: append-only constraints on certain tables, no peer-service calls inside DB transactions, no cross-schema foreign keys, every mutating endpoint accepts an idempotency key. A frontend project might instead pin: no direct DOM manipulation outside the renderer, all network calls go through the SDK layer, no `any` in a public component prop. A CLI might pin: no global mutable state, every subcommand accepts `--json`, no implicit network access without `--remote`. Inherit the actual list from the project; don't invent generic ones.

### 4. Module / directory layout impact

A code-block tree showing what the codebase looks like at the end of the plan. New files / new directories called out. Any new touchpoints in peer services listed separately. Useful both for orientation and for spotting when a sprint silently introduces a file the layout didn't mention.

### 5. Cross-sprint conventions

Project-specific rules every sprint inherits. The goal is to pin one canonical answer per axis so individual sprints don't re-decide. Identify the axes that apply to *your* project from `CLAUDE.md`, similar prior plans, and the existing code.

Common axes (drawn from a backend-service example; adapt to your stack — drop axes that don't apply, add axes that do):

- **Branching / source-control workflow.**
- **Build / type / test / lint commands** the sprint must leave green.
- **Migration / schema-change workflow** (if the project has one) — who runs the generator; what each sprint that touches the schema must enumerate for review.
- **Reference / seed data** (if applicable) — where it lives, whether it's auto-applied or human-run.
- **Generated-spec / contract sync** (e.g., OpenAPI, GraphQL SDL, gRPC `.proto`, exported SDK types, published man pages, regenerated component-library docs) — which sprints update which spec.
- **Tests** — framework, test location, fixture conventions, stub patterns, what test layers cover what seams.
- **Logging / observability** — logger choice, category-prefix or trace-attribute conventions, metric naming.
- **Errors** — error vocabulary, mapping discipline at the boundary layer.
- **Idempotency** — the project-wide idempotency-key shape, if mutating surfaces exist.
- **Feature flags** — project posture (often "none — if it isn't ready, it isn't merged"; some projects use them heavily).

A frontend project might also pin design-token sources and accessibility-test thresholds; a CLI might pin completion / man-page generation; a mobile app might pin app-config rollout policy; an infra project might pin Terraform module versions and apply gates. Add whatever axes the project enforces.

### 6. Sprint organization

Two subsections:

- **Rationale for the sprint order** — one paragraph per sprint explaining why it sits where it does in the sequence and what it unlocks.
- **Rationale for sprint-sized slices** — a short statement of the slicing principle. Aim for sprints that:
  - Each ship a coherent, independently-testable deliverable.
  - Each leave the codebase buildable, type-checking, and test-passing.
  - Each fit within roughly one to two weeks of focused work for a single engineer.
  - Form a directed graph where each sprint depends only on prior sprints (cycles are a slicing bug).
  - Are sequenced so foundational layers land before logic, which lands before surfaces.

  Sprints are not equal in size — they're equal in **shippability**. See the "Sizing heuristics for sprint slicing" section for calibration anchors and archetypes.

### 7. Sprint roster + dependency graph

A table indexing every sprint:

```
| #  | Sprint                                                   | Outcome                                                              |
| -- | -------------------------------------------------------- | -------------------------------------------------------------------- |
| 01 | [Schema, types, repositories](./01-schema.md)            | The four tables land with all indexes, partial uniques, the CHECK… |
| 02 | [Manager core: groups, supersession, soft-delete cascade](./02-manager-core.md) | Manager with deterministic type resolution, group construction… |
| …  | …                                                        | …                                                                    |
```

When the dependency graph isn't a straight line, an ASCII chart is the standard form:

```
01 ──► 02 ──► 03 ──► 06 ──► 07 ──► 08 ──► 10
        │      │                    ▲
        ├─────►04 ───────────────┐  │
        │                        │  │
        └─────►05 ───────────────┴──┤
                                    │
                                09 ─┘
```

Followed by a one-line bullet per edge explaining *why* sprint X depends on sprint Y.

### 8. Feature-wide locked decisions

A table of decisions that apply across **every** sprint — the ones every sprint would otherwise re-litigate. Sprint-level decision tables refine these but never override them.

The table is dense, deliberately. It's the "stop-relitigating-this" surface. When a sprint's locked decisions section says "see overview decision X," the answer should be in here.

**Categories that typically earn rows** (project-agnostic — pull the ones that apply to *your* stack from the project's conventions; drop the rest; add categories the project enforces but this list doesn't enumerate):

| Category                       | What it pins                                                                       |
| ------------------------------ | ---------------------------------------------------------------------------------- |
| Durable-state location         | Where the system stores its persistent data (schema file, store module, state file, terraform module path). |
| Lifecycle markers              | How rows / records / entities express their lifecycle (soft-delete column, status enum, archive flag, cache eviction policy). |
| Default-view / default-read    | The filter every "default" read applies (active-only, non-archived, current-tenant, current-environment). |
| Async / background cadence     | When background work runs (cron schedule, debounce window, queue worker concurrency). |
| Idempotency-key shape          | How retry-safe surfaces are gated (deterministic key derivation, dedup window, replay tolerance). |
| Banned patterns                | Things no sprint may introduce (hand-written migrations; specific imports; specific globals; specific frameworks). |
| Generated-spec sync source     | Which artifact is canonical when multiple representations exist (OpenAPI, GraphQL SDL, exported types, regenerated docs, man pages). |
| Test-layer assignment          | Which test layer owns which seam (unit / integration / smoke / visual / e2e), where each lives. |
| Logging / metric naming        | Logger choice, category-prefix or trace-attribute conventions, metric-name format. |
| Error vocabulary               | The set of error codes / exit codes / domain-error classes the plan commits to.    |
| Feature-flag / gating posture  | "None — if it isn't ready, it isn't merged" vs. specific flag-system + naming.     |

#### Concrete example (project-specific — translate to your stack)

> The table below is taken from a backend-service plan in a Next.js / Drizzle / Graphile-Worker codebase. The shapes, file paths, and identifiers are deliberately specific to that project — they will not match yours. The *categories* (rows) generalize; the *values* (right column) almost certainly don't. Do not copy values verbatim.

```
| Decision                  | Value                                                                            |
| ------------------------- | -------------------------------------------------------------------------------- |
| Schema location           | `src/services/patient/db/schema.ts` (extends existing `pgSchema('patient')`)     |
| Soft-delete               | `deletedAt: timestamp()` nullable column on … . **No** soft-delete on …          |
| Default-view filter       | `WHERE deletedAt IS NULL AND status = 'active' …` — embedded into every default… |
| Worker cadence            | `*/1 * * * *` cron; batched, idempotent, skip already-evaluated rows             |
| Banned patterns           | (a) Hand-written migrations. (b) `import 'server-only'` in worker-loadable code. |
| …                         | …                                                                                |
```

#### Parallel example (different stack)

> The same categories, populated for a frontend-app plan in a React / Tanstack Query / Vite codebase. Different values, same shape — the table is a stack-translation exercise, not a copy job.

```
| Decision                  | Value                                                                            |
| ------------------------- | -------------------------------------------------------------------------------- |
| Durable-state location    | `src/state/<feature>/store.ts` (Zustand slice; persisted via `idb-keyval`)       |
| Lifecycle markers         | `archivedAt: number \| null` on entity rows; **no** hard-delete in the client    |
| Default-view filter       | `archivedAt === null && tenantId === currentTenantId` in every list selector     |
| Async / background cadence| Tanstack Query: `staleTime: 30_000`; refetch on focus only for "live" surfaces   |
| Idempotency-key shape     | `crypto.randomUUID()` per mutation, attached to the optimistic-update record     |
| Banned patterns           | (a) Direct `fetch` outside the SDK layer. (b) `any` in public component props.   |
| Generated-spec sync source| `openapi-typescript`-generated types; sprint that touches the API regenerates.    |
| …                         | …                                                                                |
```

Pick the categories that apply to your project, populate them with values pulled from your project's conventions, and drop the rest. The number of rows that actually appear in a real plan is usually 10–25 — fewer is a planning-failure signal (most decisions are still implicit), more is a sign you're locking sprint-scoped decisions at the overview level.

### 9. Out-of-scope across all sprints

A bulleted list captured once at the overview level, so each sprint's "Out of scope" can forward-link without re-stating. Be aggressive: declaring something out of scope here is the highest-leverage move for keeping every sprint tight.

### 10. Open questions

A numbered list of items not yet decided across the plan as a whole. Each entry includes:
- The question, briefly.
- Why it's open / what makes it hard.
- An indicative resolution direction or an explicit "deferred until X."
- Which sprint(s) it blocks.

When a round resolves an entry, it **moves to** the Decisions log; it does not stay in Open questions with a "(resolved)" tag.

### 11. What this plan does *not* try to do

A short bulleted list at the overview level so individual sprint "Out of scope" sections stay short. Explicitly forward-looking: things the design plan covers but this implementation plan defers.

### 12. How to read each sprint

A one-paragraph summary of the sprint anatomy and a pointer at "the design plan is the source of truth on any conflict." Also note where the logs live: cross-cutting Decisions and the round-by-round Status are in `decisions.md` and `status.md`; sprint-scoped decisions and (optional) sprint-scoped Status entries are inside the sprint file itself.

> **`overview.md` does not contain a Decisions log or a Status section.** Those are top-level files in the plan directory (`decisions.md`, `status.md`) — see the next two sections for their anatomy.

---

## `decisions.md` anatomy

`decisions.md` is the plan-level (cross-cutting) Decisions log. It is its own top-level file in the plan directory — **not** a section inside `overview.md`. A returning collaborator opens `decisions.md` to recover "what got decided across the whole feature, and when." Sprint-scoped Decisions logs continue to live inside their sprint files; `decisions.md` holds only cross-cutting calls (the calls every sprint would otherwise re-litigate).

Skeleton:

```markdown
# <Feature> — Decisions

> Part of the [<Feature> Implementation Plan](./overview.md).
>
> Cross-cutting decisions only. Sprint-scoped decisions live in the relevant sprint file's Decisions log section.

- **<Decision lead in bold>.** <Brief restatement / rationale.> (R<n>)
- **<Decision lead in bold>.** <…> (R<n>)
- …
```

Rules:

- One bullet per cross-cutting decision. Each bullet **leads with a bold phrase** that names the call, followed by the rationale, followed by the round tag `(R<n>)`.
- Round tags are required on every bullet. The tag is the audit trail's anchor.
- Concrete examples:

```
- **Sprint roster sliced into 10 PR-sized units.** (R1)
- **Worker cadence: 1-minute cron, batched and idempotent.** (R3)
- **Push fan-out is post-commit and best-effort; outbox deferred.** (R5)
```

- **`decisions.md` does not hold sprint-scoped calls.** A decision that only affects one sprint stays in that sprint file's Decisions log. If a sprint-scoped decision later becomes cross-cutting (a second sprint takes a dependency on it), promote it to `decisions.md` and add a note in the new round's `status.md` entry. Don't duplicate.
- **Compression discipline.** When a later round supersedes a bullet, condense the older entry to its bold lead + round tag + supersession pointer (`- **<lead>.** Superseded by R<m>. (R<n>)`) and drop the rationale paragraph. The audit trail (round number + supersession link) survives compression; the obsolete prose does not. Keep the last 2–3 rounds in full; keep any still-actively-load-bearing entry in full regardless of age. The same compression discipline applies to every sprint's local Decisions log section.
- **Don't add narrative connective tissue** between bullets. `decisions.md` is a flat list, not an essay.

The compression rule and the round-tag rule apply identically inside any sprint file's sprint-scoped Decisions log section.

---

## `status.md` anatomy

`status.md` is the plan-level round-by-round audit trail — the doc's "git log." It is its own top-level file in the plan directory — **not** a section inside `overview.md`. A user resuming the plan via `impl-iterate` reads the latest entry's `_Next:_` clause to recover where to pick up. Sprint files may additionally carry a sprint-scoped Status / Feedback-incorporated section for round entries that materially changed just that sprint; `status.md` holds the plan-wide narrative.

Skeleton:

```markdown
# <Feature> — Status

> Part of the [<Feature> Implementation Plan](./overview.md).
>
> Plan-level round-by-round audit trail. Sprint-scoped Status / Feedback-incorporated entries live in the relevant sprint file when a round materially changes a single sprint.

- **Round 1**: <summary>. _Next:_ <one-line recommended focus for next round, citing Open question IDs + tags + scope where relevant>.
- **Round 2**: <…>. _Next:_ <…>.
- …
```

Each entry has two clauses:

1. **What changed.** Summary of resolutions / re-slices / supersessions in this round. Cite supersessions explicitly (`R5 supersedes R3's helper-in-Sprint-06 decision`).
2. **`_Next:_` clause** — a one-line italic tail naming the recommended next-round focus. Cite Open question IDs + tags + scope when relevant (`overview Q4 [blocks-v1]`, `Sprint 03 Q2 [blocks-impl]`). This persists the between-rounds recommendation into the doc itself, so a user resuming via `impl-iterate` recovers the prior recommendation without depending on chat history. When the plan is complete, the `_Next:_` clause is `hand off to implementation` (or `run impl-review first`).

Examples:

```
- **Round 1**: scaffolding + sprint roster + open questions enumerated. _Next:_ lock Sprint 01 to execution-ready.
- **Round 2**: foundational decisions table locked; Sprint 01 fleshed out. _Next:_ lock Sprint 02 — resolve the supersession-cascade question (Q4 [blocks-impl]) first.
- **Round 3**: Sprint 06 split into 06-resolve and 07-deduplicate; cron cadence locked. _Next:_ resolve the worker-idempotency-key shape (overview Q11 [blocks-v1]) before touching Sprint 07.
```

Rules:

- One entry per planning round. Round numbering is contiguous and grep-able — **never delete a round entry outright**, even when compressing.
- The round counter is plan-level (incremented in `status.md`). Sprint-scoped Status entries reuse the same round number; they don't run their own counter.
- Status entries are append-only **for the round you are appending** — never edit *prior* entries to falsify what happened.
- **Compressing older entries.** Like `decisions.md`, `status.md` grows monotonically as rounds accumulate; without active pruning, a long plan's log becomes unreadable. When you append a new round, also walk the older entries and condense any whose detail no longer carries weight: rounds whose `_Next:_` clause is long-finished drop the clause entirely; rounds whose summary detail is no longer load-bearing (the decisions they narrate were themselves superseded by later rounds) collapse to a one-line summary `**Round <n>**: <one-clause summary>`. Keep the last 2–3 rounds in full. Never delete a round outright. The same compression rule applies to every sprint's local Status section.
- **Sprint-scoped Status entries.** When a round materially changes a single sprint, append a sprint-scoped entry to that sprint file's Status / Feedback-incorporated section *in addition to* the entry in `status.md`. The sprint-scoped entry can be terser — `status.md`'s entry carries the cross-cutting narrative. For a *shipped* sprint, leave its older Status / Deviations entries alone — they are part of the historical record of what shipped and are no longer eligible for compression by a planning pass.

---

## Sprint document anatomy

Every sprint file follows the same skeleton. Drop sections that don't apply (a no-API sprint has no API surface section); add domain-specific sections where the work demands them.

### 1. Title & overview pointer

```
# Sprint NN — <Short title>

> Part of the broader [<Feature> Implementation Plan](./overview.md).
```

### 2. Goal

A single sentence — what this sprint, when done, makes possible. Not a list of activities; a description of the outcome.

> Land the four core tables (`measurement_types`, `measurement_groups`, `measurement_series`, `measurements`) plus their enums, partial indexes, polymorphic-parent CHECK, payload Zod schemas, and pure-DB repository functions. Nothing public ships in this sprint.

### 3. Prerequisites

The prior sprints (and any cross-service surfaces) whose deliverables this sprint assumes. Each prerequisite lines up with a sprint-index entry or a peer-service helper.

> - Sprint 01 (`media.storage_providers` table, `S3ClientAdapter`, `S3SignedUrlMinter`).
> - A `social.canInvite(actorId, targetId) → boolean` `internal_function` resource. If absent, this sprint creates it (Step 1 has the audit / build-if-absent branch).

### 4. Deliverables at end of sprint

A bulleted list of concrete, observable outcomes. Files, endpoints, tables, indexes, repositories, tests. The "done" gate as a list of artifacts.

### 5. Out of scope

What explicitly does *not* land in this sprint even if tempting. Each item ideally forward-links to the sprint that does own it.

### 6. (Optional) Feedback incorporated / Deviations applied during implementation

Two related but distinct surfaces:

- **Feedback incorporated (post-review)** — when a planning round produces non-trivial corrections to the sprint doc, capture them as a list near the top so the next reader can see what changed without diffing.
- **Deviations applied during implementation** — the deltas between the inline code samples in the sprint and what actually shipped. The semantic contract from the design plan stays preserved; only the implementation shape shifted. Recorded so reviewers don't read the divergence as drift. Per-step deviations live inline with the step, but the top-of-doc list catches cross-cutting deltas.

### 7. Key design decisions, locked before implementation

A table of the small choices the design plan didn't fix, decided here so the steps are unambiguous. Same form as the overview's locked-decisions table, but scoped to this sprint:

```
| Decision                                | Value                                                                       |
| --------------------------------------- | --------------------------------------------------------------------------- |
| Endpoint path                           | `POST /chat/conversations` (relative to `/api/chat/v1`)                     |
| Response on existing-found              | HTTP **200** with the existing conversation (idempotent rejoin)             |
| Race on idempotent create               | `INSERT ... ON CONFLICT (post_id, direct_pair_key) DO NOTHING RETURNING *`  |
| `targetUserId` existence check          | Manager does **not** verify directly — `social.canInvite` is the gate       |
| …                                       | …                                                                           |
```

Each row a question that, left implicit, would become a mid-coding decision.

Cover the categories that produce review-time debate. The exact list depends on the project; adapt accordingly. Examples from a backend-service shape: identifier strategies (UUIDv7, ULID, monotonic int), enum / option values, env-var or config-key names and defaults, TTLs and quotas, error codes, FK / cascade or referential-integrity behavior, retry policies, transaction boundaries, allow-list contents, naming conventions. A frontend project might instead lock: design-token names, component prop shapes, route paths, analytics event names, feature-flag keys. A CLI might lock: subcommand names, flag aliases, exit-code semantics, completion-script update policy. The principle is the same: **anything that would otherwise become a mid-coding decision belongs in this table.**

**Aim for 10–25 rows. A single-row Locked Decisions table is a planning failure** — it means the sprint will make most of its decisions inside the steps. If a decision genuinely can't be locked yet, include the row with the rationale and name the sprint where it'll close — don't omit it.

Every Locked Decisions table should resolve answers to the recurring questions for *this* project, even briefly. The exact questions depend on the project type; some examples (drawn from a backend-service shape, with the equivalent in other stacks in parens):

- **Where does the durable state live?** (Schema, table-name conventions, casing — for projects with a database. The equivalent in a frontend project might be: which store / cache / persistence layer; in an infrastructure project: which Terraform module / state file.)
- **What's the smallest atomic surface increment?** One endpoint, one component, one CLI subcommand, one library function — or a tightly-coupled pair (e.g., provision + complete)?
- **What's the contract with adjacent components?** Naming, calling conventions, dependency direction, audit-or-build branches for helpers expected to exist in peer modules / services.
- **What's the test surface?** Which test layers (unit / integration / smoke / visual / e2e) cover this sprint's deliverables; whether real external dependencies are exercised.
- **What's deferred?** Hardening, quotas, observability, polish — usually punted to a later sprint.
- **What does idempotency / retry-safety look like?** Whenever the surface includes mutations a client may retry.

#### Scope-guardrail rows: `Avoid in this sprint` vs `Banned in this sprint`

A common Locked Decisions row names files / directories / patterns this sprint should not touch. Use two distinct labels — they tell the executor how rigidly to read the rule:

- **`Avoid in this sprint`** — soft scope guardrail. The sprint's intended work does not edit these surfaces, but minor mechanical changes required to keep the code compiling (a renamed argument that must thread through a call site, an enum case that must be matched, a removed identifier that has to be replaced at the reference) are permitted. The executor records the change as a Deviation and continues.

- **`Banned in this sprint`** — hard scope guardrail. The sprint's design depends on these surfaces staying as they are: a deferred type or field that must not be pre-staged, a schema version that must not bump, a security / authorization layer that must not relax, an explicitly out-of-scope module. Even here, minor mechanical propagation is allowed (and recorded as a Deviation), but anything beyond mechanical — logic edits, new behavior, a different parameter shape, a new field — means the plan is wrong; the executor stops and the user routes to `impl-iterate`.

Default to `Avoid` for scope shaping. Reserve `Banned` for the few entries that name a real design / security / planning guardrail. If every entry in the row would be `Avoid`, fold them into "Out of scope" instead — the guardrail row earns its place only when a future executor might plausibly think they should touch the surface and needs a posture cue.

```
| Decision                                | Value                                                                                                                                                            |
| --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Avoid in this sprint                    | (a) Retired view files under `src/views/archive/*`. Identifiers may need a renamed-argument propagation; do no more. (b) Documentation that describes the surface. |
| Banned in this sprint                   | (a) Bumping `schemaVersion` — Sprint NN owns the migration. (b) Pre-staging the deferred `usageMetrics` field, even unused. (c) Relaxing the auth predicate.     |
```

### 8. Architecture notes

Short paragraphs justifying non-obvious shape decisions. The sprint analog of the design plan's "Why X" subsections. Almost every sprint earns at least one. Examples:

- **Why pre-BEGIN existence check + idempotent fallback inside the transaction.** …
- **Why participant rows are inserted in the same transaction.** …
- **Why a separate response code for created vs rejoined.** …
- **Why `MediaTx` is a free function in `db/tx.ts`, not a per-class method.** …

If a reader could plausibly ask "why this shape?" the architecture notes section is where the answer lives. Each bullet defends the choice, names the rejected alternative when one competed, and warns about adjacent bugs the choice creates or avoids.

### 9. Public surface (when applicable)

Whatever the externally-visible contract for this sprint is — HTTP endpoints, GraphQL operations, gRPC methods, SDK methods, CLI commands, UI components / routes, library exports, public events. Document the shape concretely: input form, output form, failure modes.

Adapt the form to the project. The structural goal in every case: a reader knows the exact contract this sprint commits to, and every documented failure mode is reachable from a crafted input.

For an HTTP-style surface, the standard form is request + response example bodies plus an error table:

```
| Status | Code | Cause |
|---|---|---|
| 400 | `INVALID_INPUT` | Validation failure |
| 401 | `UNAUTHORIZED` | No / invalid credential |
| 403 | `INVITE_NOT_ALLOWED` | Authorization predicate returned false |
| 404 | `POST_NOT_FOUND` | Resource does not exist |
```

Distinguish externally-callable vs. internal surfaces if the project draws that line (e.g., `public_api` vs. `internal_function`). Field-level shape lives in transport-schema sketches:

```ts
const postBody = z.object({
  kind: z.literal('direct'),
  postId: z.string().uuid(),
  targetUserId: z.string().uuid(),
});
```

For a CLI surface, enumerate flags, positional args, exit codes, and stdout/stderr contracts. For a UI surface, sketch component props, state machines, accessibility roles, and visual states. For an SDK / library surface, list method signatures, return shapes, and thrown / rejected error types. Whatever the form, surface every failure mode the sprint commits to.

### 10. Progress

A markdown checklist of every step in the sprint, each starting unchecked:

```
## Progress

- [ ] Step 1 — Add columns.ts payload aliases
- [ ] Step 2 — Add enums to schema.ts
- [ ] Step 3 — Add the `measurementTypes` table
- [ ] Step 4 — Add `measurementGroups` table
- [ ] Step 5 — Add `measurementSeries` table
- [ ] Step 6 — Add `measurements` table
- [ ] Step 7 — Add types.ts entity types and Zod payload schema
- [ ] Step 8 — Repository: `measurement-type.ts`
- [ ] Step 9 — Repositories: `measurement-group.ts`, `measurement-series.ts`, `measurement.ts`
- [ ] Step 10 — Generate and apply migration (gated)
```

The sprint-level Progress is a duplicate of the corresponding section in `progress.md`, intentionally — the per-sprint progress lets a reader inside the sprint file see immediate state without context-switching to the master file. Both are kept in sync.

### 11. Implementation Steps

Each step is its own subsection. Ordered. Self-contained.

```
## Step N — <Short title>

**Goal**: one sentence — what this step delivers when done.

**Build/test boundary**: omitted when the step leaves the repo compilable and testable; required when it does not. If present, it must say this step may leave the code uncompilable / untestable, why that cannot be avoided, and which later step restores the gates.

**Actions**:

1. Concrete action with file paths and code samples where helpful.
2. The next concrete action.
3. **Subtle bug**: <gotcha>. Spelled out so a reader doesn't trip on it.

**Deliverables**:
- File(s) created or modified, observable artifacts.

**Verification**:
- How to confirm this step is correct (tests, type checks, manual checks, psql commands).
```

The four headings — **Goal / Actions / Deliverables / Verification** — are mandatory for every step. They make a step graspable in 30 seconds and unambiguous when executed. **Build/test boundary** is usually omitted, but becomes mandatory for the rare step that cannot leave the repo compilable and testable.

Step content guidance:

- **Code samples are encouraged.** When a column shape, an index predicate, a transaction skeleton, or a Zod schema is the actual decision, write the code. Don't paraphrase it. A junior engineer should rarely have to reach for the design plan to disambiguate.
- **Inline gotchas — "Subtle bug:" lines.** Promote them above prose when the implementer would otherwise stumble. Examples: index ordering (`.desc()` belongs in the column list, not the WHERE), tx-aborted state (use ON CONFLICT, don't catch 23505), enum CHECK comparison form, drizzle vs SQL casing collisions.
- **Cross-step / cross-sprint dependency notes.** When a helper this step ships is consumed by a later sprint, say so. When this step depends on a helper already shipped, say so.
- **Every step should be a compilable/testable boundary.** Slice steps so each completed step can be committed without leaving the code uncompilable, untypecheckable, or untestable. If a temporary violation absolutely cannot be avoided, the step must include a **Build/test boundary** paragraph naming the expected failure, why no safer slicing exists, and the later step that restores the repo to a green state.
- **Verification per step is non-negotiable.** "How do I know this step is done?" must have a concrete answer — a test name, a `yarn check:types` run, an SQL query, a manual smoke test. "It compiles" is not verification.
- **Optional Pre-step block.** When a step has setup that must happen before the actions begin (a policy matrix to confirm with a peer, an env var to source), call it out as a `**Pre-step —**` paragraph between Goal and Actions.
- **Step sizing.** A step is roughly half a day's focused work. If it balloons past that, split it; if it's under an hour and shares a seam with a neighbor, fold it. Step counts of 5–10 per sprint are typical.
- **Code-sample sizing.** Aim for 10–30 lines per snippet — enough to make the contract unambiguous, not the whole module. Pick the snippet that pins the contract: function signature, table builder, Zod schema, error-mapping table.
- **Name env vars and their defaults inline.** When a TTL, cap, allow-list, or feature path is config-driven, the step says the env var and the default value (`MEDIA_MAX_BYTES_IMAGE=8388608`). Vagueness produces drift.
- **Caveats baked into code comments.** When an invariant must survive future edits at the source level (an ordering constraint, a never-update column, a reason a value is hard-coded), instruct the implementer to leave a one-line comment in the code itself — not just in the doc. Code comments survive doc rot.
- **Verification names commands and assertions.** "Tests pass" is not a check. Concrete commands (`yarn check:types`, `psql -c "\d+ patient.measurements"`, `swagger-cli validate api/openapi.yml`) and concrete assertions ("raises sqlstate `23514` against constraint `measurement_parent_xor`", "URL host equals CDN host", "trigger advances `updated_at` across separate transactions") are. Manual checks are explicitly labeled as such.

### 12. Step Dependency Chart

A small ASCII chart showing which steps gate which. Helps the implementer parallelize what's parallelizable.

```
1 (columns.ts) ──► 7 (types.ts)
                       │
2 (enums) ──► 3 (types) ──► 4 (groups) ──► 5 (series) ──► 6 (measurements)
                       │           │            │              │
                       └───────────┴────────────┴──────────────┴──► 8 (type repo)
                                                                       │
                                                                       └──► 9 (other repos) ──► 10 (migration)
```

Followed by a few one-liners on what's truly parallelizable and what's hard-sequenced.

### 13. Acceptance checklist for the sprint as a whole

The user-visible "done" gate as a checklist. Same checkbox format as `progress.md` but scoped to acceptance criteria, not per-step completion:

```
- [ ] Every index from the design doc appears with the matching name.
- [ ] `measurement_parent_xor` CHECK is enforced (constraint-violation test passes).
- [ ] `yarn check:types` passes.
- [ ] Repository integration tests pass against a real DB.
- [ ] CI green on `main` after merge.
```

This is *not* the same as the per-step Verification — Verification proves a step is done; the Acceptance checklist proves the *sprint* is done.

Recurring acceptance items that earn a row in most sprints, beyond domain-specific assertions:

- **Cross-step invariants restated as checks** — the things that would silently ship wrong if missed (a constraint omitted, a wiring step skipped, work in the wrong order, a cascade direction that defeats soft-delete).
- **Every documented failure mode is reachable from a crafted input.** If the public surface lists an error / exit / failure case, a test drives that path.
- **Generated specs / docs are updated** (or explicitly unchanged — say which, in the same PR). Applies to whatever the project regenerates: OpenAPI, GraphQL SDL, exported type definitions, README sections, man pages, component-library docs.
- **Tests live at the right layer for the project.** Inherit the project's test layering (unit / integration / smoke / visual / e2e) and verify the sprint's coverage matches the seam it touched. A sprint that ships only unit-test coverage when an integration seam landed has missed something.
- **Build / type / lint / CI is green on the merge target.**

### 14. (Optional) Deviations

Per-step deviations between the original plan and what shipped, captured inline next to the affected step or in a top-of-step `### Step N deviations` subsection. The granular complement to the top-of-doc "Deviations applied during implementation" list.

### 15. (Optional) Post Mortem

After-the-fact notes that don't affect the contract but are worth preserving — naming-convention drift, a partially-mitigated risk, a follow-up worth filing. These are stylistic / non-actionable observations that future maintainers benefit from but that don't gate sprint acceptance.

**Distillation into `post-mortem.md`.** When the sprint's final step is checked off (i.e., the sprint ships), distill this section into a sprint-keyed entry in the directory's top-level `post-mortem.md`. Create the file if it doesn't yet exist; otherwise append a new section for this sprint. The inline Post Mortem section in the sprint file remains the authoritative long-form record; `post-mortem.md` is the cross-sprint reading surface so reviewers can scan all post mortems in one place without opening every sprint file. The distillation lands in the same PR that ships the sprint's final step. See "`post-mortem.md` anatomy" below.

---

## `progress.md` anatomy

`progress.md` is a flat markdown checklist mirroring the master sprint+step structure. It is the single page a human visits to answer "what's done?"

Skeleton:

```markdown
# <Feature> — Progress

> Part of the [<Feature> Implementation Plan](./overview.md).

## Sprint 01 — Schema, types, repositories

- [x] Step 1 — Add columns.ts payload aliases
- [x] Step 2 — Add enums to schema.ts
- [x] Step 3 — Add the `measurementTypes` table
- [ ] Step 4 — Add `measurementGroups` table
- …

## Sprint 02 — Manager core

- [ ] Step 1 — …
- [ ] Step 2 — …
- …
```

Rules:

- One section per sprint. Sprint heading matches the sprint file's title.
- One bullet per step. Bullet text matches the step heading in the sprint file (`Step N — <title>`).
- Boxes are `[ ]` until the step is complete; `[x]` when it ships. Update the box at the moment the step lands, not in batch.
- The per-sprint Progress section in each sprint file is kept in lockstep with `progress.md` — the per-sprint copy is for in-context reading; `progress.md` is for the bird's-eye scan.
- When sprints are re-sliced or steps are re-ordered, `progress.md` is regenerated in the same round.
- Don't add narrative or commentary here. `progress.md` is a checklist, not a status report.

---

## `post-mortem.md` anatomy

`post-mortem.md` is the cross-sprint, distilled, after-the-fact reading surface. It accrues one section per shipped sprint, in sprint order. A reader who wants to know "what surprised us across this whole feature?" reads this file end to end.

It is **not** a status log, a deviation log, or a re-statement of the sprint's Post Mortem section. It's the *distilled* summary — three to ten bullets per sprint, written for someone who didn't ship the sprint and isn't going to read the sprint file.

Skeleton:

```markdown
# <Feature> — Post Mortems

> Part of the [<Feature> Implementation Plan](./overview.md).

## Sprint 01 — <title>

- <distilled observation, lesson, or follow-up>
- <distilled observation, lesson, or follow-up>
- …

## Sprint 02 — <title>

- <distilled observation, lesson, or follow-up>
- …
```

Rules:

- One section per shipped sprint, in sprint order. The heading matches the sprint file's title (`Sprint NN — <title>`).
- The file is created the first time a sprint ships. It does not exist during Round 1 scaffolding.
- A new sprint section is appended in the **same PR** that checks off that sprint's final step. Don't batch.
- The entry is a *distillation* of the sprint file's Post Mortem section, not a copy. Strip names, drop step-level minutiae, keep observations a future maintainer would want to know.
- If a sprint had nothing post-mortem-worthy, write `- _No post-mortem-worthy observations._` as the section's only bullet — don't omit the section. The presence of every shipped sprint as a heading is the index humans use to confirm coverage.
- Don't edit prior sprint sections retroactively. If a later sprint surfaces context that recolors an earlier sprint's post mortem, add it to the current sprint's section with a back-reference (`Re Sprint 03: …`).
- Don't add narrative connective tissue between sprint sections. `post-mortem.md` is a list of distilled per-sprint summaries, not an essay.

---

## Tone, voice, and conventions

- **Concrete and prescriptive.** Prefer "the manager INSERTs both participant rows in the same transaction" to "the manager persists membership atomically." Prefer "run `yarn db:generate patient`" to "regenerate the migration."
- **Justify every non-obvious decision.** "Architecture notes" subsections, inline rationale paragraphs, "Subtle bug" call-outs. The implementer should never wonder *why* the step says what it says.
- **Surface trade-offs explicitly when alternatives competed.** A `✅ / ⚠️` list captures them well:

```
- ✅ ON CONFLICT keeps the transaction usable on race recovery.
- ✅ Race recovery and idempotent rejoin share the same code path.
- ⚠️ Slightly more SQL than a naïve INSERT; acceptable given the race is real.
```

- **Be honest about v1 compromises.** Label them in place: `Tier 0 trade-off`, `defers to Sprint 11`, `acceptable at this scale; revisit if Z becomes hot`.
- **No marketing words.** Avoid "elegant," "clean," "robust," "powerful." If a step is good, the rationale should make that obvious.
- **Bold the decision lead.** Each Architecture note, each Decisions-log bullet, leads with a bold phrase that names the call. Skim-readability matters.
- **Identifiers in backticks.** `chatMessages`, `MEDIA_MAX_BYTES_IMAGE`, `'pending'`, `(post_id, direct_pair_key)`.
- **Wiki-style links** if the project uses them; otherwise relative markdown links. Always link the sprint back to `overview.md` and forward-link "Out of scope" entries to the sprint that owns them.
- **Active voice.** "The manager calls `social.canInvite` before BEGIN." Not "the canInvite check is performed."
- **Em-dashes for asides;** parentheses for quick clarifications. Don't mix.
- **Dates in absolute form** (`2026-05-08`), not relative (`Thursday`, `last week`). Relative dates rot.
- **Treat future-you as the primary reader.** A future engineer (or agent) picks up the sprint cold; the rationale paragraph that took 30 seconds to write saves them an hour of archaeology. Err on the side of explaining once too often.
- **Test cases are enumerated, not hand-waved.** "Cover the validation matrix" is not a test plan. "Inserts with `scope='post', target_id=NULL` raise sqlstate `23514` against constraint `media_assets_scope_post_target`" is.
- **Errors are thrown at one layer and mapped at one boundary.** Whatever the project's layering — domain → transport, core → adapter, model → controller, engine → CLI — keep error generation away from the boundary that produces user-facing codes. A sprint that ships a public surface documents the domain-error → user-facing-error mapping at the boundary step, not buried in the layer below. (In a backend project this is typically "manager throws domain errors; resource layer maps to HTTP.")
- **Be specific about names and shapes.** Real file paths, real type names, real constraint names, real env-var keys. Vagueness produces drift; drift produces re-litigation.

---

## Recurring constructs you should reach for

- **"Subtle bug" call-outs inside Actions.** Promote gotchas above prose. They earn their own bolded lead. Examples: cascade direction on FKs, partial-index predicate ordering, enum-vs-text comparison, AsyncLocalStorage tx propagation.
- **Code-shape samples.** Drizzle schema blocks, Zod schemas, type aliases, abstract class skeletons. Not full implementations — enough that a reader can pattern-match.

```ts
export const measurementTypes = schema.table('measurement_types', {
  id: uuid().primaryKey().defaultRandom(),
  canonicalName: text().notNull(),
  // …
});
```

- **Per-step Verification lists.** Concrete, not generic. "A trivial test: `measurementPayloadSchema.parse({ kind: 'lab_result', payload: { … } })` succeeds." "`psql -c "\d+ patient.measurements"` shows all 10 indexes."
- **Architecture-notes "Why X" paragraphs.** Almost every sprint earns at least one. They differentiate "this is the long-term answer" from "this is a deliberate stop-gap."
- **Tables for matrices.** When N entities behave differently along the same axis, tabulate. Error tables, push-eligibility tables, decision tables, allowlist tables.
- **ASCII dependency charts** for sprint and step graphs.
- **Worked JSONC examples** for request/response payloads, with comments describing nullability.
- **Contract amendment at the top of a sprint.** When a sprint takes a dependency on a future sprint that hasn't shipped (e.g., "Sprint 02 must accept caller-supplied IDs for Sprint 04's idempotency story to land"), capture the obligation at the top of *both* sprint files so neither side ships without the contract. Don't bury the dependency inside a step.
- **Footgun categories worth flagging** whenever the step touches them — the gotcha line is non-optional. The *categories* of footgun are project-specific; what makes one worth flagging is that an experienced engineer recognizes the shape but a junior implementer would not. Inherit the project's known-footgun set from existing plans, code-review history, post-mortems, and `CLAUDE.md`. A frontend project's gallery looks nothing like a database project's; the *posture* (call it out inline so the implementer doesn't rediscover it via a flaky test) translates regardless. Examples below are drawn from a backend / database shape — illustrative, not exhaustive:
  - **Indexes on the wrong column.** Indexing on `id` or PK silently makes a partial-UNIQUE constraint a no-op; partial uniqueness needs the discriminator column.
  - **Async work that reorders past a signing boundary.** SigV4 binds Host; rewriting the host after signing invalidates the signature. Same hazard for any signed-then-mutated payload.
  - **Constraints that need to be unconditional vs. status-gated.** A CHECK that should fire on every row is wrong if it's wrapped in `WHERE deleted_at IS NULL`; conversely, a partial UNIQUE on soft-deletable rows MUST exclude tombstones.
  - **In-process caches that won't pick up DB changes.** Per-process LRUs are fine for read-only reference data; they're bugs when the underlying row is mutable.
  - **Cross-tx vs. in-tx behavior of triggers.** `set_updated_at`'s `NOW()` returns the per-transaction snapshot — an in-tx INSERT + UPDATE shares one timestamp. Cross-tx tests need two separate connections.
  - **Catching `23505` vs. `ON CONFLICT DO NOTHING`.** A constraint violation aborts the transaction; recovery requires a fresh handle. `ON CONFLICT DO NOTHING RETURNING *` keeps the transaction usable and is the canonical race-recovery pattern.

  Equivalent gotchas in other stacks: stale closures in React effect hooks; signal-handler reentrancy in CLIs; layout-thrash from synchronous reads inside RAF; cache-key collisions from missing namespace prefixes; race conditions from background-task IDs that don't survive process restarts. Pull from the project, not from this list.

---

## Anti-patterns to avoid

- **Don't redesign the system inside an implementation plan.** If a step needs a decision the design plan didn't make, surface the question — don't bury the answer in a step.
- **Don't skip "why."** A step without rationale is half a step.
- **Don't write planning narrative inside a step.** Steps are executable. If a paragraph reads "we considered X but went with Y," it belongs in Architecture notes, not Actions.
- **Don't promise behavior the verification can't catch.** If "Verification: `yarn check:types` passes" is the only check on a step that ships behavior, the step is under-specified.
- **Don't leave dead text** when a round changes the design. Purge the obsolete wording from the body and let the Decisions/Status logs carry the audit trail.
- **Don't bury supersession.** If round N invalidates a sprint's earlier shape, the body must reflect the new design AND the Status log must call out the supersession explicitly.
- **Don't let `progress.md` and per-sprint Progress drift.** They are duplicates by design; if they disagree, the master `progress.md` wins, and the sprint copy is reconciled in the same round.
- **Don't conflate the design plan's Decisions log with the implementation plan's Decisions log.** The design plan logs what the system **is**; the implementation plan logs how and in what order it gets **built**. Cross-reference, don't merge.
- **Don't write build-spec specifics into the design plan**, and don't write design-level rationale into a sprint step. The two layers stay distinct on purpose.
- **Don't add file-path or function-name detail in a sprint that conflicts with the project's conventions** (e.g., kebab-case directories, repository placement rules). Sprints inherit the project's CLAUDE.md / conventions.md; restating with drift is worse than not restating.
- **Don't auto-resolve the user's open questions.** When a question requires a judgment call, surface alternatives and let the user choose. The agent's job is to frame the call sharply, not to make it.
- **Don't append `(resolved)` tags inside Open questions.** Move the entry to Decisions log.
- **Don't ship a sprint that leaves the codebase in a half-broken state.** If a sprint can't merge cleanly, it's a re-slice signal — split it differently or fold it into its dependent.
- **Don't slice steps that require uncompilable / untestable intermediate commits unless there is no viable alternative.** If unavoidable, the step owns an explicit Build/test boundary note with the expected failure, rationale, and repair step.
- **Don't pre-commit to step counts in Round 1.** Sprint stubs in Round 1 may not have steps yet. That's fine; they get fleshed out in subsequent rounds.
- **Don't write "implementation details TBD."** That's a planning failure dressed up as humility. Lock the decision, surface it as an Open Question with rationale, or name the future sprint that closes it.
- **Don't paper over conflicts** between the design plan and prior sprint decisions. Flag the conflict and resolve it before the sprint proceeds; silent reconciliation produces two sources of truth.
- **Don't write tests as an afterthought.** Each step's Verification block names specific assertions. A step whose Verification is "tests pass" is under-specified.
- **Don't paraphrase the design plan into one giant sprint.** Re-derive sprint structure from outcomes (what ships when), not from chapter headings of the design plan. A sprint roster that mirrors the design plan's section list isn't slicing — it's re-titling.
- **Don't bury the layer-vs-boundary error-mapping decision.** Domain errors live in the inner layer; user-facing-code mapping lives at the boundary. The boundary step documents the mapping table explicitly. (In a backend service: managers throw, resources map to HTTP. In a CLI: the engine throws, the command-handler maps to exit codes. Adapt.)
- **Don't ship a sprint's final step without distilling its Post Mortem into `post-mortem.md`.** The cross-sprint reading surface is load-bearing for future maintainers; skipping the distillation forces them to grep every sprint file. If the sprint genuinely has nothing post-mortem-worthy, append the heading with `_No post-mortem-worthy observations._` — silent omission is the failure mode, not a bare bullet.
- **Don't pre-create `post-mortem.md` at scaffold time.** The file appears the first time a sprint ships. An empty `post-mortem.md` in a Round 1 directory is noise.

---

## Driving a round with the user

### Continuing an existing implementation plan

1. **Read the entire directory, end to end.** `overview.md`, `decisions.md`, `status.md`, `progress.md`, every sprint file, and `post-mortem.md` if it exists. Identify Open questions (overview-level + sprint-level), recent `decisions.md` entries (plus sprint-scoped Decisions logs), and the most recent `status.md` round.
2. **Pick 1–3 questions or 1 sprint** to focus this round. Prefer questions whose resolutions are reasonably independent.
3. **For each, frame the choice** for the user: 2–3 viable options with trade-offs, your recommendation, and one sentence on what gets unblocked by the answer.
4. **After the user answers**, update the affected files:
   - Sprint body gets the new design (purge the old wording).
   - Decisions log gets a tagged bullet — cross-cutting calls in `decisions.md`; sprint-scoped calls in the affected sprint file's Decisions log section.
   - Open questions loses the resolved entry; gains any newly-surfaced sub-questions.
   - `status.md` gets a new numbered entry (plus, optionally, a sprint-scoped Status entry inside the affected sprint file).
   - `progress.md` is updated if step ordering or step count changed.
5. **Re-read the affected sprint(s)** to catch any wording from earlier rounds that the resolution silently invalidated. Cross-check against the overview's locked-decisions table — sprint-level decisions never override overview-level ones.

### Starting a new implementation plan

The bootstrap workflow — taking a finished design plan and producing the initial scaffold — is captured in [design-to-impl.md](./design-to-impl.md). That document is the agent-facing brief for Round 1 specifically.

### What to do when the user doesn't know

- **Surface the analogues.** Find similar implementation plans in the repo (or the precedents under `plans/`) and present how they resolved the same shape question. Borrow generously; explain what you're borrowing.
- **Default to v1 simplicity.** When the user is genuinely undecided, prefer the smallest answer that doesn't paint into a corner — and note the alternative as deferred.
- **Document the punt.** "Deferred — revisit when concrete demand emerges" is a legitimate resolution. Tag it in the appropriate Decisions log (`decisions.md` for cross-cutting; the sprint file's Decisions log for sprint-scoped) so a future round can reopen with proper context.

### When sprint slicing changes mid-stream

Re-slicing is normal. When it happens:

- Update `overview.md`'s sprint roster, dependency graph, and any rationale paragraphs that reference the old shape.
- Rename / split / fold sprint files. Renumber if the directory ordering would otherwise lie about execution order.
- Update `progress.md` to reflect the new structure.
- Add a Status entry **to `status.md`**: "Round 4: re-sliced 06 into 06-resolve-type and 07-deduplicate; re-numbered 07–10 → 08–11. R3's combined-worker decision superseded."
- Cross-references in adjacent sprints get updated in the same round.

---

## Round-end completeness assessment

After every round (including Round 1), the agent emits a completeness assessment to chat. This is **not** part of any file in the directory — it's a chat-only recommendation that helps the user decide whether to keep iterating, run an external review, or hand the plan off to an implementer.

The assessment is the agent's honest read on the plan's state. It is allowed to be opinionated; the user is allowed to disagree. Do not pad the "complete" verdict to be polite, and do not list every minor wording quirk to look thorough.

Implementation plans differ from design plans in two important ways for completeness:

1. **Implementation plans are a directory of files**, not a single doc. "Complete" requires every required file to be present and consistent with the others.
2. **Implementation plans are sequential by design**. Sprint 01 may be execution-ready while Sprint 06 is still a stub — that's normal and not a defect. The completeness assessment grades the directory's *current readiness for the next unshipped sprint*, not the readiness of every file equally.

### When an implementation plan is "complete enough"

There are three different "complete enough" thresholds; the assessment picks the most relevant one for the round just finished.

**Scaffold-complete** — Round 1 has just landed:

- `overview.md` is populated per the spec — title, framing, philosophy, architectural invariants, module/directory layout, cross-sprint conventions, sprint organization rationale, sprint roster (with one-line outcomes), dependency graph, feature-wide locked decisions table, out-of-scope across all sprints, open questions, what this plan does *not* try to do, how to read each sprint. **No Decisions log or Status section inside `overview.md`.**
- `decisions.md` exists at the top level of the plan directory — empty, or with `(R1)`-tagged entries capturing slicing decisions agreed in Round 1.
- `status.md` exists at the top level of the plan directory with a single entry: `**Round 1**: scaffolding + sprint roster + open questions enumerated. _Next:_ …`.
- One stub sprint file per roster entry, each with at least Goal, Prerequisites, Deliverables (best-effort first cut), Out of scope (forward-linked to owner sprints), and Open questions.
- `progress.md` has one section per sprint, no step bullets yet.
- No `post-mortem.md` (it's created lazily when the first sprint ships).
- The sprint roster slicing is agreed with the user.

**Sprint-NN-execution-ready** — the round just locked Sprint NN:

- Sprint NN's stub has graduated to a full sprint file: Locked Decisions table populated (10–25 rows; not a single row), Architecture notes ("Why X" subsections) for non-obvious shape decisions, Public surface sketched (when applicable) with request/response shapes and an error table, Progress checklist, Implementation Steps (5–10) each with Goal/Actions/Deliverables/Verification, Step Dependency Chart, Acceptance checklist.
- Every step is sliced as a compilable/testable boundary, or explicitly includes a Build/test boundary note explaining the unavoidable temporary breakage and the later step that restores it.
- Every step's Verification names specific assertions, commands, or test names — not "tests pass" / "compiles."
- `progress.md`'s Sprint NN section is in lockstep with the sprint file's Progress section.
- Sprint NN's Prerequisites all match a prior sprint's Deliverables (or a peer-service surface that exists, or an audit-or-build branch).
- Sprint NN's Out-of-scope items each forward-link to a sprint that owns them.
- Sprint NN's Open questions are empty *or* contain only items that don't block this sprint's execution.

**Plan-complete** — every sprint is execution-ready:

- Every sprint file has cleared the Sprint-NN-execution-ready bar.
- The dependency graph in `overview.md` agrees with every sprint's Prerequisites list.
- The cross-sprint coherence checks (overview locked decisions vs. sprint locked decisions; module-tree drift; convention drift; invariant honoring) all pass.
- `progress.md` reflects the full step structure.
- Open Questions at every scope are empty *or* contain only items explicitly deferred with rationale.
- `decisions.md` and `status.md` are coherent with every sprint file's sprint-scoped Decisions / Status sections — round numbering consistent, supersessions paired with body edits, no orphaned references.
- Tone and conventions clean — no marketing words, no design-level rationale leaking into sprints, no implementation-spec specifics leaking up to the overview.

If the round didn't aim at a specific sprint (e.g., it was a feedback-incorporation round across many files), grade against whichever threshold the directory's overall state best matches.

### What to emit after each round

The chat output uses this exact structure (one block, terse, no preamble):

```markdown
### Round <N> — completeness assessment

**Verdict**: <not-yet-complete | substantially-complete | complete>
**Threshold graded**: <scaffold-complete | sprint-NN-execution-ready | plan-complete>

**What's still load-bearing-open** *(if not complete; omit otherwise)*:
- <file or sprint + section + the gap>
- …

**Top nits worth resolving before declaring done** *(if substantially-complete or complete; omit otherwise)*:
- <file + section + the wording / consistency / coverage issue>
- …

**Recommended next-round focus**:
- <one-line recommendation; usually the load-bearing-open items, or "lock Sprint NN+1" if the current sprint is execution-ready, or "hand off to implementation" if plan-complete>
```

Verdict semantics (relative to the chosen threshold):

- **`not-yet-complete`**: At least one load-bearing item from the threshold's checklist is still open or undecided. The "What's still load-bearing-open" block enumerates the gaps with file + section pointers.
- **`substantially-complete`**: All load-bearing items at this threshold are decided, but rough edges remain — wording inconsistencies, a `progress.md` ↔ per-sprint Progress mismatch, a Decisions log missing a recent decision's tag, marketing language to purge, an Out-of-scope entry without a forward-link. The plan *could* graduate at this threshold; the user may want to clean up first. The "Top nits" block enumerates them.
- **`complete`**: Load-bearing items at this threshold are decided *and* the affected files are internally clean. Recommend the next move — usually locking the next sprint, running `impl-review` for an external check, or handing the plan off to an implementer if `plan-complete`.

Calibration:

- **Always name the threshold.** A round that locks Sprint 03 doesn't grade against `plan-complete` — it grades against `sprint-03-execution-ready`. Listing "Sprint 04–11 are still stubs" as "load-bearing-open" against a Sprint 03 round is noise; that's the next round's work.
- **Don't oscillate.** If round N said `substantially-complete` at the same threshold and round N+1 only changed a comment, the verdict shouldn't drop back to `not-yet-complete`.
- **Cross-sprint coherence is load-bearing at every threshold above scaffold-complete.** A Prerequisite that names a Deliverable spelled differently in a prior sprint is a load-bearing-open item, not a nit.
- **Don't pad either side.** A 3-bullet "load-bearing-open" list is a real list. A 12-bullet "top nits" list is a sign the plan is not actually substantially-complete — promote the most material items into "load-bearing-open" and re-grade.
- **Cite the file + section.** Every bullet names the file *and* the section so the user can navigate (`Sprint 04 § Locked Decisions`, `overview.md § Dependency graph`, `progress.md § Sprint 06`). Bare-section references are too ambiguous in a directory of files.
- **One block per round.** Don't emit multiple completeness assessments in the same round; the user reads the latest as authoritative.

This assessment is the agent's recommendation, not a gate. The user decides when to graduate. But the assessment forces the agent to take a position each round, which surfaces drift before it compounds across sprint files.

---

## How to use the plan to execute

A sprint is "execution-ready" when a junior engineer can:

1. Open `overview.md`, scan the philosophy + locked-decisions table + sprint roster (≤ 5 minutes).
2. Open the target sprint file, read Goal → Prerequisites → Deliverables → Out of scope → Locked decisions → Architecture notes (≤ 15 minutes).
3. Pick up Step 1 from the Implementation Steps section and execute it without consulting the design plan unless an explicit pointer says to.
4. After each step, mark its checkbox `[x]` in **both** the per-sprint Progress section **and** the master `progress.md`. (Or just the master if the per-sprint section has been removed for compactness — but typically both.)
5. Record any Deviations applied during implementation **in the same PR** that ships the divergence — top-of-doc list for cross-cutting deltas, inline `### Step N deviations` subsection for granular ones.
6. After the last step, walk the Acceptance checklist; only when every box is `[x]` is the sprint shippable.
7. In the same PR that checks off the final step, distill the sprint's Post Mortem section into a sprint-keyed entry in the directory's `post-mortem.md` (creating the file if it doesn't yet exist). If there's nothing post-mortem-worthy, the entry is a single `_No post-mortem-worthy observations._` bullet — don't omit the sprint heading.

If the engineer hits a question the sprint doc doesn't answer:

- **Easy answer**: it's in `overview.md`'s locked-decisions table, `decisions.md`, or the design plan. Resolve and continue.
- **Mid-difficulty**: it's a sprint-level call the planning rounds missed. Surface to the planning collaborator (the user / the senior engineer) to add a sprint-level locked decision. Don't make the call inside a step.
- **Hard answer**: it's a design-level call the design plan didn't make. Stop the sprint, log the question into the design plan's Open questions, escalate. Don't re-decide architecture inside an implementation plan.

---

## Skeleton templates

### `overview.md`

```markdown
# <Feature> — Implementation Plan

> Source design: [<design plan>](<path/to/design-plan.md>). Read end-to-end before picking up Sprint 01.

This plan breaks <feature> into <N> PR-sized sprints. Each sprint ships a coherent, independently-testable deliverable. Earlier sprints establish foundations; later sprints layer …

## Philosophy

### The plan is the source of truth
…

### Tier-0-aware, but never tier-0-locked
…

### Cross-service contract over internal cleverness
…

### Test the cross-service seams hardest
…

### Small, shippable sprints
…

## Architectural invariants the sprints uphold

1. <invariant — restated from the design plan; tests must verify when touched>
2. …

## Module / directory layout impact

```
src/services/<svc>/
├── …
```

## Cross-sprint conventions

- **Branching:** …
- **Migrations:** …
- **Seed data:** …
- **OpenAPI / spec sync:** …
- **Tests:** …
- **Logging:** …
- **Errors:** …
- **Idempotency:** …
- **Feature flags:** …

## Sprint organization

### Rationale for the sprint order

- **Sprints 01–02 — foundation.** …
- **Sprint 03 — …** …
- …

### Rationale for sprint-sized slices

A sprint is scoped to roughly one to two weeks of focused work by a single engineer. Each:
- Targets one architectural seam or one user-visible capability.
- Has a clear "done" line with acceptance criteria.
- Leaves the repo buildable, testable, and demonstrable.
- Can be reviewed as a single PR.

The slices are not equal in size. They are equal in shippability.

## Sprint roster

| #  | Sprint                                       | Outcome                                                  |
| -- | -------------------------------------------- | -------------------------------------------------------- |
| 01 | [<title>](./01-<topic>.md)                   | …                                                        |
| 02 | [<title>](./02-<topic>.md)                   | …                                                        |
| …  | …                                            | …                                                        |

## Dependency graph

```
01 ──► 02 ──► 03 ──► …
```

- **01 → 02**: <why>
- **02 → 03**: <why>
- …

## Feature-wide locked decisions

| Decision   | Value |
|------------|-------|
| …          | …     |

## Out-of-scope across all sprints

- …

## Open questions

1. **<title>.** <question + state of debate + indicative direction or "deferred until X.">
2. …

<!--
  No Decisions log or Status section here.
  Plan-level decisions live in ./decisions.md.
  Plan-level round-by-round audit trail lives in ./status.md.
-->

## What this plan does *not* try to do

- …

## How to read each sprint

Every sprint follows the same skeleton: Goal → Prerequisites → Deliverables → Out of scope → Locked decisions → Architecture notes → API surface (when applicable) → Progress → Implementation Steps with verification → Step Dependency Chart → Acceptance checklist. Implementers should be able to pick up a sprint and execute it without re-reading the design doc, though the design doc is the source of truth for any conflict.
```

### `NN-<topic>.md`

```markdown
# Sprint NN — <Short title>

> Part of the broader [<Feature> Implementation Plan](./overview.md).

## Goal

<one sentence — outcome, not activity>

## Prerequisites

- <prior sprint or peer-service surface>
- …

## Deliverables at end of sprint

- <observable artifact>
- …

## Out of scope

- <forward-link to the sprint that owns it>
- …

## Key design decisions, locked before implementation

| Decision                        | Value |
|---------------------------------|-------|
| …                               | …     |

## Architecture notes

- **Why <choice>.** …
- **Why <other choice>.** …

## API surface

### `<METHOD> <path>`

#### Request

```jsonc
{ … }
```

#### Response <code>

```jsonc
{ … }
```

#### Errors

| Status | Code | Cause |
|---|---|---|
| … | … | … |

---

## Progress

- [ ] Step 1 — …
- [ ] Step 2 — …
- …

---

# Implementation Steps

## Step 1 — <title>

**Goal**: <one sentence>

**Build/test boundary**: <omit unless this step may temporarily leave the repo uncompilable / untestable; if needed, explain why unavoidable and which later step restores the gates>

**Actions**:

1. <concrete action with file paths and code samples>
2. …
3. **Subtle bug**: <gotcha spelled out>

**Deliverables**:
- …

**Verification**:
- …

## Step 2 — <title>

…

---

## Step Dependency Chart

```
1 ──► 2 ──► …
```

- <one-liner per parallelizable / hard-sequenced edge>

## Acceptance checklist for the sprint as a whole

- [ ] …
- [ ] …
```

### `progress.md`

```markdown
# <Feature> — Progress

> Part of the [<Feature> Implementation Plan](./overview.md).

## Sprint 01 — <title>

- [ ] Step 1 — …
- [ ] Step 2 — …
- …

## Sprint 02 — <title>

- [ ] Step 1 — …
- …
```

### `decisions.md`

```markdown
# <Feature> — Decisions

> Part of the [<Feature> Implementation Plan](./overview.md).
>
> Cross-cutting decisions only. Sprint-scoped decisions live in the relevant sprint file's Decisions log section.

- **<Decision lead in bold>.** <Brief restatement / rationale.> (R<n>)
- **<Decision lead in bold>.** <…> (R<n>)
- …
```

### `status.md`

```markdown
# <Feature> — Status

> Part of the [<Feature> Implementation Plan](./overview.md).
>
> Plan-level round-by-round audit trail. Sprint-scoped Status / Feedback-incorporated entries live in the relevant sprint file when a round materially changes a single sprint.

- **Round 1**: scaffolding + sprint roster + open questions enumerated. _Next:_ <one-line recommended focus, citing Open question IDs + tags + scope where relevant>.
- **Round 2**: … _Next:_ <…>.
- …
```

### `post-mortem.md`

Created lazily — the first sprint to ship creates this file in the same PR that checks off its final step. Each subsequent shipped sprint appends a new section.

```markdown
# <Feature> — Post Mortems

> Part of the [<Feature> Implementation Plan](./overview.md).

## Sprint 01 — <title>

- <distilled observation, lesson, or follow-up>
- <distilled observation, lesson, or follow-up>
- …

## Sprint 02 — <title>

- _No post-mortem-worthy observations._
```

---

## Quick reference: when in doubt

- **Add an "Architecture note"** if a reader could plausibly ask "why this shape?"
- **Promote a "Subtle bug:" line** above prose whenever an implementer would otherwise stumble.
- **Move an Open question to the Decisions log** the moment it resolves; never mark "(resolved)" in place. Cross-cutting calls land in `decisions.md`; sprint-scoped calls land in the affected sprint file's Decisions log section.
- **Tag every Decisions-log bullet with `(R<n>)`** so the audit trail is intact.
- **Update `status.md` every round**, even if the round was small. When a single sprint was materially changed, also append a sprint-scoped Status entry to that sprint file.
- **Compress older log entries** that no longer carry weight — superseded Decisions-log bullets keep their tag and supersession pointer but drop the rationale paragraph; ancient Status rounds whose `_Next:_` is long-finished collapse to a one-line summary. Keep the last 2–3 rounds and any still-load-bearing entry in full. Applies to `decisions.md`, `status.md`, and every sprint file's local logs alike. See "`decisions.md` anatomy" and "`status.md` anatomy."
- **Update `progress.md` whenever step structure changes** — a re-sliced sprint regenerates the master checklist in the same round.
- **Distill the sprint's Post Mortem into `post-mortem.md`** in the same PR that checks off the sprint's final step — creating the file if it doesn't yet exist. An empty entry (`_No post-mortem-worthy observations._`) is fine; a missing entry is not.
- **Purge stale wording** when a round supersedes it; trust the Decisions/Status logs to carry the audit trail.
- **Prefer concrete over abstract**: an example JSON, an actual SQL fragment, a typed interface sketch — over prose paraphrase.
- **Surface alternatives, don't decide unilaterally** when the user has skin in the game.
- **When the design plan didn't decide it, escalate** — don't pin the call inside a step.
- **The sprint doc is the executable artifact; the design plan is the tiebreaker.** Keep them aligned but distinct.
