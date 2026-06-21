---
name: impl-iterate
description: Iterate on an Implementation Plan
argument-hint: <root>
disable-model-invocation: true
---

# Implementation Plan — Iterate Skill

Skill prompt for an LLM agent. Activated when a user wants to drive an existing **implementation plan** through one more planning round (Round 2 onward).

The implementation plan is a **set of markdown files directly in the feature root** (`overview.md`, `decisions.md`, `status.md`, `progress.md`, sprint files, optionally `post-mortem.md`), not a single document. **`decisions.md` and `status.md` live directly in the feature root — not as sections inside `overview.md`.** Most planning failures hide in the seams between files — every rule in this prompt is calibrated to keep those seams coherent.

The round mechanics below are **load-bearing** and easy to drop under load. Read the prompt end-to-end every invocation; do not rely on memory of prior rounds.

---

## Required inputs

The user supplies the feature root `<root>`. Resolution rule:

- If the user named a feature root (e.g., "iterate the impl plan at `docs/refunds/`"), use that path as `<root>`.
- If the user named `design.md`, `overview.md`, `progress.md`, `decisions.md`, `status.md`, or a sprint file, use its parent directory as `<root>`.
- If neither holds — the path is ambiguous or doesn't exist — ask the user and stop.

After resolution, `<root>` is the feature root the rest of this brief operates on.

Before doing anything, in this order:

1. Read the [Implementation Plan Documents — Authoring Guide](../specs/implementation-plan.md) end to end. Do not skim.
2. Read **every implementation-plan file** in `<root>`, top to bottom, in this order:
   - `overview.md` — internalize philosophy, feature-wide locked decisions, sprint roster, dependency graph, open questions.
   - `decisions.md` — the plan-level (cross-cutting) Decisions log. Note the latest round tag.
   - `status.md` — the plan-level round-by-round audit trail. Note the current round number and the latest `_Next:_` clause.
   - `progress.md` — note which steps are checked off (some sprints may have shipped).
   - Each sprint file in numeric order (`01-*.md`, `02-*.md`, …) — Goal, Prerequisites, Deliverables, Out of scope, Locked decisions, Architecture notes, Public surface, Steps, Step Dependency Chart, Acceptance checklist, Open questions, sprint-scoped Decisions log, sprint-scoped Status / Feedback-incorporated / Deviations.
   - `post-mortem.md` if it exists.
3. Read the source design plan that `overview.md` cites in its framing block. The implementation plan implements the design plan; conflicts go back to the design plan, not into a sprint step.
4. Skim the project's `CLAUDE.md` (or equivalent) for conventions, banned patterns, authorization rules.

If the user provided no input beyond the skill invocation, stop here and say: *"Ready for Round <next>. Tell me which Open questions or which sprint to focus on, or share the input you'd like incorporated."* Do not pick on your own.

If the user provided input with the invocation, treat it as the round's seed and proceed with the workflow below.

---

## The round workflow

Every round follows the same shape, regardless of whether the round is fleshing out a sprint, resolving an overview-level question, fixing cross-sprint coherence, or re-slicing the roster. Skipping any step has cost the system a round at some point.

### Step 1 — Take inventory

Before saying anything substantive, internally note:

- The plan's current Round number per `status.md` (next entry will be `Round N+1`).
- Open questions at every scope — overview-level + per-sprint — with counts.
- Most recent Decisions log entries at every scope (plan-level in `decisions.md`; sprint-scoped inside each sprint file).
- Which sprints have shipped (final step `[x]` in `progress.md`) — those bodies are mostly read-only; capture corrections via "Feedback incorporated" or "Deviations applied during implementation" surfaces.
- Which sprints are still stubs vs. execution-ready (an execution-ready sprint has a populated Locked Decisions table, Architecture notes, Implementation Steps with concrete Verification, Step Dependency Chart, and Acceptance checklist).
- Any wording the most recent Status entry called out as superseded.

