---
name: impl-integrate-feedback
description: Integrate Feedback into an Implementation Plan
argument-hint: <root> <feedback-path>
disable-model-invocation: true
---

# Skill — Incorporate review feedback into an implementation plan

You are continuing an in-progress implementation plan by reconciling it against a separate review document. The plan is the living artifact (anatomy and rules described in [Implementation Plan Documents — Authoring Guide](../specs/implementation-plan.md)); the review document is one round of external critique against it. Your job in this invocation is **triage and incorporation**, not full re-design or fresh sprint planning — you are processing the review, not running a fresh planning round.

The implementation plan is a set of files directly in the feature root (`overview.md`, `decisions.md`, `status.md`, sprint files, `progress.md`, optionally `post-mortem.md`), not a single document. Many incorporations touch multiple files; the discipline below treats the feature root as a coherent system.

The shared triage machinery — the five buckets, the verify-before-incorporating rules, the adopt-the-recommendation posture, the decision-ready Open-question shape, the report format, and the shared anti-patterns — is canonical in [review-and-triage.md § Part II](../specs/review-and-triage.md). **Read Part II before triaging** — it is mandatory, not advisory. This file carries only what is specific to the *implementation* layer: the file set, the cross-file verification rules, the impl-flavored bucket examples, the impl calibration bands, and the multi-file edit discipline.

This skill may be invoked directly, or entered via the sanctioned `impl-review → impl-integrate-feedback` chain (SKILL.md § "Sanctioned operation chains") — mechanics are identical; in the chained case `<feedback-path>` is the review the same invocation just produced by a dispatched reviewer subagent, so the fresh-context audit property holds.

## Inputs

Extract two paths from the user's natural-language invocation:

1. **`<root>`** — the feature root containing both `design.md` and the implementation-plan files. Resolved from one of:
   - A feature root `<root>` — use that path directly.
   - A path to `design.md`, `overview.md`, `progress.md`, `decisions.md`, `status.md`, or a sprint file — use its parent directory as `<root>`.
   - If neither holds, ask the user and stop.
2. **`<feedback-path>`** — a review of that plan.

The review is typically organized as: Executive summary, High-level concerns, Cross-sprint coherence, Per-sprint findings (one block per sprint), optional Per-step findings, Substance/authoring-guide deviations, Minor nits, Suggested next-round focus. Each item typically has a *Where*, *Concern*, *Why it matters*, and *Suggested directions* (often a menu of options).

If either is missing, ask the user for it and stop.

## Scope of this skill

Per [review-and-triage.md § "What the integrator is (and is not) doing"](../specs/review-and-triage.md). At the implementation layer that means:

- You read **every implementation-plan file in the feature root** before doing anything else, and you edit the plan's body (`overview.md`, sprint files, `progress.md`), Decisions logs (plan-level `decisions.md` + per-sprint Decisions log sections), Open questions (overview-level + per-sprint), and Status logs (plan-level `status.md` + per-sprint Status / Feedback-incorporated sections).
- You keep `progress.md` and per-sprint Progress sections in lockstep when step structure changes.
- You do **not** re-slice the sprint roster on your own. If a review item recommends a split / fold / re-order, file it as an Open question (or in the report) — re-slicing is a planning-round move, not a triage move. The single exception: a review item that flags a clear naming or numbering bug (e.g., a sprint file that doesn't match the roster) can be fixed mechanically.
- You do **not** edit the source design plan. If a review item routes back to a design-level call, surface it in the report — don't touch the design plan from here.
- You do **not** flesh out a stub sprint to execution-ready. If a review item demands fresh sprint-step decomposition, file it as an Open question for the relevant sprint.

## Verify before incorporating — impl-layer additions

Follow [review-and-triage.md § "Verify before incorporating"](../specs/review-and-triage.md) in full. For impl plans verification is *especially* load-bearing because the cross-sprint coherence axis (the highest-leverage axis in the review) requires the reviewer to read two or more files in lockstep — a hard task that's easy to get subtly wrong. Impl-specific failure modes, on top of the shared ones:

- The reviewer **misread one of the two files** in a cross-sprint claim — they say "Sprint 04's Prerequisites lists `X` but Sprint 02's Deliverables don't include it," but Sprint 02 actually does include it (under a different heading or with slightly different wording the reviewer didn't recognize as the same thing).
- The reviewer **conflated the overview's Locked Decisions with a sprint's Locked Decisions** — they flag a "contradiction" that's actually a refinement (the sprint pinning a value the overview left general).
- The reviewer **counted wrong** — they say "Sprint 06's Implementation Steps lists 8 steps but `progress.md` lists 6," but a quick recount shows both have the same number under different formatting.
- The reviewer **flagged stale wording that was already purged** — they're reading an older copy of the plan than what's in the feature root.

Impl-specific verification requirements, on top of the shared ones:

1. **Confirm cross-file claims by reading both files.** When the review item asserts a contradiction or drift between two files (Prerequisite vs. Deliverable, overview Locked Decision vs. sprint Locked Decision, `progress.md` vs. per-sprint Progress, sprint-A surface vs. sprint-B surface), open both files and confirm the claim. Cross-file claims are the highest-error category — verify them with extra care.
2. **Re-count when the review counts.** If the review claims "Sprint NN has 8 steps but `progress.md` has 6," count both yourself before acting.
3. **Verification is also required for Minor-incorporate items that span multiple files** (e.g., a name-drift fix in Sprint 04's Prerequisites that mirrors a Deliverable in Sprint 02). Even though the edit is mechanical, the *premise* — that the two files actually disagree — is a cross-file claim and must be confirmed. Single-file Minor incorporates skip verification per the shared spec.

For adopted reviewer recommendations, the shared reasoning check applies with an impl flavor: a material consideration the reviewer missed is often a cross-sprint dependency or a locked decision the recommendation collides with.

The shared stop-early rules apply unchanged (leave applied edits in place; record the partial pass in `status.md`; the count includes cross-file Minor-incorporate items).

## The triage rule — impl-layer specifics

The five buckets and their semantics are canonical in [review-and-triage.md § "The five buckets"](../specs/review-and-triage.md). Impl-layer specifics:

### Items that typically land in Incorporate

A contradiction between a Locked-Decisions row and a Public-surface error table (you have to pick a side); a missing `progress.md` entry for a sprint already in the roster *with steps the sprint file enumerates*; a foundational invariant restated in `overview.md`'s Architectural-invariants list when an existing sprint already verifies it but the list was missing the entry; a judgment-call finding (a sprint-level locked decision, an idempotency-key shape, an API error code, a contract clause, a sprint dependency rewire) where the reviewer recommended an option whose reasoning survives verification.

Two impl-specific exclusions: the fix must not require **re-slicing the sprint roster** (renaming a sprint file to match its roster title is mechanical and lives in Minor incorporate; splitting a sprint is a planning move), and Decisions-log entries route by scope (cross-cutting calls in `decisions.md`; sprint-scoped calls in the affected sprint file's Decisions log section).

### Items that typically land in Minor incorporate

- A typo, casing fix, or canonical-term swap where `overview.md`'s framing block already settled the term.
- A marketing-language purge the authoring guide explicitly flags.
- Removing a `(resolved)` tag inside Open questions and moving the entry to the Decisions log under its original round tag.
- Deleting stale wording from a superseded round still sitting in any file.
- Adding a missing forward-link in an Out-of-scope entry that already names the target sprint elsewhere.
- Adding a missing entry in `overview.md`'s Cross-references / dependency-graph one-liner list for an edge already drawn in the chart.
- Adding a missing entry in `overview.md`'s Module/Directory tree for a file an existing sprint already commits to creating.
- Splitting a bundled Open question into two for clarity (preserves both, no decision made).
- Correcting a sprint Decisions-log bullet that drops a clause the body already decided.
- Repairing numbering (skipped Q5; duplicate `(R3)` tag; out-of-order Status entries; sprint roster row missing for an existing file).
- Reconciling `progress.md` ↔ per-sprint Progress drift (the master file is the source of truth; the per-sprint copy is reconciled to it, or the master is fixed if it's the broken side).
- Correcting a `post-mortem.md` violation that's mechanical (missing section heading for a shipped sprint; section out of sprint order). *Note*: deleting a `post-mortem.md` that was pre-created at scaffold time is a Minor incorporate; creating one when no sprint has shipped is forbidden — see Authorization scope.
- Renaming a sprint file to match the title already used in `overview.md`'s sprint roster.
- A `status.md` entry that omits the supersession callout for a re-slice it announces (add the callout, don't reopen the slice).

### Impl-specific Open-question signals

Beyond the shared signals, two impl-layer planning moves always file as Open questions regardless of how well the reviewer reasoned them: **"re-slice these sprints"** and **"promote / demote between open question and locked decision."** When filing, pick the **right scope** — overview-level for cross-sprint / feature-wide questions; sprint-level for sprint-scoped ones. When in doubt, prefer the sprint level (lower scope is cheaper to relocate later). Also include: whether it blocks anything currently in scope, and the sprint(s) it touches.

### Calibration bands (impl plan)

**These bands assume a review with ≳8 distinct items** (see the shared spec for the small-review caveat). Impl-plan reviews surface a high concentration of cross-sprint coherence drift, most of which is mechanical — so the Minor bucket runs heaviest. A healthy impl-plan triage is roughly:

- **10–25% Incorporate (full)** — material edits that earn a Decisions-log entry (cross-cutting calls in `decisions.md`; sprint-scoped calls in the affected sprint file's Decisions log section), including adopted reviewer recommendations on shape questions.
- **40–60% Minor incorporate** — cross-sprint coherence fixes plus the usual mechanical fixes. This bucket runs heaviest for impl plans because the multi-file shape produces many seam-drift findings.
- **5–15% Open question** — only the genuinely-needs-a-human residue, plus re-slicing / promotion-demotion planning moves. If this bucket is large, you are probably being too conservative — re-check that you adopted every sound recommendation.
- **0–10% Reviewer-wrong** — a small but non-zero rate is normal, especially on cross-file coherence claims (the highest-error category for reviewer agents). Above ~20% → stop, per the shared spec; when the failures concentrate in cross-sprint coherence findings, call that out to the user separately.
- **5–15% Ignore** — duplicate / out-of-scope / over-engineering / authoring-guide-contradicting / stylistic / non-planning-doc items.

Combined Incorporate + Minor incorporate is typically 50–85%.

Impl-specific out-of-band signals (the shared spec carries the Open-question and Reviewer-wrong signals):

- If **Open question is holding more than ~20% of items**, re-triage the boundary cases per the shared spec's rule (most reviewer-surfaced wording / consistency / cross-sprint-coherence drift belongs in Minor incorporate).
- If **Incorporate (full) is holding more than ~30%**, re-check whether you're making fresh design or sprint-shape calls the reviewer did *not* recommend. Also check you're not promoting Minor incorporates.
- If **Minor incorporate is holding more than ~70%**, the review may be flailing — heavy on micro-coherence findings without a load-bearing shape concern. The triage is fine, but flag the pattern in the report — the user may want a different review prompt next time, or to skip the next review entirely if the plan has stabilized.

## Edit discipline

### Multi-file incorporations (the high-risk path — read this first)

Most cross-sprint-coherence findings are multi-file by nature. They are also the highest-error path in this skill: a Prerequisite renamed in Sprint 04 but left untouched in the consuming Sprint 06 produces a *silently-broken* plan — every file looks internally coherent, only the seam between them is wrong. The seam is invisible to a reviewer reading any single file. Treat multi-file incorporations as their own discipline.

The core mechanic is [implementation-plan-mechanics.md § "Multi-file edit discipline"](../specs/implementation-plan-mechanics.md): **list the file cluster** before editing any of it (which file owns the canonical wording; which files reference it; whether `overview.md` / `decisions.md` / `status.md` / `progress.md` and the per-sprint Progress section need updating), **edit the canonical-owner first and consumers second**, then **grep the feature root for the rejected wording** (`grep -rn '<old wording>' <root>` — zero hits is the only acceptable result; a hit means a consumer was missed). Editing a consumer before its owner opens a window where both old and new wording coexist, and an incomplete pass leaves the plan in that broken state. The worked examples below show what that discipline looks like for the most common impl-plan incorporation shapes.

#### Worked example 1: Prerequisite ↔ Deliverable name drift

Reviewer finding: *"Sprint 04's Prerequisites lists `social.canInvite` but Sprint 02's Deliverables call it `social.checkInvitePermission`. Pick one."*

File cluster:
- `02-social-helpers.md` — owns the Deliverable.
- `04-conversation-create.md` — owns the Prerequisite.
- `overview.md` — dependency-graph one-liner ("02 → 04: invite-permission predicate") may name the helper.
- Possibly `progress.md` if a step in Sprint 02 mentions the helper by name.

Order:
1. Decide canonical name (one option, no menu — this is a Minor incorporate, not a design call).
2. Update Sprint 02's Deliverables list + any inline mention.
3. Update Sprint 04's Prerequisites list.
4. Update `overview.md`'s dependency-graph edge label, if it named the helper.
5. Update any Sprint 02 step heading or `progress.md` row that referenced the old name.
6. Grep the feature root for the rejected name. Zero hits expected.

#### Worked example 2: a sprint adds a file the overview's module-tree doesn't predict

Reviewer finding: *"Sprint 03 introduces `src/services/measurement/managers/measurement-resolver.ts` but `overview.md` § Module/Directory layout doesn't list it."*

File cluster:
- `03-manager-core.md` — already names the file in its Deliverables / Steps. No edit needed there.
- `overview.md` § Module/Directory layout — needs the file added in the right tree position.

Order:
1. Add the file to `overview.md`'s tree.
2. Confirm Sprint 03's name for the file matches exactly (canonical wording lives in the sprint that ships it, since the sprint is closer to the actual code).
3. Grep the feature root for any other reference to the same module — sometimes a downstream sprint references the file too.

#### Worked example 3: `progress.md` ↔ per-sprint Progress drift

Reviewer finding: *"Sprint 06's Implementation Steps lists 8 steps but `progress.md` § Sprint 06 lists only 6."*

File cluster:
- `06-resolve-type.md` — owns Implementation Steps + per-sprint Progress section.
- `progress.md` — owns the master checklist.

Order:
1. Decide which surface is correct. The Implementation Steps section is the source of truth for *what work is being done*; `progress.md` and the per-sprint Progress section both mirror it. (If `progress.md` is correct because two steps were folded but the body wasn't updated, that's the rare reverse case — fix the body.)
2. Reconcile the per-sprint Progress section in `06-resolve-type.md` to match the Implementation Steps.
3. Reconcile `progress.md` § Sprint 06 to match.
4. Grep both files for the missing step labels to confirm they now appear identically.

#### Authorization scope on multi-file edits

A multi-file edit touches more files but does *not* expand the skill's authorization. Edits stay within the implementation-plan docs in the feature root. If the cluster names code, configs, conventions files, or another non-planning file, surface that in the report as an out-of-scope action — do not act on it.

---

### Single-file edit discipline

When you do incorporate (whether single-file or multi-file):

- **Edit the relevant body section directly**, in whichever file owns it. Do not leave both old and new wording side-by-side; purge the obsolete phrasing per the authoring guide's supersession rule.
- **Keep `progress.md` and per-sprint Progress in lockstep.** If a step is renamed, re-ordered, added, or removed, update both surfaces in the same pass. The master `progress.md` wins on conflict; the per-sprint section is reconciled to it (unless the master is the broken one, in which case correct the master).
- **Add a tagged bullet to the relevant Decisions log** for each material incorporation, using the next round number (`(R<n>)`). Cross-cutting calls go in the feature root's **`decisions.md`** (not a section inside `overview.md`); sprint-scoped calls go in the affected sprint file's Decisions log section. For pure typo / wording / structural-cleanup edits, a Decisions-log entry is optional — but for anything that names a call the plan now relies on, log it. Lead with a bold phrase that names the call.
- **Compress older Decisions-log entries** at every scope this pass touched — walk both `decisions.md` and the affected sprint files' Decisions log sections, condensing any entry this pass (or an earlier round) superseded plus any whose details have gone stale, per [implementation-plan.md § "`decisions.md` anatomy"](../specs/implementation-plan.md). Keep the last 2–3 rounds and any still-load-bearing entry in full.
- **Update Open questions**: remove an entry if the incorporation resolves it (rare in this skill — usually means you should have triaged it as Open question instead); add an entry to the right scope (overview vs. sprint) for each item triaged into the Open-question bucket.
- **Append exactly one new Status entry** in **`status.md`** for this incorporation pass. Format: `**Round <n>**: incorporated review feedback — <one-line summary spanning the most material edits>; <count> items added to Open questions; <count> items declined. _Next:_ <one-line recommended focus for the next planning round, citing Open question IDs + tags + scope where relevant>.` Round number is the plan's next integer at the plan level (per `status.md`). Typical `_Next:_` patterns: `lock Sprint <NN>`, `resolve overview Q<n> [blocks-v1] before touching Sprint <NN>`, `re-slice Sprints <06>/<07> — seam unstable`, or `hand off to implementation` when no load-bearing items remain. If the incorporation supersedes earlier-round wording in any sprint, call out the supersession explicitly in the same `status.md` line ("R3's helper-in-Sprint-06 wording superseded").
- **Sprint-level Status / Deviations.** If a sprint file has its own Status / "Feedback incorporated" section, append a tagged entry there too for sprint-scoped incorporations (in addition to the entry in `status.md`). If a sprint shipped before this review (its final step is `[x]`), prefer the "Feedback incorporated (post-review)" / "Deviations applied during implementation" surfaces over editing the body of an already-shipped sprint — the shipped code is the source of truth, and the doc captures the divergence rather than pretending it didn't happen.
- **Compress older Status entries** that no longer carry weight — after appending, walk both `status.md` and any sprint-level Status / Feedback-incorporated logs you just appended to, per [implementation-plan.md § "`status.md` anatomy"](../specs/implementation-plan.md). Keep the last 2–3 rounds in full at each scope; never delete a round outright — round numbering stays contiguous and grep-able. For a *shipped* sprint, leave its older Status / Deviations entries alone — they are part of the historical record of what shipped and are no longer eligible for compression by a planning pass.
- **Re-read affected files** after editing to catch wording from earlier rounds that your edits silently invalidated. If you find any, fix them in the same pass and note the supersession in the Status entry.
- **Preserve cross-file consistency.** If incorporating one review item creates a tension with another part of the plan you didn't touch (a sibling sprint that no longer agrees, a Prerequisite that no longer matches a renamed Deliverable), either resolve it by extending the edit or file the new tension as an Open question. Do not leave the plan internally contradictory.
- **Honor authoring conventions.** Bold the decision lead in Decisions-log entries. Tag every Decisions-log bullet with `(R<n>)`. Use canonical terms. No marketing words. Identifiers in backticks. Absolute dates only. File paths real, not invented.

When you file an Open question, follow [review-and-triage.md § "Filing a decision-ready Open question"](../specs/review-and-triage.md), at the right scope per the impl-specific signals above.

When you ignore: do not edit the plan. Track the item only in your final chat report.

## Authorization scope for this skill

This skill edits an implementation plan in place — across `overview.md`, `decisions.md`, `status.md`, every sprint file, `progress.md`, and (when present) `post-mortem.md` within the feature root. That is the explicit purpose, so you do not need additional confirmation for body edits, `decisions.md` / sprint Decisions-log additions, Open-questions additions, `status.md` / sprint Status appends, or `progress.md` reconciliation *within those implementation-plan docs*. You **do** need confirmation before:

- Editing `design.md`, code, configs, sibling docs, or project-level conventions files.
- Renaming plan files (sprint file renames, overview-file renames, etc. — propose, don't act; the one exception is the mechanical roster-title rename in Minor incorporate).
- Re-slicing the sprint roster (splitting a file, folding two files together, re-numbering). These are planning-round moves; surface them in the report or as Open questions instead.
- Touching code, schemas, migrations, or anything outside the implementation-plan docs in the feature root.
- Creating `post-mortem.md` if no sprint has shipped yet (the file appears the first time a sprint ships, not at triage time). If a review item asks for a `post-mortem.md` entry but no sprint has shipped, decline it and explain.

If a review item recommends changes outside the implementation-plan docs in the feature root, treat those as Open questions or surface them in the final chat report; do not act on them.

## Workflow

1. **Read every implementation-plan file in the feature root, end-to-end.** No skimming. `overview.md` first; then `decisions.md`; then `status.md`; then `progress.md`; then each sprint file in numeric order; then `post-mortem.md` if present. Note the plan's current Round count in `status.md`, the latest Decisions-log round tags (in `decisions.md` and in each sprint's Decisions log section), the current Open-questions numbering at each scope, and which sprints have shipped per `progress.md`.
2. **Read the review document end-to-end.** Note its overall structure and which sections you'll process.
3. **Enumerate the review items.** Most reviews are already itemized (numbered concerns / nits). Treat each numbered item as one unit. If a review item bundles multiple sub-points, split it into sub-units before triaging — each sub-unit gets its own bucket. Per-sprint and per-step findings are each their own units.
4. **First-pass triage** each unit into incorporate / minor-incorporate / open-question / ignore. Keep a running internal table: `unit-id, source section, target file(s), bucket, one-line rationale`. Note when an incorporation will require multi-file edits. Reviewer-wrong is a *result* of verification (Step 5), not a first-pass call.
5. **Verify every Incorporate-bucket item, plus every cross-file Minor-incorporate item**, per [review-and-triage.md § "Verify before incorporating"](../specs/review-and-triage.md) and the impl-layer additions above. For cross-sprint claims (the most error-prone category), open both files. Items whose cited evidence doesn't hold move to **Reviewer-wrong** — do not edit the plan based on a claim that didn't survive verification.
6. **Calibration check.** Count the buckets against the impl-plan bands above. If Open question holds >20%, re-triage boundary cases. If Incorporate (full) holds >30%, re-check for fresh design or sprint-shape calls the reviewer did not recommend. If Reviewer-wrong holds >20%, **stop and surface this to the user** per the shared spec's stop-early rules (leave already-applied edits in place and record the partial pass in `status.md`).
7. **Group incorporations by file** so you can apply edits to each file in one pass instead of opening it repeatedly. Apply Incorporate and Minor incorporate items in the same pass; the only difference at edit time is whether a Decisions-log entry is added afterward. Apply incorporations in dependency order: structural moves (renaming a referenced helper, fixing a `progress.md` ↔ sprint mismatch) before within-section edits that depend on them. **For any multi-file incorporation, follow § "Multi-file incorporations (the high-risk path)" — list the cluster, edit canonical-wording first, consumers second, then grep the cluster for the rejected wording.** Make all edits via the `Edit` tool against the relevant file; do not rewrite a whole file unless the volume of edits genuinely warrants it.
8. **Apply Open-question additions** as a single batch at the end of each affected Open-questions list (overview-level batch first, then per-sprint batches). Preserve existing numbering — append-only.
9. **Update Decisions logs** with one bullet per *material* incorporation (Incorporate bucket only; Minor incorporates do not earn entries), all tagged with the same new round number. Cross-cutting bullets go in `decisions.md`; sprint-scoped bullets go in the affected sprint file's Decisions log section. Sprint-level entries can use the same round number even though the sprint file's prior round count may differ — what matters is that *this incorporation pass* is one coherent round.
10. **Update `progress.md`** if any step structure changed. Reconcile each affected sprint's per-sprint Progress section with the master.
11. **Append the `status.md` entry** as the last edit. If any sprint had its own Status / Feedback-incorporated section updated, those are already in place from step 7.
12. **Re-read the feature root's implementation-plan files** end-to-end one more time, looking for: stale wording your edits orphaned across files; Decisions-log bullets that contradict body edits in a sibling file; Open-questions entries your edits silently resolved; numbering inconsistencies; cross-sprint references that broke (a Prerequisite that names a Deliverable you renamed; a forward-link to a sprint whose section you re-titled). Fix what you find inside the same round.
13. **Report back to the user** in chat, per [review-and-triage.md § "Report format"](../specs/review-and-triage.md), with these impl-layer adaptations: group bullets by file or by sprint when the volume warrants it (cross-sprint coherence Minor incorporations grouped by the seam they crossed, not by file); name all touched files for multi-file incorporations; note which Decisions log each material entry landed in; and add a final **"Non-planning actions surfaced"** section (only if the review recommended changes outside the implementation-plan docs — the design plan, code, configs, conventions files, roster re-slicing, file renames — each bullet naming the action, what it affects, and that you did **not** perform it).

## Tone

- Direct and concrete; no hedging. The reviewer was opinionated; your triage should be too.
- Do not flatter the review or the plan. State what you did.
- Quote the plan and review with backticks/italics when disambiguating; otherwise paraphrase.
- Identifiers in backticks. Absolute dates. Canonical terms. Real file paths in the report (`Sprint 04 (`./04-manager-core.md`)`, not "the manager sprint").

## Anti-patterns

The shared list is canonical in [review-and-triage.md § "Triage anti-patterns (shared)"](../specs/review-and-triage.md) — honor all of it. Impl-specific additions:

- **Don't re-slice the sprint roster** as part of triage. Splitting / folding / re-numbering sprints is a planning-round move; surface it in the report or as an Open question.
- **Don't edit a shipped sprint's body** as if the code didn't exist. If a sprint's final step is `[x]` in `progress.md`, the implementation is the source of truth — capture the review's correction in "Feedback incorporated (post-review)" or "Deviations applied during implementation" instead of pretending the original plan said the new thing all along.
- **Don't pre-create `post-mortem.md`.** If the review asks for an entry but no sprint has shipped, decline the item.
- **Don't drift `progress.md` and per-sprint Progress.** If you edit one, edit the other in the same pass.
- **Don't edit the design plan** even when a review item routes back to a design-level call. Surface it in the report.
- **Don't skip verification on cross-file Minor-incorporate items.** Cross-sprint coherence findings are the highest-error category in impl-plan reviews — verify them with extra care.

---

## Additional Instructions

The **Trellis instruction precedence chain** (`~/.trellis/instructions.md` → `<repo-root>/.trellis/instructions.md` → instructions supplied with this invocation) is applied by `SKILL.md` before this file is dispatched, so it is already in effect; honor it. Canonical statement — the single source shared by every instruction file and subagent brief: [instruction-precedence.md](../specs/instruction-precedence.md).
