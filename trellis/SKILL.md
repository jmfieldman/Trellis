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
├── post-mortem.md            # created lazily when the first sprint ships
├── design-review-R<N>.md     # review artifacts (design-review / impl-review outputs);
├── impl-review-R<N>.md       #   working records, not plan files
├── reviews/                  # per-step execution records, created during impl-execute
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

Map the user's request to one of ten operations — nine layer-specific ones plus one cross-cutting accelerator (`resolve-open-questions`). The most common phrasings:

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
| "Execute Sprint NN step M (through P) / execute the rest of the sprint / ship this sprint" | impl-execute | `instructions/impl-execute.md`             |
| "Resolve the open questions in `<file>` / answer the open questions for Sprint 03 / draft answers to this file's open questions" | resolve-open-questions | `instructions/resolve-open-questions.md`   |

If the user's wording is ambiguous between two operations (e.g., "work on the implementation plan" could be iterate, review, or execute), ask which one before reading anything else. The closest pair to disambiguate is `*-iterate` vs. `resolve-open-questions`: an iterate round is **you** framing 1–5 questions and the user deciding each live; `resolve-open-questions` **fans out a subagent per question** to investigate and propose answers, which the user then approves or tunes in a batch. When the user says "answer / resolve / draft answers to the open questions in `<one file>`," that's `resolve-open-questions`; when they say "drive the plan forward" or "let's work through Q3 together," that's iterate.

### 2. Extract the paths from the user's message

Each operation needs specific paths. Since invocation is natural-language, you have to extract them from what the user said (or ask):

Path conventions (apply across every operation):

- **The design plan is always `<root>/design.md`.** The user names only the feature root `<root>`; the agent appends `design.md`.
- **Implementation plan files live directly in `<root>`.** For impl operations the user names the same feature root.
- **Tolerance.** If the user types a path ending in `design.md`, `overview.md`, `progress.md`, `decisions.md`, `status.md`, a sprint file like `03-foo.md`, or a review artifact like `impl-review-R5.md`, treat its parent as `<root>`. Don't re-prompt for paths whose meaning is unambiguous.

Per-operation inputs:

- **design-create** needs `<root>` — the directory in which to create `design.md`. Create the directory if it doesn't exist.
- **design-iterate** needs `<root>` — reads and rewrites `<root>/design.md`.
- **design-review** needs `<root>`; `<review-output-path>` is optional (defaults to `<root>/design-review-R<N>.md`). Dispatches a fresh-context reviewer subagent against `<root>/design.md`.
- **design-integrate-feedback** needs `<root>` and `<feedback-path>`. Reads and rewrites `<root>/design.md`.
- **impl-create** needs `<root>` — reads `<root>/design.md` and creates implementation files directly in `<root>`.
- **impl-iterate** needs `<root>` — operates on the implementation files directly in `<root>`.
- **impl-review** needs `<root>`; `<review-output-path>` is optional (defaults to `<root>/impl-review-R<N>.md`). Dispatches a fresh-context reviewer subagent against the implementation files directly in `<root>`.
- **impl-integrate-feedback** needs `<root>` and `<feedback-path>` — edits the implementation files directly in `<root>`.
- **impl-execute** needs `<sprint-path>`; `<from-step>` and `<to-step>` are optional. With neither ("execute the sprint" / "execute the rest of Sprint 03"), the range defaults to the remaining unchecked steps; with only `<from-step>`, a single step. The sprint path implies its parent as `<root>`.
- **resolve-open-questions** needs `<trellis-file-path>` — a single file with an Open Questions section (`design.md`, `overview.md`, or a sprint file `NN-*.md`). Its parent is `<root>`. By default it includes blocking questions (`[blocks-v1]` / `[blocks-impl]`) **plus any untagged Open questions**; untagged questions are a process bug, but they stay in scope unless the user explicitly excludes them. Optional inline instructions: "all questions" (also include `[deferred]` / `[exploratory]`) and "then write the steps" (decompose into steps after integrating — sprint files only). It operates on **one file per invocation**; never accept "resolve all open questions across the plan."

Inside the relevant instruction file, paths are referenced by their semantic names (`<root>`, `<review-output-path>`, etc.). Substitute the values you extracted from the user's natural-language invocation wherever those placeholders appear.