### Step 2 — Triage what this round will address

Pick **one** of these focuses — but recognize that resolving a single sprint's blocking Open questions and locking that same sprint to execution-ready are the **same focus**, not two. The menu below is about cross-sprint scope, not about stopping points within one sprint's arc:

- **Lock a single sprint to execution-ready.** Take a stub sprint, resolve any of its remaining `[blocks-impl]` / `[blocks-v1]` Open questions in the same round, and populate its Locked Decisions (10–25 rows), Architecture notes, Public surface, Implementation Steps with Goal/Actions/Deliverables/Verification, Step Dependency Chart, Acceptance checklist. The most common round shape after Round 1.
- **Resolve 1–5 overview-level Open questions.** When the questions are reasonably independent and don't snowball.
- **Resolve sprint-level Open questions for a single sprint — and continue in the same round to lock that sprint** when the resolutions clear the last blocker. The open questions exist precisely so the sprint can be locked; stopping between "questions resolved" and "sprint locked" makes the user re-invoke the skill for ceremony rather than for a decision. See "Aggressive locking" below.
- **Re-slice the sprint roster.** Splitting / folding / re-numbering sprints. Update `overview.md`'s roster and dependency graph, rename sprint files, regenerate `progress.md`, and call out the supersession in this round's `status.md` entry.
- **Fix cross-sprint coherence drift.** Prerequisite ↔ Deliverable name mismatches, dependency-graph contradictions, `progress.md` ↔ per-sprint Progress drift.

If the user named a focus, that's the round. Don't expand beyond what they asked for unless you flag the expansion explicitly.

**Aggressive locking.** When this round resolves sprint-level Open questions for Sprint NN, check at the end of Step 4 whether Sprint NN still has any `[blocks-impl]` or `[blocks-v1]` Open questions. If the answer is no, **do not stop and ask permission to continue** — populate the Locked Decisions table, Architecture notes, Public surface (when applicable), Implementation Steps with concrete Verification, Step Dependency Chart, and Acceptance checklist in this same round, then grade the round against `sprint-NN-execution-ready` (not the lesser "questions resolved" threshold). `[exploratory]` and `[deferred]` questions are allowed to survive into a locked sprint — they exist precisely so they don't gate execution. Flag any you carried through in the hand-off so the user can decide whether to close them before implementation.

The brake on aggressive locking: if locking would require inventing answers the design plan and the user haven't given you, **stop and surface the gap as a new Open question** — don't fabricate locked decisions to look productive. The bar is "I have enough information to populate every section honestly," not "the user resolved one question, so I'll write steps regardless." See "Don't fabricate locked decisions" in Posture rules.

### Step 3 — Frame each choice with alternatives

For each Open question this round will resolve:

- Surface **2–3 viable options** with trade-offs.
- Recommend one — be opinionated.
- Name what (which sprint, which surface, which downstream decision) gets unblocked if the question resolves this way.

For sprint-locking rounds, additionally surface:

- The **archetype** you're treating the sprint as (Foundation / Logic-layer / Capability / Integration / Helper / Async / Surface / Hardening per [implementation-plan.md § "Common sprint archetypes"](../specs/implementation-plan.md)). The archetype shapes what categories the Locked Decisions table needs.
- Any **structural ambiguity** the sprint has that you'd resolve by splitting / folding before locking — frame this as an option even if your recommendation is "lock as-is."

Then **wait for the user**. Do not auto-resolve. Do not pre-decide structure. Even when you have a strong opinion, the user picks.

If the user says "you pick," capture each call as a `(R<n>)` Decisions log entry tagged with rationale — cross-cutting calls in `decisions.md`, sprint-scoped calls in the affected sprint file — and surface the unilateral decisions in the round hand-off so the user can react.

### Step 4 — Update the affected files

For every resolution, edits land in **all** the files that surface holds. Many edits cross files — this is the highest-error path; follow the multi-file discipline below.

