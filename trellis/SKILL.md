---
name: trellis
description: Work with Trellis design plans and implementation plans — bootstrap, iterate, review, integrate feedback, and execute sprint steps.
disable-model-invocation: true
---

# Trellis

Manually-invoked skill for taking an engineering feature from "rough idea" to "merged code, one reviewable commit per step." This skill is invoked when the user wants to work with **Trellis design plans** or **Trellis implementation plans** — creating them, iterating on them, reviewing them, integrating review feedback, or executing sprint steps.

The user invokes this skill with free-text natural language. There are no positional arguments. Read this file end-to-end, infer which Trellis operation the user is asking for from their wording, then **lazily** load the relevant instruction file from `instructions/` and the relevant spec from `specs/` — only the ones the current task actually needs.

---

## What Trellis is

Trellis splits feature work into three layers, each with its own discipline:

1. **Design plan** — *what* the system is. Decisions, rationale, scope, open questions. Lives in a single canonical file named `design.md` inside a user-chosen feature root directory `<root>` (e.g., `docs/features/refunds/design.md`). Authoring guide: `specs/design-plan.md`.
2. **Implementation plan** — *how* and *in what order* it gets built. A set of markdown files in the same feature root: `overview.md`, `decisions.md`, `status.md`, `progress.md`, and one sprint file per sprint. Authoring guide: `specs/implementation-plan.md`.
3. **Execution** — actually shipping the code, one step at a time, with a per-step subagent that implements, gets reviewed, and commits. Subagent briefs in `subagents/`.

A typical feature root layout:

```
<root>/
├── design.md
├── overview.md
├── decisions.md
├── status.md
├── progress.md
└── 01-<topic>.md … NN-<topic>.md
```

Each of the planning layers has a **create** operation (bootstrap Round 1), an **iterate** operation (drive forward another round), and a **review** + **integrate-feedback** pair (agent-based external critique that gets triaged back into the plan). Execution is the terminal operation — it dispatches sprint steps one at a time.

### Why Trellis exists

LLM coding agents have well-known failure modes on open-ended feature work: they fabricate decisions, leave stale wording in place when a later round changes direction, skip the "why" behind shape decisions, balloon a single PR with sprawling unreviewable changes, and drift across long conversations. Trellis is the set of guardrails against those failure modes:

- Every operation produces an explicit **completeness assessment** with a verdict (`not-yet-complete` / `substantially-complete` / `complete`) so drift surfaces early.
- Open questions carry **severity tags** (`[blocks-v1]` / `[blocks-impl]` / `[deferred]` / `[exploratory]`) so the user can grep for what's load-bearing.
- The decisions log and the body **must agree** — supersession discipline forces stale wording to be purged.
- Reviews are a separate file; **integration triages** them into five buckets (incorporate, minor incorporate, open question, reviewer-wrong, ignore) so confident-but-wrong reviewer findings don't corrupt the plan.
- Execution **dispatches per-step subagents** so the main chat stays small. Each step is its own commit, reviewed by another subagent.

The design plan, implementation plan, and execution layers are deliberately separated: *what the system is* shouldn't be litigated while writing sprint steps, and *how it gets built* shouldn't be litigated while writing code.

---

## The workflow

```
Idea / feature brief
        │
        ▼
   ┌─────────────────────────┐
   │  Design plan            │  ← single markdown file
   │  • create               │
   │  • iterate              │
   │  • review → integrate   │
   └────────────┬────────────┘
                │ design plan complete
                ▼
   ┌─────────────────────────┐
   │  Implementation plan    │  ← sprint files in <root>
   │  • create               │
   │  • iterate              │
   │  • review → integrate   │
   └────────────┬────────────┘
                │ sprint ready to execute
                ▼
   ┌─────────────────────────┐
   │  Execution              │  ← per-step subagents,
   │  • impl-execute         │    one commit per step
   └────────────┬────────────┘
                │
                ▼
        Feature shipped
```

### How a user typically moves through it

