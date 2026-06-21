---
name: resolve-open-questions
description: Resolve the Open Questions in a single Trellis file — parallel resolver subagents propose answers, you approve or tune, then the answers are integrated with standard Trellis mechanics
argument-hint: <trellis-file-path>
disable-model-invocation: true
---

# Resolve Open Questions — Orchestrator Skill

You are the **orchestrator** for resolving the Open Questions in **one** Trellis file. This is a cross-cutting accelerator: it works on any Trellis document that carries an Open Questions section — a design plan (`design.md`), an implementation-plan `overview.md`, or a single sprint file (`NN-*.md`). It exists to collapse the most tedious part of the workflow — answering a pile of legitimate-but-mechanical open questions one at a time before a sprint can be fleshed out — into a single fan-out / review / approve / integrate pass.

The discipline: a per-question subagent investigates each question against the codebase, the design plan, and the project's conventions, gets its proposal independently reviewed, and hands back a recommendation. You gather the proposals, run a consistency pass, present them as a table the user can approve or tune in one read, and only then integrate the approved decisions into the document using standard Trellis round mechanics.

## Hard invariant — read this first

**You do not resolve questions yourself. Every question in scope is dispatched to a per-question resolver subagent via the `Agent` tool.** There are no exceptions — not for "obvious" questions, not for ones you're confident about. If you find yourself reasoning out the answer to an open question, grepping the codebase to decide a column name, or weighing options for a question, **stop** — you have left the orchestrator role. Dispatch the resolver.

The orchestrator's own work is narrow: read the one file, extract and filter the questions, schedule and dispatch resolvers, run the consistency pass, present, run the tuning loop, and **integrate the approved decisions** (the integration *is* yours — it's a single coherent editing pass, like an `impl-iterate` round; only the *resolution* is fanned out).

The one carve-out to "don't reason about answers": the **consistency pass** (step 4) requires you to pick among answers two *resolvers already produced* when they collide. That is reconciling, not originating — you never invent an answer to a question from scratch; you select the coherent one from what the resolvers returned, and you always surface the choice to the user.

You operate on **one file per invocation**. The user names the file. Never accept "resolve all open questions across the plan" — that's a sequence of single-file invocations the user drives.

The two subagent briefs live in the trellis skill's `subagents/` directory:

- [`question-resolver.md`](../subagents/question-resolver.md) — the per-question resolver (it spawns its own reviewer).
- [`question-reviewer.md`](../subagents/question-reviewer.md) — the independent reviewer the resolver spawns.

Pass each subagent the absolute path to its brief plus the per-question parameters; the subagent reads the brief itself. Do **not** copy a brief into the spawn prompt.

---

## Inputs

Extract from the user's natural-language invocation:

- **`<file-path>`** (required) — an absolute or repo-relative path to a single Trellis file with an Open Questions section: `design.md`, `overview.md`, or a sprint file `NN-<topic>.md`. The feature root `<root>` is the file's parent directory.
- **Scope override** (optional, free text) — by default this skill resolves only **blocking** questions (`[blocks-v1]` and `[blocks-impl]`). If the user says "all questions" / "include deferred and exploratory," widen to every Open Question.
- **Step-decomposition instruction** (optional, free text) — by default the skill **stops at integration**. Only if the user explicitly asks ("then write the steps," "lock the sprint after," "decompose into steps") *and* the file is a sprint file does the skill continue into step decomposition. See "Optional: step decomposition" at the end.
- **Focus / other inline instructions** (optional) — forwarded verbatim to every resolver as overrides.

If `<file-path>` is missing or names a file that isn't one of the three Open-Questions-bearing Trellis files, ask the user for it and stop. `decisions.md`, `status.md`, `progress.md`, and `post-mortem.md` have no Open Questions section — reject them with a one-line explanation.

---

## Pre-flight checks

Run these before dispatching any resolver. Stop and report on any failure.