Within a single file:

1. **Rewrite the affected section** to reflect the chosen design. Purge obsolete wording — do not leave both old and new versions side-by-side. Supersession discipline is described in [implementation-plan.md § "Supersession"](../specs/implementation-plan.md).
2. **Add a tagged bullet to the relevant Decisions log** with `(R<n>)`. Cross-cutting calls go in the feature root's **`decisions.md`** (not a section inside `overview.md`); sprint-scoped calls go in the affected sprint file's Decisions log section. Each bullet leads with a **bold phrase** naming the call.
3. **Compress older Decisions-log entries** at every scope this round touched — walk both `decisions.md` and the affected sprint files' Decisions logs, condensing any entry this round (or an earlier round) superseded plus any whose details have gone stale, per [implementation-plan.md § "`decisions.md` anatomy"](../specs/implementation-plan.md). Keep the last 2–3 rounds and any still-load-bearing entry in full.
4. **Remove the resolved entry from Open questions** at the right scope. Do not append `(resolved)` in place — entries **move** to Decisions log.
5. **Add newly-surfaced sub-questions** to the right Open-questions list (overview vs. sprint). Keep them numbered append-only.

Across files (multi-file edits): follow [implementation-plan.md § "Multi-file edit discipline"](../specs/implementation-plan.md) — list the file cluster before editing any of it (which file owns the canonical wording; which files reference it; whether `overview.md` / `decisions.md` / `status.md` / `progress.md` and the per-sprint Progress section need updating), edit the canonical-owner first and consumers second, then `grep -rn '<old wording>' <root>` (zero hits is the only acceptable result; a hit means a consumer was missed). `progress.md` and per-sprint Progress sections stay in lockstep — edit one, edit the other in the same pass; the master wins on conflict.

### Step 5 — Re-read the plan end-to-end

Rewriting one file can silently invalidate wording elsewhere. Re-read the affected files, plus:

- `overview.md`'s feature-wide Locked Decisions table — does anything you locked at sprint level contradict an overview-level decision?
- The sprint roster + dependency graph — do all Prerequisite ↔ Deliverable pairs still match?
- `progress.md` against every sprint file's per-sprint Progress section.
- Architectural-invariants list in `overview.md` — does the new wording uphold them?

Fix what you find in the **same round**. Do not leave the plan internally contradictory.

### Step 6 — Sanity-check before stopping

Walk these checks before sending the hand-off message. Each one has cost the system a round when missed.

**Cross-file coherence:** walk every check in [implementation-plan.md § "Cross-sprint coherence checklist"](../specs/implementation-plan.md) — Prerequisite ↔ Deliverable matching, "deferred to Sprint NN" ownership, sprint-vs-overview decision conflicts, module-tree drift, `progress.md` ↔ per-sprint Progress lockstep, `status.md` supersessions paired with body edits, and the rest. Fix any drift this round introduced or surfaced.

**Within-file quality:**
- **Every new Decisions-log entry has rationale** — bold lead followed by a one-paragraph defense, not a bare bullet.
- **Every new Open question carries exactly one severity tag** (`[blocks-v1]` / `[blocks-impl]` / `[deferred]` / `[exploratory]`) and names the sprint(s) it blocks for `[blocks-v1]` / `[blocks-impl]` entries.
- **Every step's Verification names specific assertions, commands, or test names** — never "tests pass" / "compiles" / generic phrases.
- **No `(resolved)` tags** inside Open questions.
- **No marketing words** (`elegant`, `clean`, `robust`, `powerful`, `best-in-class`).
- **No "implementation details TBD."** Either lock the call, surface it as an Open question with rationale, or name the future round / sprint that closes it.
- **No design-level rationale leaking into a sprint** (re-arguing what the design plan already decided) — and no implementation specifics leaking up to `overview.md`.
- **v1 / Tier-0 compromises are labeled in place.**
- **Identifiers in backticks; absolute dates only; active voice.**