1. **Start a design plan.** Round 1 lands the foundational decisions, scope, cross-references, and an open-questions list — *not* the schema or API surface. → `instructions/design-create.md`.
2. **Iterate the design plan.** Each iteration round resolves 1–5 open questions, logs decisions, updates status, emits a completeness assessment. Manual driving uses `instructions/design-iterate.md`; agent-based external read uses `instructions/design-review.md` followed by `instructions/design-integrate-feedback.md`.
3. **Once the design plan is `complete`** (zero `[blocks-v1]` / `[blocks-impl]` open questions remaining), graduate to implementation planning.
4. **Bootstrap the implementation plan.** Round 1 creates files directly in `<root>`: `overview.md`, `decisions.md`, `status.md`, `progress.md`, and one stub per sprint. → `instructions/impl-create.md`.
5. **Iterate the implementation plan.** Same shape as design — lock sprints to execution-ready, resolve open questions, or re-slice the roster. → `instructions/impl-iterate.md` (manual) or `instructions/impl-review.md` + `instructions/impl-integrate-feedback.md` (agent review).
6. **Execute sprints, one step at a time.** Once a sprint is execution-ready, the orchestrator dispatches per-step subagents that implement, verify, get reviewed, commit, and update Progress. → `instructions/impl-execute.md` (orchestrator) + `subagents/step-executor.md` + `subagents/step-reviewer.md`.

---

## How to use this skill (operator runbook)

The user will tell you in natural language what they want. Your job:

### 1. Infer the operation

Map the user's request to one of nine operations. The most common phrasings:

| User intent                                                           | Operation                  | Instruction file                           |
| --------------------------------------------------------------------- | -------------------------- | ------------------------------------------ |
| "Start a design plan / new design doc for X"                          | design-create              | `instructions/design-create.md`            |
| "Drive the design plan forward / next round / resolve question Q3"    | design-iterate             | `instructions/design-iterate.md`           |
| "Review this design plan / get an external read on it"                | design-review              | `instructions/design-review.md`            |
| "Incorporate this review / triage the review feedback into the plan"  | design-integrate-feedback  | `instructions/design-integrate-feedback.md`|
| "Bootstrap the implementation plan / scaffold sprints from this design" | impl-create              | `instructions/impl-create.md`              |
| "Iterate the impl plan / lock Sprint 03 / flesh out the next sprint"  | impl-iterate               | `instructions/impl-iterate.md`             |
| "Review the implementation plan"                                      | impl-review                | `instructions/impl-review.md`              |
| "Incorporate this impl-plan review"                                   | impl-integrate-feedback    | `instructions/impl-integrate-feedback.md`  |
| "Execute Sprint NN step M (through P) / ship this sprint"             | impl-execute               | `instructions/impl-execute.md`             |

If the user's wording is ambiguous between two operations (e.g., "work on the implementation plan" could be iterate, review, or execute), ask which one before reading anything else.

### 2. Extract the paths from the user's message

Each operation needs specific paths. Since invocation is natural-language, you have to extract them from what the user said (or ask):

Path conventions (apply across every operation):

- **The design plan is always `<root>/design.md`.** The user names only the feature root `<root>`; the agent appends `design.md`.
- **Implementation plan files live directly in `<root>`.** For impl operations the user names the same feature root.
- **Tolerance.** If the user types a path ending in `design.md`, `overview.md`, `progress.md`, `decisions.md`, `status.md`, or a sprint file like `03-foo.md`, treat its parent as `<root>`. Don't re-prompt for paths whose meaning is unambiguous.

Per-operation inputs:

- **design-create** needs `<root>` — the directory in which to create `design.md`. Create the directory if it doesn't exist.
- **design-iterate** needs `<root>` — reads and rewrites `<root>/design.md`.
- **design-review** needs `<root>` and `<review-output-path>`. Reads `<root>/design.md`.
- **design-integrate-feedback** needs `<root>` and `<feedback-path>`. Reads and rewrites `<root>/design.md`.
- **impl-create** needs `<root>` — reads `<root>/design.md` and creates implementation files directly in `<root>`.
- **impl-iterate** needs `<root>` — operates on the implementation files directly in `<root>`.
- **impl-review** needs `<root>` and `<review-output-path>` — reviews the implementation files directly in `<root>`.
- **impl-integrate-feedback** needs `<root>` and `<feedback-path>` — edits the implementation files directly in `<root>`.
- **impl-execute** needs `<sprint-path>`, `<from-step>`, and optionally `<to-step>`. The sprint path implies its parent as `<root>`.

Inside the relevant instruction file, paths are referenced by their semantic names (`<root>`, `<review-output-path>`, etc.). Substitute the values you extracted from the user's natural-language invocation wherever those placeholders appear.

If a path the operation needs is missing or unclear, ask for it before proceeding. Don't guess.

