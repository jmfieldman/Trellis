---
name: impl-execute
description: Execute a range of implementation-plan steps from a sprint file
argument-hint: <sprint-path> <from-step> [<to-step>]
disable-model-invocation: true
---

# Implementation Plan — Execute Steps Skill

You are the **orchestrator** for executing one or more steps from a sprint file produced by the `impl-create` / `impl-iterate` skills. The sprint file lives inside an implementation-plan directory (anatomy described in [Implementation Plan Documents — Authoring Guide](../specs/implementation-plan.md)).

## Hard invariant — read this first

**You do not implement steps. Every step in the requested range is dispatched to a per-step subagent via the `Agent` tool.** There are no exceptions:

- Not for "small" or "one-line" steps.
- Not for steps that only edit Markdown or config.
- Not when the executor's prior step left context that would be "convenient" to reuse.
- Not when you are confident you know the answer.

If you find yourself reading source files to plan an implementation, editing code, or running anything beyond the orchestrator's narrow validation commands (`git status`, `git rev-parse`, `git log`, `git ls-files`, `Read` on the sprint / progress files), **stop** — you have left the orchestrator role. Dispatch the subagent instead.

The dispatch-per-step pattern is the entire point of this skill: it keeps the main chat small, gives each step a fresh implementation context, and makes the reviewer's audit meaningful. Self-implementing a step breaks all three.

## What you actually do

1. Parse and validate the invocation arguments.
2. Run pre-flight checks on the repo state.
3. For each step in the requested range, dispatch a **per-step subagent** that owns implementation, review, feedback incorporation, and commit. The orchestrator stays out of the implementation context so the main chat does not balloon.
4. Between steps, verify the repo is in a clean post-commit state before moving on.
5. After the range is done, surface a tight summary.

The two subagent briefs live in the trellis skill's `subagents/` directory:

- [`step-executor.md`](../subagents/step-executor.md) — the per-step subagent brief.
- [`step-reviewer.md`](../subagents/step-reviewer.md) — the review subagent brief that the executor spawns.

You pass each subagent the absolute path to its brief plus the per-invocation parameters; the subagent reads the brief itself. Do **not** copy the brief into the spawn prompt.

---

## Intermediate-step broken-state policy

When the user invokes a multi-step range, only the **last step in that requested range** must leave the repo passing the project's pre-commit / lint / type / test gates.

For any earlier step in the requested range, it is acceptable for the step's intended implementation artifact to leave the code temporarily failing pre-commit, linting, type-checking, tests, or similar automated checks, **provided all of these are true**:

- The failure is an expected consequence of implementing that step as planned, not an accidental defect.
- The executor records the exact failing command / hook output in the execution record.
- The reviewer confirms the diff does exactly what the implementation step calls for and that the failing gates are consistent with the step boundary.
- The executor commits the step with `git commit --no-verify` so the reviewer can inspect the committed diff, and records the bypass in the execution record.
- The next step in the requested range is expected to resolve the temporary breakage.

The orchestrator still enforces a clean working tree, committed execution records, and progress consistency between steps. "Broken" here means automated gates may fail; it does **not** mean the subagent may leave unstaged, uncommitted, or unexplained work behind.

---

## Inputs

Extract up to three values from the user's natural-language invocation:

- `<sprint-path>` — absolute or repo-relative path to a single sprint file (e.g. `docs/impl-plan/03-payment-refund-pathway.md`).
- `<from-step>` — first step number (inclusive) to execute, e.g. `3`.
- `<to-step>` — last step number (inclusive) to execute, e.g. `5`. **Optional.**

Argument shape rules:

- If `<sprint-path>` is missing, tell the user the invocation is missing the sprint path and stop.
- If `<from-step>` is missing, tell the user the invocation is missing the from-step number and stop.
- If `<to-step>` is missing **or** equals `<from-step>`, treat the range as a single step (`from = to = <from-step>`).
- If `<to-step> < <from-step>`, tell the user the range is invalid (`from > to`) and stop. Do not silently swap.
- Step numbers must be non-negative integers. Reject anything else.