For shipped sprints: did you preserve the body and capture corrections via "Feedback incorporated" / "Deviations applied during implementation"? The shipped code is the source of truth; never edit a shipped sprint's body to pretend the original plan said the new thing.

Fix what you find here, in this round.

### Step 7 — Append exactly one Status entry

In the feature root's **`status.md`** (not a section inside `overview.md`). Two clauses — what changed, then a `_Next:_` clause naming the recommended next-round focus:

```
- **Round <n>**: <one-line summary>; <one phrase about what was superseded, if anything>. _Next:_ <one-line recommended focus, citing Open question IDs + tags + scope where relevant>.
```

Examples:

```
- **Round 4**: Sprint 03 locked to execution-ready; cron cadence + idempotency-key shape pinned. _Next:_ lock Sprint 04 — resolve the worker-reentrancy question (Sprint 04 Q2 [blocks-impl]) first.
- **Round 7**: Sprint 06 split into 06-resolve and 07-deduplicate; renumbered 07–10 → 08–11. R3's combined-worker decision superseded. _Next:_ flesh out new Sprint 07 (deduplicate) — Sprint 06 already locked from prior rounds.
- **Round 12**: every sprint now execution-ready; cross-sprint coherence checks pass. _Next:_ hand off to implementation.
```

Why the `_Next:_` clause: it persists the chat-only hand-off recommendation into the doc itself. A user resuming the plan a week later via `impl-iterate` reads the latest `_Next:_` and knows where to pick up — without depending on prior chat history.

Pick the `_Next:_` clause from your completeness assessment's "Recommended next-round focus" (Step 8) — they should agree. When the threshold graded is `sprint-NN-execution-ready` and that verdict is `complete`, the `_Next:_` clause is typically `lock Sprint <NN+1>`. When the threshold is `plan-complete`, the clause is `hand off to implementation` (or `run impl-review first`).

Cite supersessions explicitly (e.g., `R7 supersedes R5's contiguity-cache rule`; `re-sliced 06 into 06-resolve and 07-deduplicate; renumbered 07–10 → 08–11`).

If the round materially changed a single sprint, append a sprint-scoped Status / Feedback-incorporated entry to that sprint file too (in addition to the entry in `status.md`). Status entries are append-only **for the round you are appending** — never edit *prior* entries to falsify what happened.

However, **compress older Status entries** that no longer carry weight — when you append the new round, walk both `status.md` and any sprint-level Status / Feedback-incorporated logs, per [implementation-plan.md § "`status.md` anatomy"](../specs/implementation-plan.md). Keep the last 2–3 rounds in full at each scope; never delete a round entry outright — round numbering must stay contiguous and grep-able.

### Step 8 — Emit the completeness assessment to chat

This is **mandatory** at the end of every round, including small ones. The assessment is a chat-only output (not part of any plan file).

Emit it exactly as specified in [implementation-plan.md § "Round-end completeness assessment"](../specs/implementation-plan.md) — the verbatim block structure (including the `**Threshold graded**` field), the verdict semantics, and the calibration rules all live there. Don't restate them here.

Round-specific deltas for an iterate round:

- **Pick the threshold this round aimed at** — `scaffold-complete` (Round 1 just landed), `sprint-NN-execution-ready` (the round just locked Sprint NN), or `plan-complete` (every sprint is execution-ready). The full definitions are in the spec section above.
- **Always name the threshold and grade only against it.** A round that locks Sprint 03 grades against `sprint-03-execution-ready`, not `plan-complete`; "Sprint 04+ are still stubs" is not a load-bearing-open item for that round.
- **Cross-sprint coherence is load-bearing at every threshold above scaffold-complete** — a Prerequisite that names a Deliverable spelled differently in a prior sprint is a load-bearing-open item, not a nit.
- **The verdict feeds Step 7's `_Next:_` clause** — they must agree.

### Step 9 — Stop. Wait for the user

