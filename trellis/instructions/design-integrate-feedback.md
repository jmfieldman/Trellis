---
name: design-integrate-feedback
description: Integrate Feedback into a Design Plan
argument-hint: <root> <feedback-path>
disable-model-invocation: true
---

# Skill — Incorporate review feedback into a design plan

You are continuing an in-progress design plan by reconciling it against a separate review document. The plan is the living artifact (anatomy and rules described in [Design Plan Documents — Authoring Guide](../specs/design-plan.md)); the review document is one round of external critique against it. Your job in this invocation is **triage and incorporation**, not full re-design — you are processing the review, not running a fresh planning round.

The shared triage machinery — the five buckets, the verify-before-incorporating rules, the adopt-the-recommendation posture, the decision-ready Open-question shape, the report format, and the shared anti-patterns — is canonical in [review-and-triage.md § Part II](../specs/review-and-triage.md). **Read Part II before triaging** — it is mandatory, not advisory. This file carries only what is specific to the *design* layer: the file set, the design-flavored bucket examples, the design calibration bands, and the design edit discipline.

This skill may be invoked directly, or entered via the sanctioned `design-review → design-integrate-feedback` chain (SKILL.md § "Sanctioned operation chains") — mechanics are identical; in the chained case `<feedback-path>` is the review the same invocation just produced by a dispatched reviewer subagent, so the fresh-context audit property holds.

## Inputs

Extract two paths from the user's natural-language invocation:

1. **`<root>`** — the directory containing the canonical design plan file `<root>/design.md`. If the user gave a path ending in `design.md`, treat its parent directory as `<root>`.
2. **`<feedback-path>`** — a review of that plan.

Each review item typically has a *Where*, *Concern*, *Why it matters*, and *Suggested direction*.

If either is missing, ask the user for it and stop. If `<root>/design.md` does not exist, stop and report the missing file.

## Scope of this skill

Per [review-and-triage.md § "What the integrator is (and is not) doing"](../specs/review-and-triage.md). At the design layer that means: you edit the plan's body, Decisions log, Open questions, and Status log — one file, `<root>/design.md`. You do not run a fresh design round, resolve pre-existing Open questions, or fabricate content the review doesn't support.

## Verify before incorporating

Follow [review-and-triage.md § "Verify before incorporating"](../specs/review-and-triage.md) in full — the failure modes, when verification is required (every material Incorporate; skipped for Minor incorporate / Open question / Ignore), the reasoning check on adopted recommendations, the routing of failed claims to Reviewer-wrong, the verification budget, and the stop-early handling (leave applied edits in place; record the partial pass in the Status entry). The design layer adds no extra required checks beyond the shared four — the plan is a single file, so there are no cross-file claims to double-read.

## The triage rule — design-layer specifics

The five buckets and their semantics are canonical in [review-and-triage.md § "The five buckets"](../specs/review-and-triage.md). Design-layer specifics:

### Items that typically land in Incorporate

Missing Schema-section sketch when the plan already proposes columns; missing API-surface table when the plan already commits to a new resource method; an Out-of-scope bullet that contradicts an Open question (you have to pick a side); a foundational-decision rationale paragraph the body discussed but didn't capture in the numbered list; a shape question (an API verb, a column type, a lifecycle choice) where the reviewer enumerated the options and recommended one whose reasoning holds up under verification.

### Items that typically land in Minor incorporate

- A typo, casing fix, or canonical-term swap where the framing block already settled the term.
- A marketing-language purge (`elegant`, `clean`, `robust`, `powerful`, `best-in-class`) the authoring guide explicitly flags.
- Removing a `(resolved)` tag inside Open questions and moving the entry to the Decisions log under its original round tag (recoverable from the Status log).
- Deleting stale wording from a superseded round still sitting in the body.
- Adding a missing forward-link in an Out-of-scope entry that already names the target elsewhere.
- Adding a missing entry in Cross-references for a service the body already names.
- Splitting a bundled Open question into two for clarity (preserves both, no decision made).
- Correcting a Decisions-log bullet that drops a clause the body already decided.
- Repairing numbering (skipped Q5; duplicate `(R3)` tag; out-of-order Status entries).
- An identifier-in-backticks pass over a section that drifted.

### Design-specific Open-question signals

Beyond the shared signals, the design layer treats **"promote this open question to a foundational decision"** as a planning move that always files as an Open question — promotion is the human's call.

### Calibration bands (design plan)