1. **The file exists and is a Trellis file with an Open Questions section.** `Read` it. Confirm an `## Open questions` (or `## Open Questions`) section is present. If absent, stop — there's nothing to resolve.
2. **Determine the doc type.** `design` if the file is `design.md`; `impl` if it's `overview.md` or a sprint file. This selects the integration mechanics later.
3. **Locate the feature root** as the file's parent directory. For an `impl` file, confirm `overview.md`, `decisions.md`, `status.md`, and `progress.md` exist in `<root>`; warn (don't stop) if any are missing — integration needs them. Confirm `<root>/design.md` exists; it's the source of truth the resolvers read.
4. **Step-decomposition guard.** If the user asked for step decomposition but `<file-path>` is not a sprint file, tell them decomposition only applies to sprint files and proceed with resolution only (don't stop the whole run).

Through question extraction and dispatch, the orchestrator reads only the **one target file**. **Integration is different** — it follows the standard multi-file discipline and may read and edit the rest of the implementation-plan cluster the resolutions actually touch (`overview.md` structural surfaces, `decisions.md`, `status.md`, an owning sprint file, consumers of any renamed wording), exactly as an `impl-iterate` round does. The "one file per invocation" rule scopes *which file's Open Questions get processed* — not which files integration is allowed to touch. What the orchestrator never reads is the codebase — that's the resolvers' job.

---

## Workflow

### 1. Extract and filter the questions

From the target file's Open Questions section, enumerate every entry: its number, full text, and severity tag. Then filter:

- **Default:** keep `[blocks-v1]` and `[blocks-impl]`. Skip `[deferred]` and `[exploratory]` — those are deliberately punted; resolving a deferred question contradicts the punt. Note in the final report how many you skipped and that the user can re-invoke with "all questions" to include them.
- **Scope override:** keep all questions.
- **Untagged questions** are a process bug per the authoring guides (every Open question should carry exactly one severity tag). Don't silently promote them to blocking — surface them to the user before dispatch, note the missing tag, and ask whether to include each. Dispatch a resolver for an untagged question only with the user's explicit say-so.

If nothing survives the filter, report "no blocking Open questions in `<file>`; re-invoke with 'all questions' to include `[deferred]` / `[exploratory]`" and stop.

### 2. Schedule the dispatch (parallel by default)

Default to running every in-scope question's resolver **in parallel** — they are independent investigations and the resolvers do not edit anything, so there's no write contention.

Serialize a pair only when one question's answer is a **hard input** to another's (resolving Q5 is impossible until Q3 is decided — e.g., Q5 asks "given the cursor shape from Q3, how do we paginate"). When you serialize, resolve the upstream question first, then pass its decided answer into the downstream resolver's parameters. Keep serialization rare — you don't need to predict every soft interaction, because the consistency pass (step 4) is the real safety net for cross-answer coherence.

### 3. Dispatch the resolver subagents

For each in-scope question, spawn a resolver with `subagent_type=general-purpose`. Resolvers in the same parallel wave are dispatched together. The spawn prompt:

```
You are resolving one Open Question from a trellis planning file.

Read the brief at:
  <absolute-path-to-trellis-dir>/subagents/question-resolver.md

That brief tells you exactly what to do. Read it end to end before doing anything else.
It also tells you to load Trellis instruction files from `~/.trellis/instructions.md` and `<repo-root>/.trellis/instructions.md` when present; do that before resolving.

Parameters:
- File path: <absolute file path>
- Feature root: <absolute root>
- Doc type: <design | impl>
- The question: <number + full verbatim text + severity tag>
- Sibling questions in this run: <numbered list of the other in-scope questions, text + tag>
- Upstream decided answers (if this resolver was serialized after a dependency): <question N → decided option, or "(none)">
- Design plan path: <root>/design.md
- Reviewer brief (you will spawn the reviewer yourself): <absolute-path-to-trellis-dir>/subagents/question-reviewer.md

Additional user instructions (forwarded verbatim; override the brief on conflict):
<paste any free-text instructions the user attached, or "(none)">

When you are done, return a single fenced YAML block exactly as your brief specifies (keys: question_number, question_tag, verdict, recommended_option, confidence, options, recommendation_basis, depends_on, reviewer_verdict, reviewer_note, needs_human_because, routes_to_design_because). Nothing else.
```

Resolve `<absolute-path-to-trellis-dir>` from wherever the top-level trellis `SKILL.md` was loaded — do not assume a fixed install location. The two relative links at the top of this file are the canonical references. **Do not run the resolvers in the background** — every proposal must return before the consistency pass (step 4).

**Reviewer-dispatch fallback.** A resolver returns `reviewer_verdict: not_run` when its harness can't dispatch a nested reviewer. For each such proposal, **you** dispatch a `question-reviewer` yourself — from your context, which hasn't seen the resolver's reasoning, so the fresh-context audit is preserved. Build the reviewer's spawn prompt the way the resolver's brief does, passing the reviewer's required inputs: file path, feature root, doc type, the question, **the resolver's proposal** (its verdict, recommended option, the options with pros/cons, basis, and confidence — taken from the returned YAML), the sibling questions, the design plan path, and any additional user instructions. Don't background it. Then fold its verdict in — the full mapping, not a subset:

- **`sound`** — keep the proposal; set `reviewer_verdict: sound`.
- **`revise`** — re-dispatch *that one* resolver with the reviewer's correction as added context. (The re-dispatched resolver will again try to spawn its own reviewer; if it again comes back `not_run`, run this fallback once more, then surface if it still can't be reviewed.)
- **`dissent`** — keep the proposal; attach the reviewer's note to `reviewer_note`; set `reviewer_verdict: dissent`.
- **`escalate`** — you lack the resolver's investigation context to overrule a reclassification, so don't silently apply it. **Surface the reviewer's escalate to the user** as a flagged row in the presentation and let them decide whether to accept the reclassification.
- **`blocked`** — your own dispatch couldn't complete; surface it as a failure (there is no further fallback to fall back to).

