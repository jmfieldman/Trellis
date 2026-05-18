---
name: impl-iterate
description: Iterate on an Implementation Plan
argument-hint: <root> | <impl-plan-dir>
disable-model-invocation: true
---

# Implementation Plan — Iterate Skill

Skill prompt for an LLM agent. Activated when a user wants to drive an existing **implementation plan** through one more planning round (Round 2 onward).

The implementation plan is a **directory** of markdown files (`overview.md`, `decisions.md`, `status.md`, `progress.md`, sprint files, optionally `post-mortem.md`), not a single document. **`decisions.md` and `status.md` are top-level files — not sections inside `overview.md`.** Most planning failures hide in the seams between files — every rule in this prompt is calibrated to keep those seams coherent.

The round mechanics below are **load-bearing** and easy to drop under load. Read the prompt end-to-end every invocation; do not rely on memory of prior rounds.

---

## Required inputs

The user supplies the feature root `<root>` (most common case), or an explicit `<impl-plan-dir>`. Resolution rule:

- If the user named a feature root (e.g., "iterate the impl plan at `docs/refunds/`"), set `<impl-plan-dir>` to `<root>/impl/`.
- If the user named a directory that already contains `overview.md` (e.g., "iterate `docs/refunds/sprint-plans/`"), treat that as `<impl-plan-dir>` directly.
- If neither holds — the path is ambiguous or doesn't exist — ask the user and stop.

After resolution, `<impl-plan-dir>` is the canonical directory the rest of this brief operates on.

Before doing anything, in this order:

1. Read the [Implementation Plan Documents — Authoring Guide](../specs/implementation-plan.md) end to end. Do not skim.
2. Read **every file** in the plan directory at `<impl-plan-dir>`, top to bottom, in this order:
   - `overview.md` — internalize philosophy, feature-wide locked decisions, sprint roster, dependency graph, open questions.
   - `decisions.md` — the plan-level (cross-cutting) Decisions log. Note the latest round tag.
   - `status.md` — the plan-level round-by-round audit trail. Note the current round number and the latest `_Next:_` clause.
   - `progress.md` — note which steps are checked off (some sprints may have shipped).
   - Each sprint file in numeric order (`01-*.md`, `02-*.md`, …) — Goal, Prerequisites, Deliverables, Out of scope, Locked decisions, Architecture notes, Public surface, Steps, Step Dependency Chart, Acceptance checklist, Open questions, sprint-scoped Decisions log, sprint-scoped Status / Feedback-incorporated / Deviations.
   - `post-mortem.md` if it exists.
3. Read the source design plan that `overview.md` cites in its framing block. The implementation plan implements the design plan; conflicts go back to the design plan, not into a sprint step.
4. Skim the project's `CLAUDE.md` (or equivalent) for conventions, banned patterns, authorization rules.

If the user provided no input beyond the skill invocation, stop here and say: *"Ready for Round &lt;next&gt;. Tell me which Open questions or which sprint to focus on, or share the input you'd like incorporated."* Do not pick on your own.

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

Pick **one** of these focuses (a round that tries to do all of them at once usually bungles cross-sprint coherence):

- **Lock a single sprint to execution-ready.** Take a stub sprint and populate its Locked Decisions (10–25 rows), Architecture notes, Public surface, Implementation Steps with Goal/Actions/Deliverables/Verification, Step Dependency Chart, Acceptance checklist. The most common round shape after Round 1.
- **Resolve 1–5 overview-level Open questions.** When the questions are reasonably independent and don't snowball.
- **Resolve 1–5 sprint-level Open questions** within a single sprint.
- **Re-slice the sprint roster.** Splitting / folding / re-numbering sprints. Update `overview.md`'s roster and dependency graph, rename sprint files, regenerate `progress.md`, and call out the supersession in this round's `status.md` entry.
- **Fix cross-sprint coherence drift.** Prerequisite ↔ Deliverable name mismatches, dependency-graph contradictions, `progress.md` ↔ per-sprint Progress drift.