If a path the operation needs is missing or unclear, ask for it before proceeding. Don't guess.

### 3. Lazy-load the relevant context — only what's needed

This skill bundles a lot of material. **Do not read everything up front.** Load files per this map, only as the operation requires:

1. **Always:** the matching instruction file in `instructions/`. That's the runbook for what the operation does and the quality bar it has to hit.
2. **Design-plan work** (design-create, design-iterate, design-integrate-feedback, or when impl-create needs to consume a design plan): `specs/design-plan.md` is the authoring guide. Load it before producing or auditing design-plan content.
3. **Implementation-plan work** — the impl spec is split in three; load only the parts the operation needs:
   - `specs/implementation-plan.md` — anatomy + authoring conventions. Load for impl-create, impl-iterate, impl-integrate-feedback, and resolve-open-questions integration on an impl file.
   - `specs/implementation-plan-mechanics.md` — round mechanics, completeness assessment, multi-file edit discipline, cross-sprint coherence + around-corner checklists. Load for impl-iterate, impl-integrate-feedback, and resolve-open-questions integration on an impl file; impl-create needs only its "Round 1" and "Round-end completeness assessment" sections.
   - `specs/implementation-plan-templates.md` — skeleton templates. Load for impl-create; for impl-iterate only when the round creates a new sprint file.
4. **Feedback integration** (design-integrate-feedback, impl-integrate-feedback): `specs/review-and-triage.md` **Part II** is the shared triage machinery — load it alongside the layer's spec files above (Part I is for reviewers; skip it).
5. **Reviews** (design-review, impl-review): the orchestrator loads almost nothing — it dispatches a subagent whose brief lives at `subagents/design-plan-reviewer.md` / `subagents/impl-plan-reviewer.md`. Pass the brief path; do not read the brief end-to-end. The subagent itself loads `specs/review-and-triage.md` Part I plus the layer's spec files.
6. **Only for impl-execute:** the orchestrator instruction file at `instructions/impl-execute.md` directs spawning subagents whose briefs live at `subagents/step-executor.md` and `subagents/step-reviewer.md`. The orchestrator passes the brief paths and needs no spec files at all; the subagents load what their briefs name.
7. **Only for resolve-open-questions:** the orchestrator at `instructions/resolve-open-questions.md` spawns subagents whose briefs live at `subagents/question-resolver.md` and `subagents/question-reviewer.md` (the resolver spawns the reviewer). The orchestrator passes the brief paths; it does not read them end-to-end. At *integration* time it loads the spec matching the target file's doc type — `specs/design-plan.md` for `design.md`; `specs/implementation-plan.md` + `specs/implementation-plan-mechanics.md` for `overview.md` / a sprint file — to apply standard round mechanics (move to Decisions log, append one Status entry, completeness assessment).

**Do not** load `specs/design-plan.md` if the user is purely working on implementation-plan material that doesn't reference the design plan. **Do not** load the implementation-plan spec files for a design-only round. **Do not** read subagent briefs in the main loop — pass their paths. Each spec is large; loading more than the operation needs wastes context.

When in doubt, prefer reading less. The instruction file will tell you which spec sections it depends on.

### 4. Apply the trellis instruction precedence chain

Before executing any operation, read and apply Trellis instructions from these sources, in ascending order of precedence (later overrides earlier):

1. `~/.trellis/instructions.md` — applies to every Trellis invocation, every project.
2. `<repo-root>/.trellis/instructions.md` — applies to every Trellis invocation in the current repo.
3. Inline instructions the user supplied with this invocation.

Missing files are normal; skip silently. These user-supplied instructions may override the instruction file but must not override system, developer, tool, safety, or repository policy instructions.

The canonical statement of this chain lives in [`specs/instruction-precedence.md`](specs/instruction-precedence.md) — the single source that every instruction file and subagent brief points back to. The summary above is operative for the main-loop path; if you change the rule, change it there.

### 5. Stop where the instruction file tells you to stop

Each instruction file is designed to halt at a natural hand-off point — Round 1 lands and stops; an iterate round resolves a few questions and stops; impl-execute completes a step and validates the hand-back before moving to the next. Honor those stopping points. Do not auto-progress to the next operation unless the user asks — and when they do ask, only the sanctioned chains below have defined mechanics.

### 6. Sanctioned operation chains