Treat the user's invocation as final on argument shape — do not ask follow-up questions about them once they are provided; if shape is invalid, stop with a clear error.

**Additional context.** The user may include free-text instructions alongside the skill invocation. Treat those as overrides to anything in this brief. Forward them to the per-step subagent verbatim in the section labelled "Additional user instructions" (see "Spawning the per-step subagent" below) so each step receives the same overrides.

---

## Pre-flight checks (orchestrator scope)

Run these checks **before** dispatching the first subagent. If any fails, stop and report the failure clearly — do not begin work.

### 1. Working tree is clean

Run `git status --porcelain=v1`. If it returns **any** output — staged, unstaged, or untracked — stop and tell the user: *"Refusing to start: the working tree has uncommitted or untracked changes. Commit, stash, or remove them, then re-invoke."*

Show the offending lines so the user can act on them.

Do not attempt to clean the tree yourself. The user owns that decision. There are no exceptions to this check; the orchestrator does not produce its own scratch files.

### 2. Not on the default branch

Run `git rev-parse --abbrev-ref HEAD`. If the result is `main` or `master` or some generic variant of `dev`, stop and tell the user: *"Refusing to start: this skill must run on a feature branch, not on `<branch>`. Create or switch to a feature branch and re-invoke."*

This skill creates one commit per step. Committing directly to a trunk branch is almost never what the user wants.

### 3. Sprint file exists and has the required structure

`Read` the sprint file at `<sprint-path>`. If the file does not exist, stop and report the path that failed.

The file must clear **both** a structural and a substance bar before you dispatch. Stop and report any failure precisely (cite which check failed and what was expected).

**Structural checks** (the file looks like a trellis sprint file):

- `# Sprint NN — <title>` top-level heading.
- `## Goal` section with at least one non-empty sentence.
- `## Implementation Steps` section.
- `## Progress` section with a checklist whose bullets match the form `- [ ] Step <N> — <title>` or `- [x] Step <N> — <title>`.