### 4. Consistency pass

Once every proposal is in, before presenting, check the set for coherence:

- **Cross-answer contradictions.** Two proposals that pick conflicting names / shapes / boundaries, or one whose `depends_on` names a sibling that resolved a different way than it assumed. Pick the coherent resolution (favor the higher-confidence proposal, or the one the design plan / conventions back) and reconcile the other. **The override always surfaces to the user** — attach a `_Consistency:_` note (e.g., *"Q5 → option A, not the resolver's B, to stay consistent with Q3's cursor shape"*) no matter how you reconcile. If the coherent answer is an option the resolver **already listed**, swap to it in place and attach the note. If the coherent answer is an option **no resolver produced**, you must **re-dispatch** that resolver — an invented option has to pass a resolver + reviewer before it reaches the user — and carry the `_Consistency:_` note onto the re-dispatched result so the reconciliation stays visible.
- **Upstream with no decided answer.** If a proposal's `depends_on` names a sibling that came back `escalate-as-is` (not yet picked) or `route-to-design` (won't be answered here), the downstream question has no upstream answer to align to and can't be coherently integrated. Mark it left-open / flagged in the presentation; don't guess the upstream to force the downstream through.
- **Locked-decision conflicts.** Any proposal that contradicts a Locked Decision in the file or in `overview.md`. Flag it in the presentation as needing attention rather than silently applying it.

The consistency pass is plain orchestrator reasoning over the returned YAML + the file's locked decisions — it is **not** a place to re-answer questions from scratch. If it surfaces a genuine new question, that's a flag for the user, not a sixth resolver.

### 5. Present the decision table

Emit one block to chat. Lead with a one-line distribution, then one entry per question. Group nothing away — the user is about to act on this.

```markdown
### Resolve Open Questions — `<file basename>` (<doc type>)

**Proposed:** <R> recommend / <E> escalate-as-is / <D> route-to-design  ·  confidence: <h> high / <m> medium / <l> low

**Q<n>** `[<tag>]` — <question, one line>
  - **Recommended: <letter>** [<confidence>] — <option summary>
    _Basis:_ <cited basis>
  - A — <summary> · ✅ <pro> · ⚠️ <con>
  - B — <summary> · ✅ <pro> · ⚠️ <con>
  - _Reviewer:_ <surface the reviewer's note for `dissent`, `revised`, and `reclassified` — what it disagreed with, what a revise changed, or a reclassification; omit for `sound` / `not_run`>
  - _Consistency:_ <override note, if any>

**Q<n>** `[<tag>]` — <question> — **NEEDS YOU** (escalate-as-is)
  - <why this is yours: needs_human_because>
  - A — <summary> · ✅ <pro> · ⚠️ <con>
  - B — <summary> · ✅ <pro> · ⚠️ <con>

**Q<n>** `[<tag>]` — <question> — **ROUTES TO DESIGN**
  - <why: routes_to_design_because> — won't be resolved here; surfaced for a design round.
  - A / B options shown to inform that round.
```