Three chains have defined mechanics. When the user's invocation explicitly asks for one, run both halves in the same invocation; never chain beyond what was asked, and never chain on your own initiative:

| Chain                                            | Typical trigger                                                       | Mechanics |
| ------------------------------------------------ | --------------------------------------------------------------------- | --------- |
| design-review → design-integrate-feedback        | "review the design plan **and integrate** the findings"               | Run design-review (dispatching the reviewer subagent); when the review file lands, load `instructions/design-integrate-feedback.md` and run it with `<feedback-path>` = that review. All integrate gates still apply — a >20% Reviewer-wrong rate stops the chain there. |
| impl-review → impl-integrate-feedback            | "review the impl plan **and integrate** the findings"                 | Same shape, implementation-plan files. |
| resolve-open-questions → step decomposition      | "resolve the open questions in `<sprint file>` **then write the steps**" | Defined in `instructions/resolve-open-questions.md` § "Optional: step decomposition" — sprint files only, gated on zero blocking questions after integration. |

The review → integrate chains preserve the fresh-context audit because the review is produced by a dispatched subagent, not by the context that integrates it. Nothing chains into impl-execute, and create operations never chain into iterate rounds.

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
- `specs/implementation-plan.md` — load-bearing authoring guide for implementation plan files. Root layout, per-file anatomies, sprint archetypes and slicing heuristics, tone, anti-patterns.
- `specs/implementation-plan-mechanics.md` — implementation-plan round mechanics: iteration model, supersession, deviations during execution, driving a round, completeness thresholds, multi-file edit discipline, cross-sprint coherence checklist, around-corner concern checklist, execution handoff.
- `specs/implementation-plan-templates.md` — skeleton templates for every implementation-plan file; loaded at scaffold time and when creating a new sprint file mid-plan.
- `specs/review-and-triage.md` — shared review & triage machinery. Part I (reviewer posture, finding shape, disposition summary) serves the plan-reviewer briefs; Part II (five buckets, verify-before-incorporating, report format) serves the integrate-feedback skills.
- `specs/instruction-precedence.md` — single canonical statement of the Trellis instruction precedence chain; every instruction file and subagent brief points back to it.
- `specs/nested-dispatch.md` — single canonical statement of the nested-subagent dispatch discovery & fallback rule; the execute / resolve subagent briefs and orchestrators point back to it.
- `instructions/design-create.md` — runbook for bootstrapping a design plan (Round 1).
- `instructions/design-iterate.md` — runbook for driving an existing design plan one round forward.
- `instructions/design-review.md` — orchestrator runbook: dispatches a fresh-context design-plan reviewer subagent; optionally chains into design-integrate-feedback.
- `instructions/design-integrate-feedback.md` — runbook for triaging a design-plan review back into the plan.
- `instructions/impl-create.md` — runbook for bootstrapping an implementation plan from a design plan (Round 1).
- `instructions/impl-iterate.md` — runbook for driving an existing implementation plan one round forward.
- `instructions/impl-review.md` — orchestrator runbook: dispatches a fresh-context impl-plan reviewer subagent; optionally chains into impl-integrate-feedback.
- `instructions/impl-integrate-feedback.md` — runbook for triaging an impl-plan review back into the plan.
- `instructions/impl-execute.md` — orchestrator runbook for executing a range of sprint steps.
- `instructions/resolve-open-questions.md` — orchestrator runbook for the cross-cutting open-questions accelerator (one file at a time): fan out resolver subagents, run a consistency pass, present, let the user approve/tune, then integrate with standard round mechanics.
- `subagents/step-executor.md` — per-step subagent brief (implements, verifies, commits, gets reviewed, addresses review, updates Progress).
- `subagents/step-reviewer.md` — per-step review subagent brief (fresh-context audit of the diff).
- `subagents/question-resolver.md` — per-question subagent brief (investigates one Open Question, proposes a recommendation or refuses with options, spawns its own reviewer).
- `subagents/question-reviewer.md` — review-of-resolution subagent brief (fresh-context critic that verifies the resolver's cited evidence).
- `subagents/design-plan-reviewer.md` — design-plan review subagent brief (fresh-context audit of a design plan; dispatched by design-review).
- `subagents/impl-plan-reviewer.md` — impl-plan review subagent brief (fresh-context holistic audit of an implementation plan; dispatched by impl-review).