**These bands assume a review with ≳8 distinct items** (see the shared spec for the small-review caveat). A healthy design-plan triage is roughly:

- **20–40% Incorporate (full)** — material edits that earn a Decisions-log entry, including adopted reviewer recommendations on shape questions. This bucket runs heavier than under the old "frame, don't resolve" posture — that is intended.
- **15–30% Minor incorporate** — mechanical fixes (typos, marketing-language purges, missing forward-links, stale wording from superseded rounds, numbering repairs).
- **10–25% Open question** — only the genuinely-needs-a-human residue. If this bucket is large, you are probably being too conservative — re-check that you adopted every sound recommendation.
- **0–10% Reviewer-wrong** — a small but non-zero rate is normal. Above ~20% → stop, per the shared spec.
- **5–15% Ignore** — duplicate / out-of-scope / over-engineering / authoring-guide-contradicting / stylistic items.

Combined Incorporate + Minor incorporate is typically 35–70%.

Design-specific out-of-band signals (the shared spec carries the Open-question and Reviewer-wrong signals):

- If **Open question is holding more than ~35% of items**, re-triage the boundary cases per the shared spec's rule.
- If **Incorporate (full) is holding more than ~45%**, re-check whether you're making fresh design calls the reviewer did *not* recommend — adopting a reviewer's recommendation is in scope, inventing your own resolution is not. Also check you're not promoting Minor incorporates.
- If **Minor incorporate is holding more than ~50%**, the review may be heavy on style / wording nits (a sign the plan is structurally sound and the review is flailing for findings). The triage is fine, but flag the pattern in the report — the user may want a different review prompt next time.

## Edit discipline

When you do incorporate:

- **Edit the relevant body section directly.** Do not leave both old and new wording side-by-side; purge the obsolete phrasing per the authoring guide's supersession rule.
- **Add a tagged bullet to the Decisions log** for each material incorporation, using the next round number (`(R<n>)`). For pure typo / wording / structural-cleanup edits, a Decisions-log entry is optional — but for anything that names a call the plan now relies on, log it. Lead with a bold phrase that names the call.
- **Compress older Decisions-log entries** that this incorporation pass (or an earlier round) has superseded, plus any whose details have gone stale, per [design-plan.md § 14 → "Compressing older entries"](../specs/design-plan.md). Keep the last 2–3 rounds and any still-load-bearing entry in full.
- **Update Open questions**: remove an entry if the incorporation resolves it (rare in this skill — usually means you should have triaged it as Open question instead); add an entry for each item triaged into the Open-question bucket.
- **Append exactly one new Status entry** for this incorporation pass. Format: `**Round <n>**: incorporated review feedback — <one-line summary of what changed>; <count> material incorporations; <count> items added to Open questions; <count> items declined. _Next:_ <one-line recommended focus for the next planning round, citing Open question IDs + tags>.` Round number is the plan's next integer. The `_Next:_` clause persists the recommendation into the doc itself — a user resuming the plan via `design-iterate` reads it to recover where to pick up. When all newly-filed Open questions are `[deferred]` / `[exploratory]` and no `[blocks-v1]` / `[blocks-impl]` items remain anywhere in the plan, the `_Next:_` clause is `graduate to implementation plan`.
- **Compress older Status entries** that no longer carry weight after appending the new entry, per [design-plan.md § 15 → "Compressing older Status entries"](../specs/design-plan.md). Keep the last 2–3 rounds in full; never delete a round outright — round numbering stays contiguous and grep-able.
- **Re-read affected sections** after editing to catch wording from earlier rounds that your edits silently invalidated. If you find any, fix them in the same pass and note the supersession in the Status entry.
- **Preserve internal consistency.** If incorporating one review item creates a tension with another part of the plan you didn't touch, either resolve it by extending the edit or file the new tension as an Open question. Do not leave the plan internally contradictory.
- **Honor authoring conventions.** Bold the decision lead in Decisions-log entries. Tag every Decisions-log bullet with `(R<n>)`. Use canonical terms. No marketing words. Identifiers in backticks. Absolute dates only.

When you file an Open question, follow [review-and-triage.md § "Filing a decision-ready Open question"](../specs/review-and-triage.md) — options, why-a-human-is-needed, pros/cons per option, appended without renumbering, cross-linked to the review item. Plus the design layer's standard fields:

- **Tag at creation** with exactly one of `[blocks-v1]`, `[blocks-impl]`, `[deferred]`, `[exploratory]` per [design-plan.md § "Open questions" → "Severity tag taxonomy"](../specs/design-plan.md). Pick the most blocking tag that applies. If the reviewer's framing makes the tag obvious, use it; if not, default to `[blocks-impl]` for shape questions whose answer the impl plan needs and `[exploratory]` for context-only mentions.
- The named blockee for `[blocks-v1]` / `[blocks-impl]` entries; an explicit "deferred until X" for `[deferred]` entries.

When you ignore: do not edit the plan. Track the item only in your final chat report.

## Authorization scope for this skill

This skill edits a design plan in place. That is the explicit purpose, so you do not need additional confirmation for body edits, Decisions-log additions, Open-questions additions, or Status appends *within the named plan file*. You **do** need confirmation before:

- Editing any other file (including creating sibling docs, splitting the plan, or moving sections out).
- Renaming the plan file (a rename is a structural action — propose, don't act).
- Touching code, schemas, migrations, or anything outside the named plan path.

If a review item recommends changes outside the plan file, treat those as Open questions or surface them in the final chat report; do not act on them.

## Workflow

1. **Read both documents end-to-end.** No skimming. Note the plan's current Round count, last Status entry, current Open-questions numbering, and the Decisions-log round tags.
2. **Enumerate the review items.** Most reviews are already itemized (numbered concerns / nits). Treat each numbered item as one unit. If a review item bundles multiple sub-points, split it into sub-units before triaging — each sub-unit gets its own bucket.
3. **First-pass triage** each unit into incorporate / minor-incorporate / open-question / ignore. Keep a running internal table: `unit-id, bucket, one-line rationale`. The fifth bucket, **Reviewer-wrong**, is a *result* of verification (Step 4), not a first-pass call — that's why first-pass triage names only four.
4. **Verify every Incorporate-bucket item** before applying it, per [review-and-triage.md § "Verify before incorporating"](../specs/review-and-triage.md). Items whose cited evidence doesn't hold move to **Reviewer-wrong** — do not edit the plan based on a claim that didn't survive verification.
5. **Calibration check.** Count the buckets against the design-plan bands above. If Open question holds >35%, re-triage boundary cases. If Incorporate (full) holds >45%, re-check for fresh design calls the reviewer did not recommend. If Reviewer-wrong holds >20%, **stop and surface this to the user** per the shared spec's stop-early rules.
6. **Apply incorporations** in dependency order: structural moves (split a bundled question, add a missing section) before within-section edits. Apply Incorporate and Minor incorporate items in the same pass; the only difference at edit time is whether a Decisions-log entry is added afterward. Make all edits via the `Edit` tool against the plan file; do not rewrite the whole file unless the volume of edits genuinely warrants it.
7. **Apply Open-question additions** as a single batch at the end of the existing Open-questions list.
8. **Update Decisions log** with one bullet per *material* incorporation (Incorporate bucket only; Minor incorporates do not earn entries), all tagged with the same new round number.
9. **Append the Status entry** as the last edit.
10. **Re-read the plan** end-to-end one more time, looking for: stale wording your edits orphaned, decisions-log bullets that contradict body edits, open-questions that your edits silently resolved, numbering inconsistencies. Fix what you find inside the same round.
11. **Report back to the user** in chat, per [review-and-triage.md § "Report format"](../specs/review-and-triage.md).

## Tone

- Direct and concrete; no hedging. The reviewer was opinionated; your triage should be too.
- Do not flatter the review or the plan. State what you did.
- Quote the plan and review with backticks/italics when disambiguating; otherwise paraphrase.
- Identifiers in backticks. Absolute dates. Canonical terms.

## Anti-patterns

The shared list — don't silently resolve open questions, don't promote the round number twice, don't fabricate Decisions-log entries, don't abuse Minor incorporate, don't write build-spec specifics, don't reorder Open-questions lists, don't skip the final re-read, don't skip verification, don't bury Reviewer-wrong in Ignore — is canonical in [review-and-triage.md § "Triage anti-patterns (shared)"](../specs/review-and-triage.md). Honor all of it.

---

## Additional Instructions

The **Trellis instruction precedence chain** (`~/.trellis/instructions.md` → `<repo-root>/.trellis/instructions.md` → instructions supplied with this invocation) is applied by `SKILL.md` before this file is dispatched, so it is already in effect; honor it. Canonical statement — the single source shared by every instruction file and subagent brief: [instruction-precedence.md](../specs/instruction-precedence.md).