If the user named a focus, that's the round. Don't expand beyond what they asked for unless you flag the expansion explicitly.

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
2. **Add a tagged bullet to the relevant Decisions log** with `(R<n>)`. Cross-cutting calls go in **`decisions.md`** (the plan-level top-level file — not a section inside `overview.md`); sprint-scoped calls go in the affected sprint file's Decisions log section. Each bullet leads with a **bold phrase** naming the call.
3. **Compress older Decisions-log entries** at every scope this round touched. Walk both `decisions.md` and the affected sprint files' Decisions logs; condense any entry that this round (or an earlier round) has superseded, plus any entry whose details have gone stale. Compressed form: `- **<lead>.** Superseded by R<m>. (R<n>)` — round tag + supersession pointer preserved, rationale paragraph dropped. Keep the last 2–3 rounds and any still-actively-load-bearing entry in full. See [implementation-plan.md § "`decisions.md` anatomy"](../specs/implementation-plan.md). The audit trail survives; only obsolete prose is dropped.
4. **Remove the resolved entry from Open questions** at the right scope. Do not append `(resolved)` in place — entries **move** to Decisions log.
5. **Add newly-surfaced sub-questions** to the right Open-questions list (overview vs. sprint). Keep them numbered append-only.

Across files (multi-file edits):

1. **List every file the resolution will touch before editing any of them.** Which file owns the canonical wording? Which files reference it? Does `overview.md` need updating (module-tree, dependency-graph one-liner, sprint roster, feature-wide locked decisions)? Do `decisions.md` or `status.md` need a new entry? Does `progress.md` need updating? Does the affected sprint's per-sprint Progress section need to mirror a `progress.md` change?
2. **Edit canonical wording first**, consumers second. The file that *owns* a renamed Deliverable / Locked Decision / module-tree entry is updated before any file that *references* it.
3. **After the edit pass, grep the directory for the rejected wording** (`grep -rn '<old wording>' <impl-plan-dir>`). Zero hits is the only acceptable result. One or more hits means a consumer was missed.

`progress.md` and per-sprint Progress sections must stay in lockstep. If you edit one, edit the other in the same pass. The master `progress.md` wins on conflict; the per-sprint copy is reconciled to it (unless the master is the broken side).

### Step 5 — Re-read the directory end-to-end

Rewriting one file can silently invalidate wording elsewhere. Re-read the affected files, plus:

- `overview.md`'s feature-wide Locked Decisions table — does anything you locked at sprint level contradict an overview-level decision?
- The sprint roster + dependency graph — do all Prerequisite ↔ Deliverable pairs still match?
- `progress.md` against every sprint file's per-sprint Progress section.
- Architectural-invariants list in `overview.md` — does the new wording uphold them?

Fix what you find in the **same round**. Do not leave the directory internally contradictory.

### Step 6 — Sanity-check before stopping

Walk these checks before sending the hand-off message. Each one has cost the system a round when missed.

**Cross-file coherence:**
- **Every Prerequisite has a matching Deliverable** in a prior sprint, or names a peer-service surface that exists, or is flagged as an audit-or-build branch.
- **Every "deferred to Sprint NN" out-of-scope entry** corresponds to an actual Sprint NN whose Deliverables list owns the item.
- **Sprint-level Locked Decisions don't override overview-level ones.** They refine; they do not contradict.
- **Module-tree drift:** every file an existing sprint commits to creating appears in `overview.md` § Module/Directory layout.
- **`progress.md` ↔ per-sprint Progress** match exactly for every sprint.
- **`status.md` supersessions are paired with body edits.** A re-slice / supersession announced in `status.md` actually shows in the affected sprint files.

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

In **`status.md`** (the top-level plan-level Status file — not a section inside `overview.md`). Two clauses — what changed, then a `_Next:_` clause naming the recommended next-round focus:

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