Don't auto-progress to the next round. Each round is a discrete user-driven step.

"Stop" means the round's threshold (Step 8) has been met — not that the first focus item finished. If this round resolved a sprint's blocking Open questions and you have enough information to lock the sprint, keep going within this round per Step 2's "Aggressive locking" — the round ends at `sprint-NN-execution-ready`, not at "questions resolved." The next round begins when the user picks a new focus (typically the next sprint to lock, or a re-slice).

---

## Posture rules (read every round)

- **Surface alternatives, don't decide unilaterally.** When the user has skin in the game, the agent's job is to frame the call sharply.
- **Don't fabricate locked decisions.** If neither the design plan nor the user decided it, it's an Open question for the affected sprint — even if you have a confident default. Single-row Locked Decisions tables are a planning failure; so are tables padded with fake "locks" the agent invented.
- **Don't redesign inside a sprint.** When a step needs a decision the design plan didn't make, surface it as an Open question (or escalate to the design plan) — don't bury the answer in a step.
- **Don't paraphrase the design plan into one giant sprint.** Re-derive sprint structure from outcomes (what ships when), not from the design plan's chapter headings.
- **Steps are executable, not narrative.** "We considered X but went with Y" belongs in Architecture notes, not Actions.
- **Verification per step is non-negotiable** and concrete. Name a test, a command, a query, an assertion. "It compiles" is not verification.
- **Bold the decision lead** in every Decisions-log bullet, every Architecture note.
- **Treat shipped sprints as read-only.** If a sprint's final step is `[x]`, the code is the source of truth — capture corrections via "Feedback incorporated (post-review)" or "Deviations applied during implementation" instead of editing the body to pretend the original plan said something else.

---

## Anti-patterns specific to iteration

- **Don't stop after resolving a sprint's blocking Open questions when you have enough information to lock it.** Resolving the questions and writing the Locked Decisions / Architecture notes / Implementation Steps / Step Dependency Chart / Acceptance checklist is one round, not two. Stopping early forces the user to re-invoke the skill just to ask you to do the next obvious thing. The exception is when locking would require inventing answers — then surface the gap as a new Open question instead. See Step 2's "Aggressive locking" guidance.
- **Don't skip the completeness assessment.** Mandatory every round.
- **Don't skip the `status.md` entry** when the round was small. Every round earns one entry.
- **Don't put a Decisions log or Status section inside `overview.md`.** Those live directly in the feature root as `decisions.md` and `status.md`. Any "Decisions log" / "Status" heading inside `overview.md` is a planning bug — move the content out, then delete the heading.
- **Don't skip the sanity-check pass.** It is the cheapest way to catch the cross-file inconsistencies your own edits introduced.
- **Don't skip the post-edit grep** for multi-file edits. A consumer left referencing the rejected wording is silent drift.
- **Don't append `(resolved)` tags** inside Open questions.
- **Don't pre-create `post-mortem.md`.** It appears the first time a sprint ships — never at scaffold or iteration time.
- **Don't auto-progress to the next round.**
- **Don't renumber Open questions** within a list. Append-only.
- **Don't re-slice the sprint roster casually.** It is a structural change with high blast radius — surface it explicitly to the user, get confirmation, then renumber files, regenerate `progress.md`, update the dependency graph, and note the supersession in the Status entry.
- **Don't conflate the design plan's Decisions log with the implementation plan's.** The design plan logs what the system **is**; the implementation plan logs how and in what order it gets **built**. Cross-reference; don't merge.

---

## Additional Instructions

The **Trellis instruction precedence chain** (`~/.trellis/instructions.md` → `<repo-root>/.trellis/instructions.md` → instructions supplied with this invocation) is applied by `SKILL.md` before this file is dispatched, so it is already in effect; honor it. Canonical statement — the single source shared by every instruction file and subagent brief: [instruction-precedence.md](../specs/instruction-precedence.md).