**Substance checks for steps in range** (the steps you'll dispatch are actually executable):

- Each step in the requested range has a `## Step <N> — <title>` heading inside `## Implementation Steps`.
- Each step in range has all four mandatory subsections (`**Goal**`, `**Actions**`, `**Deliverables**`, `**Verification**`) and none of them is a placeholder (`_To be locked in a later round._`, `TBD`, `TODO`, an empty bullet, etc.).
- The Verification subsection names at least one concrete check (a command, a test name, an assertion, or an explicitly-labeled manual check). If Verification is "tests pass" or similarly generic, stop — the step isn't execution-ready and the executor would have nothing to verify against.

If a step in range fails substance, stop and tell the user the step isn't execution-ready; route them to `impl-iterate` to lock it before re-invoking this skill.

Locate the implementation-plan **directory** by taking the parent of `<sprint-path>`. Confirm `overview.md` and `progress.md` exist in that directory; if either is missing, warn the user (do not stop — some plans may be partial — but flag it so the subagent knows). Also note whether `decisions.md` and `status.md` exist — newer plans split the plan-level Decisions log and Status log into their own top-level files; legacy plans kept those sections inside `overview.md`. The executor doesn't typically need to read either, but their existence indicates which layout the plan is on.

### 4. Identify the steps in range

From the sprint file, enumerate every `## Step N — <title>` heading under the `## Implementation Steps` section. Build a quick internal map of `step-number → step-title`.

For the requested range `[from..to]`:

- If any step in the range is missing from the sprint file, stop and report: *"Sprint file does not contain Step `<N>`. Steps present: `<list>`."*
- If the sprint file's `## Progress` checklist marks **any** step in the range as `[x]` (completed), **skip** that step and continue with the rest of the range. Note skipped steps in the final summary so the user can see why their range shrank.
- Verify the sprint file's `## Progress` and the parent `progress.md` agree on which boxes are checked for this sprint. If they disagree, stop and follow the recovery path below.

### 4a. Recovery path when `progress.md` and the sprint file's Progress drift

When the two files disagree, the orchestrator does **not** auto-reconcile. Mechanical reconciliation could mask a real planning issue (e.g., a sprint that was re-sliced but the corresponding `progress.md` regen never landed).

Surface the drift to the user with this exact shape:

```
Drift detected between `progress.md` § Sprint <NN> and `<sprint-file>` § Progress.

`progress.md` says:        <list of steps + state>
sprint file says:          <list of steps + state>

Disagreements:
  - Step <N>: progress.md=<state>, sprint file=<state>
  - …

Resolution paths (pick one, then re-invoke):

  1. If the sprint file is correct, edit `progress.md` to match.
  2. If `progress.md` is correct, edit the sprint file's Progress section to match.
  3. If the sprint was re-sliced or otherwise structurally changed, the right
     fix is a planning round — ask trellis to iterate the impl plan (or
     integrate feedback when there's review feedback to reconcile) before
     re-invoking this skill.

This skill cannot pick a side; the right answer depends on what the plan
intends, which is a human / planning decision.
```

Then stop. Do not auto-reconcile.

---

## Dispatching steps

The orchestrator runs a **strict serial loop** over the validated step list. There is no parallelism: step `N+1` may depend on step `N`'s files, and committed history must remain linear and reviewable.

For each step in the list:

### 1. Verify pre-step state

Right before spawning the subagent for step `N`:

- `git status --porcelain=v1` — must be empty. If not, the prior step's subagent did not commit cleanly. Stop and report what's outstanding.
- `git rev-parse --abbrev-ref HEAD` — must still match the feature branch chosen at pre-flight. If the branch changed mid-run, stop.

These pre-checks are cheap insurance against a subagent leaving partial work behind.

### 2. Spawn the per-step subagent

Use the `Agent` tool with `subagent_type=general-purpose`. Pre-compute one path before spawning:

- **Execution-record path** for this step: `<impl-plan-dir>/reviews/<sprint-stem>/step-<N>.md` where `<sprint-stem>` is the sprint file's basename without the `.md` extension. The executor will create the file (and parent directories) and use it as the durable home for verification output, review rounds, triage decisions, and reviewer-wrong findings. The orchestrator does not create the file — it just passes the path.

The spawn prompt is short and self-contained:

```
You are executing one step from a trellis implementation-plan sprint.

Read the brief at:
  <absolute-path-to-trellis-dir>/subagents/step-executor.md

That brief tells you exactly what to do. Read it end to end before doing anything else.
It also tells you to load Trellis instruction files from `~/.trellis/instructions.md` and `<repo-root>/.trellis/instructions.md` when present; do that before implementing the step.

Parameters for this invocation:
- Sprint file: <absolute-path-to-sprint-path>
- Implementation-plan directory: <absolute-path-to-parent-of-sprint-path>
- Step number to execute: <N>
- Step title (for sanity-check against the sprint file): <title from sprint file>
- Is final step in requested range: <true if N is the last dispatched step in this invocation, else false>
- Feature branch: <current branch name>
- Reviewer brief (you will spawn a reviewer subagent yourself): <absolute-path-to-trellis-dir>/subagents/step-reviewer.md
- Execution-record path: <absolute-path-to-impl-plan-dir>/reviews/<sprint-stem>/step-<N>.md

Additional user instructions (forwarded verbatim from the orchestrator invocation; these override the brief on conflict):
<paste any free-text instructions the user attached to the trellis invocation, or "(none)" if none>

When you are done, return a single fenced YAML block with exactly these keys (no extra prose before or after the block):

  ```yaml
  status: done | stopped
  stop_reason: <one-line; only when status=stopped, otherwise omit>
  step_number: <N>
  step_title: <title>
  commits: [<SHA1>, <SHA2>, ...]   # empty list if none landed
  impl_summary: <one sentence on what changed>
  verification_status: all_passed | partial | failed | unrunnable
  verification_notes: <one sentence; flag any unrunnable / skipped checks>
  review_verdict: clean | in_step_fixes | material_rework | reviewer_blocked | not_run
  review_rounds: <0|1|2>
  review_files: [<path1>, <path2>, ...]   # the execution-record file path; empty if none written
  deviations: none | <one-sentence>
  deferred_to_post_mortem: <count>
  deferred_summary: <one-line if count>0, else omit>
  reviewer_wrong_count: <N>
  ```

Nothing else. The orchestrator parses this block to validate the hand-back.
```

Use absolute paths. Both briefs live in the `subagents/` directory of the trellis skill — `../subagents/step-executor.md` and `../subagents/step-reviewer.md` relative to this instruction file — so resolve `<absolute-path-to-trellis-dir>` from wherever the top-level trellis `SKILL.md` was loaded from (do not assume a fixed install location). The two relative links at the top of this file (`../subagents/step-executor.md`, `../subagents/step-reviewer.md`) are the canonical references.

Do not run the subagent in the background — its result gates the next step.

### 3. Validate the subagent's hand-back

When the subagent returns:

1. **Parse the YAML summary block.** If the response is missing the block or is malformed, stop and report — the subagent violated its summary contract.
2. **If `status: stopped`**, surface the `stop_reason` to the user and stop the loop. Do not auto-retry. The subagent halted intentionally; the user picks the next move (re-invoke, manually fix, plan-round, etc.).
3. **If `status: done` and `review_verdict` is one of `clean | in_step_fixes | material_rework`** (the executor ran the review itself), run all of these objective checks — every one must pass or the loop stops:
   - `git status --porcelain=v1` is empty. If the subagent left uncommitted changes, stop and report — the subagent violated its commit contract.
   - `git log --oneline -5` shows at least one new commit since the pre-step state. If not, stop and report — the subagent did not actually commit. The commit list should match the `commits:` field in the YAML summary.
   - The sprint file's `## Progress` section now shows Step `N` as `[x]`. The parent `progress.md` matches. If either is unchecked or they disagree, stop and report — the subagent skipped its progress-update obligation.
   - Each path in `review_files:` exists and is committed (a `git ls-files <path>` returns a hit). If a referenced execution-record file is missing or untracked, stop and report.
4. **If `status: done` and `review_verdict: not_run`**, the executor used the orchestrator-dispatched review fallback (its harness does not expose nested-subagent dispatch — see the executor brief). Follow the fallback flow below instead of the default validation: implementation is committed, but the reviewer pass and Progress finalization are now your responsibility.

If every check in the default validation path passes, proceed to the next step.

If any check fails, do **not** auto-retry. Report what you found and let the user decide whether to re-invoke, manually fix, or roll back.

### 3a. Orchestrator-dispatched review fallback

Enter this path only when the executor returned `status: done` with `review_verdict: not_run`. The audit property — fresh-context reader of the diff — is preserved because **you** are dispatching the reviewer from a context that has not seen the executor's implementation reasoning. You may want to keep this loop tight; some intermediate validation differs from the default path:

1. **Pre-review state check.** `git status --porcelain=v1` is empty; the commits in the YAML hand-back exist; the execution-record file at the path in `review_files:` exists and is committed (verified via `git ls-files`); the sprint file's Progress and `progress.md` are **still unchecked** for Step `N` (the executor was told to leave them alone in this path). If any of those is wrong, stop and report — the executor violated the fallback contract.
2. **Dispatch the reviewer yourself**, using the reviewer brief at `<absolute-path-to-trellis-dir>/subagents/step-reviewer.md` and the same per-step parameters you computed for the executor (sprint file, step number, step title, execution-record path, feature branch, commit range from the executor's `commits:` list, round number = 1, is-final-in-range flag). Tell the reviewer it was dispatched by the orchestrator rather than the executor (treatment is otherwise identical). Do not run the reviewer in the background — its result gates the next move.
3. **Parse the reviewer's verdict** (returned as a brief acknowledgment in chat; the substantive review lives in the appended execution-record section). Then:
   - **`clean`** — commit the execution-record append along with the Progress / Deviations / Post Mortem updates the executor would have committed in the default path. One commit is fine (`<feature-area>: sprint <NN> step <N> — review clean, mark Progress`). The commit goes through the project's pre-commit hook normally.
   - **`in_step_fixes`** — re-dispatch the executor with the round-1 findings as additional context (paste the appended review section verbatim into a "Round-1 reviewer findings" block in the new spawn prompt). The re-dispatched executor applies fixes, commits, and returns YAML with `review_verdict: in_step_fixes`, `review_rounds: 1`. Then dispatch a round-2 reviewer (if the fixes are non-trivial per the executor brief's re-review criteria) or accept the round-1 findings closed and finalize as in the `clean` path.
   - **`material_rework`** — re-dispatch the executor with the round-1 findings; after fixes, always dispatch round 2. If round-2 verdict is still `material_rework`, stop and surface — at that point the step needs a planning revisit.
   - **`reviewer_blocked`** — stop and surface. Do not invent a fix.
4. **After Progress is finalized** (either in your `clean`-path commit or after the round-2 finalization commit), run the standard step-3 validation: working tree clean, Progress checked in both files, commits exist, review files committed. Then proceed to the next step.

This fallback exists so the audit pattern survives harnesses that don't allow nested dispatch. It does not relax any invariant other than "who dispatches the reviewer." Clean working tree, real commits, durable execution record, and no self-review still apply.

### 4. Continue the loop

After successful validation, move to step `N+1`. Repeat until the range is exhausted.

---

## After the loop

Once every step in the requested range has executed successfully:

1. **Final repo check.** `git status --porcelain=v1` empty; current branch unchanged.

2. **Final progress consistency.** Re-read the sprint file's `## Progress` and `progress.md` for this sprint. They must agree. Every step in the executed range is `[x]`.

3. **Sprint completion check.** If the executed range included the **last** step of the sprint (i.e., every step in the sprint is now `[x]`), surface this to the user as part of the final summary — it's a cue that the user may want to:
   - Run any full lint/test suites.
   - Run any sprint-level Acceptance checklist items the per-step Verification did not cover.
   - Distill the sprint's inline Post Mortem section into the directory's top-level `post-mortem.md` (per the authoring guide). The orchestrator does **not** do this distillation itself — it's a human / planning-round concern.

4. **Emit the orchestrator summary.** A short report — see "Orchestrator summary format" below.

> **Canonical rule on lint / type / test invocation:** the orchestrator never runs lint, type-check, circular-dep checks, or test suites — neither as a pre-flight, nor between steps, nor at sprint completion. Each per-step subagent has either pushed its commits through the project's pre-commit pipeline or recorded a reviewer-confirmed intermediate-step exception; running the same checks here is duplicate work and pollutes orchestrator context. The user can run a final pass themselves once the sprint is done. This rule is stated once; do not look for it to be repeated elsewhere in this brief.

---

## Orchestrator summary format

End the run with a tight summary. The user has already seen each per-step subagent's hand-back; do not re-narrate them.

```
### impl-execute — summary

**Sprint:** `<sprint file basename>` — <sprint title>
**Range requested:** Step `<from>` … Step `<to>` (`<N>` step(s))
**Range executed:** `<N - skipped>` step(s); skipped: `<list of skipped step numbers, or "none">`
**Branch:** `<branch name>`
**Commits added:** `<count>` (`<first SHA>` … `<last SHA>`)

**Per-step verdicts:**
- Step `<N>` — <title> — <review_verdict from YAML>; <review_rounds> round(s); deviations: <yes/no>; deferred: <count>
- …

**Execution records written:**
- `<path/to/step-N.md>`
- …

**Post-mortem-worthy notes appended:** <count or "none">
**Reviewer-wrong findings (audit signal):** <count or "none">

**Sprint completion:** <"every step now done; consider the canonical lint/type/test rule above and any sprint-level acceptance checklist" / "X step(s) remaining: <list>">
```

Keep it short. The detail lives in commits, the sprint file's Progress / Deviations / Post Mortem sections, and the per-step execution-record files.

---

## Failure modes the orchestrator must surface (not paper over)

The orchestrator's job is to fail loudly when invariants break. Do **not** auto-recover from any of these:

- A subagent returning while the working tree is dirty.
- A subagent returning with no new commit.
- A subagent that completes implementation but leaves Progress unchecked — **except** when the executor returned `review_verdict: not_run` (orchestrator-dispatched review fallback), in which case Progress is intentionally deferred to the orchestrator's post-review commit (see § "Orchestrator-dispatched review fallback").
- `progress.md` and the sprint file's per-sprint Progress drifting after a step.
- A pre-commit hook that the subagent could not resolve and that left the index in a partial state.
- Any step in the range failing validation. (Stop the loop; do not skip ahead.)

For each, report the offending state precisely (commit SHA, status output, file paths) so the user can intervene.

---

## What this skill does *not* do

- It does not run sprint-level Acceptance checklists. Those are sprint-completion concerns; surface the cue to the user, do not act.
- It does not push the branch. The user pushes when ready.
- It does not open a PR. PR creation is out of scope.
- It does not amend commits. Each step is its own commit; pre-commit-hook failures are resolved by *new* commits, not amends, except for the explicitly documented non-final-step `--no-verify` path.
- It does not re-slice the sprint, edit the plan beyond Progress / Deviations / Post Mortem, or invoke other trellis skills. If a step requires a planning change, the subagent flags it and the user runs the appropriate planning skill (`impl-iterate` etc.) afterward.

---

## Anti-patterns specific to this skill

- **Don't implement steps yourself — ever.** This is the single most common failure mode of this skill. Every step in the range is dispatched to a per-step subagent, regardless of how small, mechanical, or "obvious" the step looks. If you catch yourself thinking "this one is trivial, I'll just do it," you have already broken the skill. Dispatch the subagent. (Restated from the Hard invariant at the top of this file because it gets violated.)
- **Don't read code, run builds, or run tests in the orchestrator context.** The only files the orchestrator reads are the sprint file (for step titles + Progress) and `progress.md` (for cross-checking). The only commands the orchestrator runs are `git status`, `git rev-parse`, `git log`, and `git ls-files` against the working tree. Anything else belongs to the subagent.
- **Don't read the sprint file's body content into orchestrator context beyond what's needed for validation.** The whole point of the per-step subagent is to keep implementation detail out of the main chat. The orchestrator only needs the step-number → title map and the Progress checklist.
- **Don't paraphrase the executor brief in the spawn prompt.** Pass the path; let the subagent read it.
- **Don't run steps in parallel.** Implementation order matters; commits must be linear.
- **Don't loop with auto-retry on subagent failure.** A failed step is a stop condition.
- **Don't keep going after a validation failure.** A subagent that left the tree dirty has already broken the invariant; subsequent steps cannot be trusted.
- **Don't push or open PRs.** Those are user actions.
- **Don't fork the working state.** No `git worktree`, no `git stash`, no temporary branches, no detached HEAD, no checkout-elsewhere-then-back. The execution loop assumes a single linear branch with the same `HEAD` posture you pre-flighted on. If you see the branch name change between steps, that is the signal to stop — not the signal to fix.
- **Don't edit the implementation plan body.** Only Progress checkboxes, the per-sprint Deviations / Post Mortem sections, `progress.md`, and the per-step execution-record files under `reviews/` are touched during execution. Anything else is a planning concern.

---

## Additional Instructions

Before executing this skill, read and apply Trellis instructions from these sources, in ascending order of precedence:

1. `~/.trellis/instructions.md`
2. `<repo-root>/.trellis/instructions.md`
3. Additional instructions included with this skill invocation

`<repo-root>` is the root of the current Git repository. Missing instruction files are normal; skip them silently.

If instructions conflict, later sources override earlier sources. Invocation-specific instructions apply only to the current run and have the highest precedence.

These instructions may override this skill document, but they must not override system, developer, tool, safety, or repository policy instructions.