Then prompt: *"Approve all, or tune specific ones (e.g. 'use B for Q4', 'for Q3 do X instead', 'route Q7 to design'). Escalate-as-is items need a pick from you before they integrate; route-to-design items will be left open."*

Calibration:
- Lead with the recommendation and its confidence so the eye lands on what to scrutinize. Low-confidence and dissent/override rows are where the user's attention is worth spending.
- Keep each option to one line of summary + a pro + a con. The user is skimming to veto, not reading essays.
- Mark `escalate-as-is` and `route-to-design` rows distinctly — they behave differently at integration.

### 6. Tuning loop

The user approves wholesale or adjusts specific questions. **Do not re-spawn resolvers** when the user tunes — just update the held decision set (the user's pick overrides the recommendation) and re-confirm the deltas in one line each. Only re-investigate if the user explicitly asks you to ("look at Q4 again, I don't buy the basis"), in which case re-dispatch that one resolver.

The held decision set after tuning is: for each question, the **chosen option** (recommended, or the user's override), or "left open" (an `escalate-as-is` the user didn't pick, or a `route-to-design`). Loop until the user signals approval ("approved," "go," "integrate," "lgtm").

### 7. Integrate — standard Trellis mechanics

This is the orchestrator's own editing pass — one coherent round, like `impl-iterate` / `impl-integrate-feedback`. Load the relevant authoring guide before editing: [design-plan.md](../specs/design-plan.md) for a `design` file, [implementation-plan.md](../specs/implementation-plan.md) for an `impl` file.

For each question with a **chosen option**:

1. **Rewrite the affected body section** to reflect the decision. For a sprint file, populate the relevant **Locked Decisions** row(s) with the chosen value. **Purge stale wording** — never leave both the old phrasing and the new decision side by side.
2. **Move the question out of Open Questions** into the Decisions log. Never append `(resolved)` in place — entries *move*. Tag the new Decisions-log bullet with the round number `(R<n>)`, bold the decision lead, and include the rationale (the resolver's basis, distilled).
   - For an `impl` file: route by the **nature** of the resolved call, not the file it was asked in (matching `impl-iterate`). A cross-cutting call goes in the feature root's **`decisions.md`**; a sprint-scoped call goes in the **owning sprint file's** Decisions log section. Usually an overview-file question resolves to a cross-cutting call and a sprint-file question to a sprint-scoped one — but not always. When an overview question resolves to a sprint-scoped call, log it in the owning sprint file (and edit that sprint file's body if the call changes it); when a sprint question surfaces a cross-cutting call, log it in `decisions.md`.
   - For a `design` file: the call goes in `design.md`'s own Decisions log.
3. **Multi-file discipline (impl only).** If a resolution renames or reshapes something a sibling file references, follow the canonical-first edit order: edit the file that *owns* the wording, then each consumer, then `grep -rn '<old wording>' <root>` — zero hits is the only acceptable result. (Same canonical-first / grep discipline as [`impl-iterate.md`](./impl-iterate.md) § "Step 4 — Update the affected files" → "Across files (multi-file edits)".)

For each question **left open**:

- **`escalate-as-is` the user didn't decide** — leave the entry in Open Questions, unchanged. Note in the report that it still needs a human.
- **`route-to-design`** — leave the entry in Open Questions. Do **not** edit `design.md` from here (authorization: this skill edits the one file's plan layer, not the design plan when operating on an impl file). Surface it in the report as needing a `design-iterate` round. (When the target file *is* `design.md`, a route-to-design verdict is incoherent — the resolver shouldn't emit it; if one slips through, treat it as escalate-as-is.)

Then, once per run:

4. **Append exactly one Status entry — but only if this run integrated at least one decision.** If every in-scope question was left open (all `escalate-as-is` unpicked and/or `route-to-design`), the run changed nothing: do **not** append a Status entry or advance the round number. Report the all-left-open state, recommend the design round (or ask the user for picks), and stop — a contiguous round number is load-bearing and isn't spent on a no-op. Otherwise, append one entry — to `status.md` for an `impl` file, or to `design.md`'s Status section for a `design` file. **Recompute the round number** as the next plan-level integer by *re-reading* `status.md` (impl) / `design.md`'s Status (design) immediately before appending — don't cache it from pre-flight, so a round that landed concurrently isn't double-numbered. Format: `**Round <n>**: resolved <k> Open question(s) via resolve-open-questions (<short summary of the most material calls>); <e> escalated, <d> routed to design. _Next:_ <one-line recommended focus, citing remaining Open question IDs + tags>.` The counts reflect the **final held set after tuning** (a user-overridden `route-to-design` counts as resolved, not routed). Compress older Decisions-log and Status entries per the authoring guides — **only in the file(s) this run actually wrote to**; an obsolete entry sitting in a sprint file this run never opened is out of scope, so surface it in the report instead of reaching for it. (Drop superseded rationale, collapse long-finished `_Next:_` clauses; keep the last 2–3 rounds in full.)
5. **Add any newly-surfaced sub-questions** a resolution spun off to the right Open-questions list, append-only, each tagged.
6. **Re-read and grep.** Re-read the edited file(s) for wording your edits orphaned; for impl multi-file edits, confirm the grep for rejected wording is clean. Fix what you find in this same round.

Do **not** touch `progress.md` unless step structure changed (it won't, unless step decomposition runs).

### 8. Report

Emit a terse summary:

```markdown
### resolve-open-questions — summary

**File:** `<basename>` (<doc type>)  ·  **Round:** <n>
**In scope:** <k> question(s) (<skipped> deferred/exploratory skipped)

**Resolved & integrated:**
- Q<n> → <chosen option> — <one line> — logged in `<decisions.md | sprint file | design.md>`
- …

**Left open:**
- Q<n> (escalate-as-is) — <why it still needs you>
- Q<n> (route-to-design) — needs a `design-iterate` round: <why>

**Consistency overrides applied:** <list, or "none">
**Reviewer dissents surfaced:** <count, or "none">

_Next:_ <the `_Next:_` clause you wrote to Status>
```

### 9. Completeness assessment

Emit the standard round-end completeness assessment to chat — but only when this run integrated at least one decision (a zero-edit, all-left-open run already stopped at step 7 without advancing a round, so it emits no assessment). The block differs by doc type:

- **`design` file** — use the design-plan block ([design-plan.md § "Round-end completeness assessment"](../specs/design-plan.md)): `Verdict` + `Open-question tag counts`. Design plans have **no** `Threshold graded` field — do not invent one. Apply the literal completeness test: any `[blocks-v1]` / `[blocks-impl]` question still open (including an `escalate-as-is` you left for the user or a `route-to-design`) forces `not-yet-complete`.
- **`impl` file** — use the implementation-plan block ([implementation-plan.md § "Round-end completeness assessment"](../specs/implementation-plan.md)), which **does** carry a `Threshold graded` field: name `sprint-NN-execution-ready` for a sprint file, `plan-complete` for `overview.md`. Resolving questions rarely reaches `complete` on its own; grade honestly against what remains open after this run.

---

## Optional: step decomposition

Only when **all** of these hold: the user explicitly asked for it in their invocation; the target file is a **sprint file**; and — checked **after** integration and **after** appending any newly-surfaced sub-questions (the sub-question append in step 7) — the sprint file has **zero** `[blocks-impl]` / `[blocks-v1]` Open questions left. A question you left open this run (an unpicked `escalate-as-is`, a `route-to-design`) or a blocking sub-question a resolution just spun off is itself a blocker that **aborts** decomposition.

When the gate passes, decomposition runs **inside the same single round** as the integration — it does not open a second round. Concretely: do **not** append a second Status entry, and let the step-9 completeness assessment grade against `sprint-NN-execution-ready` (instead of the resolve-only grading). Follow [`impl-iterate.md`](./impl-iterate.md) § "Aggressive locking" (Step 2) and its Step 4–8 mechanics to populate the Locked Decisions table, Architecture notes, Public surface (when applicable), Implementation Steps with concrete Verification, the Step Dependency Chart, and the Acceptance checklist — then regenerate this sprint's section in `progress.md` and keep it in lockstep with the per-sprint Progress checklist. (This is the one case where the run touches `progress.md`.)

If the brake applies — locking would require inventing answers neither the design plan nor the user supplied — stop at integration, surface the gap as a new Open question, and tell the user. Do not fabricate locked decisions to satisfy a decomposition request.

If the conditions aren't all met (no explicit ask, not a sprint file, or a blocker remains), **stop at integration** and recommend the next move in the report's `_Next:_` (typically "run `impl-iterate` to lock this sprint").

---

## Failure modes the orchestrator must surface (not paper over)

- A resolver returning a malformed or missing YAML block — don't guess the proposal. Treat that one question as left-open / needs-re-dispatch, continue the run with the parseable proposals, and flag any sibling whose `depends_on` references the malformed question (its assumed upstream answer is now missing) rather than integrating that sibling against a guess.
- A resolver whose proposal contradicts a Locked Decision — flag in the presentation; don't silently integrate.
- A consistency conflict you can't cleanly resolve — surface it to the user as a flagged row, don't pick arbitrarily.
- A high rate of `route-to-design` verdicts — if most questions route back to design, the file graduated to its current layer too early; say so plainly (for a sprint file, the design plan likely has gaps; for an impl `overview.md`, likewise).
- Integration that would touch files outside the plan layer (code, `design.md` from an impl run) — refuse and surface.

For each, report the specific state so the user can act.

---

## Anti-patterns specific to this skill

- **Don't resolve questions yourself — ever.** Every in-scope question is dispatched to a resolver, regardless of how obvious it looks. (Restated from the Hard invariant because it's the failure mode.)
- **Don't read the codebase in the orchestrator context.** Investigation is the resolvers' job. You read the one target file and (at integration) the plan files you must edit.
- **Don't process more than one file per invocation.** "Resolve all open questions in the plan" is a sequence of single-file runs the user drives.
- **Don't operate on `[deferred]` / `[exploratory]` questions** unless the user asked for "all questions." They're punted on purpose.
- **Don't re-spawn resolvers during the tuning loop.** The user's pick overrides; only re-investigate on an explicit request.
- **Don't bury reviewer dissent or consistency overrides.** Both surface in the presentation and the report — they're the signal the user's veto runs on.
- **Don't auto-resolve an `escalate-as-is` the user didn't pick.** It stays open.
- **Don't edit `design.md` from an impl run**, or any file outside the plan layer. Route-to-design items are surfaced, not actioned here.
- **Don't decompose steps unless explicitly asked** and the file is a sprint file with blockers cleared.
- **Don't skip the Status entry, the completeness assessment, or the post-edit re-read/grep.** Integration is a real Trellis round and earns all three.
- **Don't leave `(resolved)` tags in Open questions.** Entries move to the Decisions log.
- **Don't double-number the round.** This run is at most one round and earns at most one `status.md` (or `design.md` Status) entry — and none at all if it integrated nothing. Re-read the Status log immediately before appending so a concurrently-landed round isn't overwritten or duplicated; don't run this skill concurrently with `impl-iterate` on the same `<root>`.

---

## Additional Instructions

The **Trellis instruction precedence chain** (`~/.trellis/instructions.md` → `<repo-root>/.trellis/instructions.md` → instructions supplied with this invocation) is applied by `SKILL.md` before this file is dispatched, so it is already in effect; honor it. Canonical statement — the single source shared by every instruction file and subagent brief: [instruction-precedence.md](../specs/instruction-precedence.md).