However, **compress older Status entries** that no longer carry weight. When you append the new round, walk both `status.md` and any sprint-level Status / Feedback-incorporated logs; condense entries whose `_Next:_` clause is long-finished (drop the `_Next:_`) or whose round-summary detail is no longer load-bearing because later rounds superseded it (collapse to `**Round <n>**: <one-clause summary>`). Keep the last 2–3 rounds in full at each scope. Never delete a round entry outright — round numbering must stay contiguous and grep-able. See [implementation-plan.md § "`status.md` anatomy"](../specs/implementation-plan.md). Compression is *not* falsification: obsolete prose is dropped, the audit trail (round number + supersession link) is preserved.

### Step 8 — Emit the completeness assessment to chat

This is **mandatory** at the end of every round, including small ones. The assessment is a chat-only output (not part of any file in the directory).

Implementation plans use a three-threshold model: pick the threshold that matches what this round was aiming at.

- **`scaffold-complete`** — Round 1 has just landed. `overview.md` populated (no Decisions/Status sections); `decisions.md` exists (empty or R1 entries); `status.md` has the R1 entry; one stub per roster entry; `progress.md` with a section per sprint; no `post-mortem.md`.
- **`sprint-NN-execution-ready`** — the round just locked Sprint NN. Its file has the populated Locked Decisions table (10–25 rows), Architecture notes, Public surface (when applicable), Implementation Steps (5–10) each with concrete Verification, Step Dependency Chart, Acceptance checklist. Cross-sprint coherence holds — Prerequisites match prior Deliverables, `progress.md` mirrors the new step list.
- **`plan-complete`** — every sprint has cleared the execution-ready bar. Cross-sprint coherence holds across the whole directory.

Use this exact structure:

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

Verdict semantics (relative to the chosen threshold; full definitions in [implementation-plan.md § "Round-end completeness assessment"](../specs/implementation-plan.md)):

- **`not-yet-complete`** — at least one load-bearing item from the threshold's checklist is open or undecided.
- **`substantially-complete`** — load-bearing items are decided but rough edges remain (drift, stale wording, marketing language, missing tag).
- **`complete`** — load-bearing items are decided *and* the affected files are internally clean.

Calibration:

- **Always name the threshold.** A round that locks Sprint 03 grades against `sprint-03-execution-ready`, not `plan-complete`. Listing "Sprint 04+ are still stubs" against a Sprint 03 round is noise.
- **Default to `not-yet-complete`** until the threshold's full checklist is met. The bar for `substantially-complete` is high.
- **Don't oscillate.** Verdicts are about plan state, not round size.
- **Cross-sprint coherence is load-bearing at every threshold above scaffold-complete.** A Prerequisite that names a Deliverable spelled differently in a prior sprint is a load-bearing-open item, not a nit.
- **Don't pad.** A 12-bullet "top nits" list is a sign the plan isn't actually substantially-complete — promote the most material items into "load-bearing-open" and re-grade.
- **Cite the file + section** in every bullet (`Sprint 04 § Locked Decisions`, `overview.md § Dependency graph`, `progress.md § Sprint 06`). Bare-section refs are too ambiguous in a directory of files.

### Step 9 — Stop. Wait for the user

Don't auto-progress to the next round. Each round is a discrete user-driven step.

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

- **Don't skip the completeness assessment.** Mandatory every round.
- **Don't skip the `status.md` entry** when the round was small. Every round earns one entry.
- **Don't put a Decisions log or Status section inside `overview.md`.** Those live in `decisions.md` and `status.md` at the top level of the plan directory. Any "Decisions log" / "Status" heading inside `overview.md` is a planning bug — move the content out, then delete the heading.
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

Before executing this skill, read and apply Trellis instructions from these sources, in ascending order of precedence:

1. `~/.trellis/instructions.md`
2. `<repo-root>/.trellis/instructions.md`
3. Additional instructions included with this skill invocation

`<repo-root>` is the root of the current Git repository. Missing instruction files are normal; skip them silently.

If instructions conflict, later sources override earlier sources. Invocation-specific instructions apply only to the current run and have the highest precedence.

These instructions may override this skill document, but they must not override system, developer, tool, safety, or repository policy instructions.
