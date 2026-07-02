# Implementation Plan Documents — Authoring Guide

Instructions for an LLM agent on producing **implementation plan documents**. This guide is the agent-facing brief for collaborating with a human user to produce one of these document sets from scratch — or to drive an existing one forward through another planning round.

The companion document, [design-plan.md](./design-plan.md), describes how upstream **design plans** are produced. An implementation plan is the next layer down: it consumes a finished (or in-progress) design plan and turns its decisions into concrete, sequenced engineering work.

This guide is one of three implementation-plan spec files — load only what your operation needs (SKILL.md's load map says which):

- **This file** — document anatomy and authoring conventions (root layout, per-file anatomies, tone, recurring constructs, anti-patterns).
- **[implementation-plan-mechanics.md](./implementation-plan-mechanics.md)** — round mechanics: the iteration model, supersession, driving a round, the round-end completeness assessment, the multi-file edit discipline, the cross-sprint coherence checklist, the around-corner concern checklist, and the execution handoff.
- **[implementation-plan-templates.md](./implementation-plan-templates.md)** — skeleton templates for every file, used at scaffold time and when creating a new sprint file mid-plan.

---

## What these documents are

An **implementation plan** is a set of markdown files in the feature root describing how a design plan will actually be built. It is the substrate a junior engineer (or an agent) consumes to ship the feature with minimal intervention — every architectural decision should already be made, every non-obvious pattern should already be explained, and every gotcha should already be flagged.

The motto: **optimize for so much rigor up front that implementation has nothing left to debate.** When a step requires the implementer to make a decision the doc didn't make, that's a planning failure, not a feature. Lock the decision, surface it as an Open Question, or name the future sprint that closes it — never punt the call into the code.

An implementation plan IS:
- A set of files directly in the feature root (not a single document) — one **overview**, N **sprint plans**, and a master **progress checklist**.
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

The structural advice in this guide (Goal → Prerequisites → Deliverables → Locked Decisions → Architecture notes → Steps → Verification; per-sprint Progress mirrored in the feature root's `progress.md`; living-during-execution Deviations log) holds in every case. The *content* of those sections must reflect the actual project the agent is working in.

If the project lacks an obvious analogue for a section the spec describes (e.g., no public API surface; no schema; no async workers), drop the section. If the project has surfaces this spec doesn't enumerate, add them. **Adapt to the codebase. Don't bend the codebase to fit the examples in this spec.**

---

## How the document set is produced

Round mechanics — the iteration model, Round 1 scaffolding, subsequent rounds, supersession, deviations applied during execution, and honest framing for v1 compromises — live in the companion file [implementation-plan-mechanics.md](./implementation-plan-mechanics.md). Load it for any operation that runs a planning round.

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

**During iteration.** Re-slicing is normal once sprint-level work begins. When a round surfaces that a sprint is too large, too small, or sequenced wrong, apply these heuristics and follow the supersession + `progress.md` regeneration steps in [implementation-plan-mechanics.md § "How the document set is produced"](./implementation-plan-mechanics.md).

---

## Root layout

The implementation plan lives directly in the feature root, alongside the source design plan:

```
<root>/
├── design.md              source-of-truth design plan
├── overview.md
├── decisions.md
├── status.md
├── progress.md
├── post-mortem.md         # created lazily; first appears when the first sprint ships
├── design-review-R<N>.md  # review artifacts (design-review / impl-review outputs);
├── impl-review-R<N>.md    #   working records, not plan files
├── reviews/               # per-step execution records, created during impl-execute
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
- Sprint files are zero-padded numerically prefixed (`01-`, `02-`, …) so filename order matches execution order. Re-numbering on a re-slice is fine — **except for shipped sprints, which are frozen** (see [implementation-plan-mechanics.md § "When sprint slicing changes mid-stream"](./implementation-plan-mechanics.md)).
- Sprint file names are `<NN>-<short-kebab-topic>.md` — short enough that the index table reads cleanly, descriptive enough that a `git log` line is meaningful.
- One file per sprint. Don't split a sprint across two files; if a sprint is too large to fit comfortably in one file, that's a re-slice signal.
- **The overview file is `overview.md` — it is not numbered, not prefixed, not a sprint.** Only sprint files take the `NN-` prefix. Numbered prefixes on `overview.md`, `decisions.md`, `status.md`, `progress.md`, or `post-mortem.md` are wrong.
- **Review artifacts are working records, not plan files.** `design-review-R<N>.md` / `impl-review-R<N>.md` (written by the review operations) and the `reviews/` directory (per-step execution records written during `impl-execute`) live in the feature root but sit outside the plan file set: planning rounds never edit them, re-slices never renumber them, and the cross-sprint coherence checks don't apply to them.

---

## `overview.md` anatomy

Drop sections that don't apply. Add domain-specific sections where the work demands them. The list below is the canonical superset.

### 1. Title & framing block

```
# <Feature> — Implementation Plan
```

Followed by 1–3 sentences saying what this plan contains and a clear link back to the source design plan. Because `overview.md` and `design.md` both live directly in the feature root, that link is `./design.md`:

> Implementation plan for the [Lab Results & Measurements design plan](./design.md). Read that document end-to-end before picking up Sprint 01 — every sprint assumes its vocabulary.

The link back to the design plan is **not optional**. The design plan is the upstream source of truth; an implementation plan that doesn't cite it is a red flag.

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

> **`overview.md` does not contain a Decisions log or a Status section.** Those live directly in the feature root as `decisions.md` and `status.md` — see the next two sections for their anatomy.

---

## `decisions.md` anatomy

`decisions.md` is the plan-level (cross-cutting) Decisions log. It lives directly in the feature root — **not** as a section inside `overview.md`. A returning collaborator opens `decisions.md` to recover "what got decided across the whole feature, and when." Sprint-scoped Decisions logs continue to live inside their sprint files; `decisions.md` holds only cross-cutting calls (the calls every sprint would otherwise re-litigate).

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

`status.md` is the plan-level round-by-round audit trail — the doc's "git log." It lives directly in the feature root — **not** as a section inside `overview.md`. A user resuming the plan via `impl-iterate` reads the latest entry's `_Next:_` clause to recover where to pick up. Sprint files may additionally carry a sprint-scoped Status / Feedback-incorporated section for round entries that materially changed just that sprint; `status.md` holds the plan-wide narrative.

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

Every sprint file follows the same skeleton. Drop sections that don't apply (a sprint with no external surface has no Public surface section); add domain-specific sections where the work demands them.

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

**Distillation into `post-mortem.md`.** When the sprint's final step is checked off (i.e., the sprint ships), distill this section into a sprint-keyed entry in the feature root's `post-mortem.md`. Create the file if it doesn't yet exist; otherwise append a new section for this sprint. The inline Post Mortem section in the sprint file remains the authoritative long-form record; `post-mortem.md` is the cross-sprint reading surface so reviewers can scan all post mortems in one place without opening every sprint file. The distillation lands in the same PR that ships the sprint's final step. See "`post-mortem.md` anatomy" below.

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
- **Don't pre-create `post-mortem.md` at scaffold time.** The file appears the first time a sprint ships. An empty `post-mortem.md` in Round 1 is noise.

---

## Round mechanics, completeness, and cross-file checks

The following live in the companion file [implementation-plan-mechanics.md](./implementation-plan-mechanics.md): driving a round with the user (including re-slicing mid-stream), the round-end completeness assessment (`scaffold-complete` / `sprint-NN-execution-ready` / `plan-complete`), the multi-file edit discipline, the cross-sprint coherence checklist, the around-corner concern checklist, and how an implementer uses the plan to execute.

Skeleton templates for every file (`overview.md`, sprint files, `progress.md`, `decisions.md`, `status.md`, `post-mortem.md`) live in [implementation-plan-templates.md](./implementation-plan-templates.md).

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