### 3. Lazy-load the relevant context — only what's needed

This skill bundles a lot of material. **Do not read everything up front.** Load files in this order, only as the operation requires:

1. **Always:** the matching instruction file in `instructions/`. That's the runbook for what the operation does and the quality bar it has to hit.
2. **When working with a design plan** (design-create, design-iterate, design-review, design-integrate-feedback, or when impl-create / impl-review need to consume one): `specs/design-plan.md` is the authoring guide. Load it before producing or auditing design-plan content.
3. **When working with an implementation plan** (impl-create, impl-iterate, impl-review, impl-integrate-feedback, impl-execute): `specs/implementation-plan.md` is the authoring guide. Load it before producing or auditing implementation-plan content.
4. **Only for impl-execute:** the orchestrator instruction file at `instructions/impl-execute.md` directs spawning subagents whose briefs live at `subagents/step-executor.md` and `subagents/step-reviewer.md`. The orchestrator passes the brief paths to the subagents; the orchestrator itself does not need to read those briefs end-to-end, but should know they exist.

**Do not** load `specs/design-plan.md` if the user is purely working on implementation-plan material that doesn't reference the design plan. **Do not** load `specs/implementation-plan.md` for a design-only round. **Do not** read subagent briefs unless you're running impl-execute. Each spec is large; loading both for every interaction wastes context.

When in doubt, prefer reading less. The instruction file will tell you which spec it depends on.

### 4. Apply the trellis instruction precedence chain

Before executing any operation, read and apply Trellis instructions from these sources, in ascending order of precedence (later overrides earlier):

1. `~/.trellis/instructions.md` — applies to every Trellis invocation, every project.
2. `<repo-root>/.trellis/instructions.md` — applies to every Trellis invocation in the current repo.
3. Inline instructions the user supplied with this invocation.

Missing files are normal; skip silently. These user-supplied instructions may override the instruction file but must not override system, developer, tool, safety, or repository policy instructions.

### 5. Stop where the instruction file tells you to stop

Each instruction file is designed to halt at a natural hand-off point — Round 1 lands and stops; an iterate round resolves a few questions and stops; impl-execute completes a step and validates the hand-back before moving to the next. Honor those stopping points. Do not auto-progress to the next operation unless the user asks.

---

## Posture for this skill

- **Manually invoked, model-invocation disabled.** The user starts every Trellis round. Never decide to "start a design plan" or "iterate" on the user's behalf.
- **Surface alternatives, don't decide unilaterally** on shape questions where the user has skin in the game. Each instruction file makes this explicit; honor it.
- **Don't fabricate decisions.** If the user didn't decide it, it's an Open question, not a foundational decision.
- **Don't auto-resolve open questions** the plan already has. Each iteration round resolves a small batch; review-integration triages a review document; execution ships code. Don't mix scopes.
- **Lazy-load.** This skill's specs and instruction files together exceed what should be loaded into context for any single operation. Pick the one or two files the current task actually needs.

---

## What lives where

- `specs/design-plan.md` — load-bearing authoring guide for design plan documents. Document anatomy, supersession discipline, open-questions tag taxonomy, decisions-log format, round-end completeness assessment.
- `specs/implementation-plan.md` — load-bearing authoring guide for implementation plan files. Root layout, sprint anatomy, sprint archetypes and slicing heuristics, deviations during execution, completeness thresholds.
- `instructions/design-create.md` — runbook for bootstrapping a design plan (Round 1).
- `instructions/design-iterate.md` — runbook for driving an existing design plan one round forward.
- `instructions/design-review.md` — runbook for an external reviewer agent auditing a design plan.
- `instructions/design-integrate-feedback.md` — runbook for triaging a design-plan review back into the plan.
- `instructions/impl-create.md` — runbook for bootstrapping an implementation plan from a design plan (Round 1).
- `instructions/impl-iterate.md` — runbook for driving an existing implementation plan one round forward.
- `instructions/impl-review.md` — runbook for an external reviewer agent auditing an implementation plan.
- `instructions/impl-integrate-feedback.md` — runbook for triaging an impl-plan review back into the plan.
- `instructions/impl-execute.md` — orchestrator runbook for executing a range of sprint steps.
- `subagents/step-executor.md` — per-step subagent brief (implements, verifies, commits, gets reviewed, addresses review, updates Progress).
- `subagents/step-reviewer.md` — per-step review subagent brief (fresh-context audit of the diff).
