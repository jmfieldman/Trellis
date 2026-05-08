---
name: trellis-design-integrate-feedback
description: Integrate Feedback into a Design Plan [$0=path-to-plan-file] [$1=feedback-file]
disable-model-invocation: true
---

# Skill — Incorporate review feedback into a design plan

You are continuing an in-progress design plan by reconciling it against a separate review document. The plan is the living artifact (anatomy and rules described in [Design Plan Documents — Authoring Guide](../trellis-design-create/design-plan.md); the review document is one round of external critique against it. Your job in this invocation is **triage and incorporation**, not full re-design — you are processing the review, not running a fresh planning round.

## Inputs

You will be given two file paths:

1. **`<plan-path>`** — the design plan in progress: $0
2. **`<feedback-path>`** — a review of that plan: $1
 
Each review item typically has a *Where*, *Concern*, *Why it matters*, and *Suggested direction*.

If one of the paths is not provided, tell the user they forgot an argument to this skill invocation, and stop.

## What you are doing (and not doing)

You ARE:

- Triaging every distinct review item into one of three buckets: **incorporate**, **open question**, or **ignore**.
- Editing the plan's body, Decisions log, Open questions, and Status log to reflect the incorporations.
- Surfacing back to the user a concise audit of what you did and what you ignored.

You are NOT:

- Resolving open questions the plan already had open. Your scope is the review document.
- Auto-resolving review items that hinge on a judgment call the human should make. Those go to Open questions, not into the body.
- Running a new planning round on the underlying domain. If a review item demands fresh design (not just a fix), it belongs in Open questions for the next planning round.
- Fabricating new content beyond what the review item supports. If a review concern is real but the right answer is not derivable from the plan + review, file it as an Open question rather than guessing.

## The triage rule

For each review item, decide which bucket:

### Incorporate (edit the body directly)

Only when **all** of the following hold:

- The review item identifies a concrete, verifiable defect — a contradiction inside the plan, a wording inconsistency, a missing section that the authoring guide explicitly requires, an inconsistency with a Decisions-log entry, a stale phrase left over from a superseded round, a missing cross-reference to a file that already exists, an authoring-guide deviation (e.g. marketing language, `(resolved)` tags inside Open questions), or a clearly-correct factual nit (typo, wrong term, casual term where canonical term was already established).
- The fix can be made **without inventing new design** — i.e. you are tightening, splitting, renaming, relocating, or deleting existing content, not making a load-bearing call.
- You have **high confidence** the fix is right. If you would hesitate to defend the edit to the human in one sentence, it is not high confidence.
- The fix does not implicitly resolve a question the plan currently lists as open. Resolving Open questions is a separate planning round.

Examples of items that typically incorporate cleanly: missing Schema-section sketch when the plan already proposes columns; missing API-surface table when the plan already commits to a new resource method; a Decisions-log entry that drops a clause the body decided; "single source of truth" or other marketing wording that the authoring guide flags; bundled open questions that should be split; a title that uses the casual term when the framing block already canonicalized a different one; a missing entry in Cross-references for a service the body names; an Out-of-scope bullet that contradicts an Open question.

### Open question (file it for the next round)

When the review item is real but resolving it requires a human judgment call, new domain knowledge, or design work beyond mechanical reconciliation. Typical signals:

- The reviewer offers multiple alternatives without one being obviously right.
- The fix requires information not in the plan (a sample external payload, an ops confirmation, a stakeholder decision).
- The fix would change the shape of a foundational decision, the resource contract, the schema, or the lifecycle invariants.
- The fix is a "promote this open question to a foundational decision" recommendation — promotion is itself a planning move the human should make.
- The fix is a category of risk (rollout, observability, partial-failure recovery) the plan has not yet sectioned for. Adding the section is design work; *naming* the gap is an open question.

When filing an Open question from a review item: capture the question, why it's open, the indicative direction the reviewer suggested (if any), and whether it blocks anything currently in scope. Use the standard Open-question shape from the authoring guide. Do **not** copy the review item verbatim — distill it.

### Ignore (and explain why)

When the review item does not warrant action. Possible reasons:

- The reviewer misread the plan (the concern is already addressed in a section they didn't cite).
- The concern duplicates another review item you are already handling.
- The concern is out-of-scope for this plan (e.g. a feature the plan explicitly defers).
- The "fix" would over-engineer the plan against a hypothetical the plan deliberately excluded.
- The recommendation contradicts the authoring guide (e.g. asking for build-spec specifics, leaving `(resolved)` tags, or adding speculative future-proofing).
- The item is a stylistic preference the plan's existing voice already handles consistently.

Ignoring is legitimate. Do not pad the incorporate bucket out of politeness to the reviewer. But every ignore must come with a one-sentence reason you can defend.

### When in doubt

Default to **Open question**, not Incorporate. The cost of an unwarranted body edit (silently hard-coding a design choice the human would have made differently) is higher than the cost of one extra Open-question bullet for the next round.

## Edit discipline

When you do incorporate:

- **Edit the relevant body section directly.** Do not leave both old and new wording side-by-side; purge the obsolete phrasing per the authoring guide's supersession rule.
- **Add a tagged bullet to the Decisions log** for each material incorporation, using the next round number (`(R<n>)`). For pure typo / wording / structural-cleanup edits, a Decisions-log entry is optional — but for anything that names a call the plan now relies on, log it. Lead with a bold phrase that names the call.
- **Update Open questions**: remove an entry if the incorporation resolves it (rare in this skill — usually means you should have triaged it as Open question instead); add an entry for each item triaged into the Open-question bucket.
- **Append exactly one new Status entry** for this incorporation pass. Format: `**Round <n>**: incorporated review feedback — <one-line summary of what changed>; <count> items added to Open questions; <count> items declined.` Round number is the plan's next integer.
- **Re-read affected sections** after editing to catch wording from earlier rounds that your edits silently invalidated. If you find any, fix them in the same pass and note the supersession in the Status entry.
- **Preserve internal consistency.** If incorporating one review item creates a tension with another part of the plan you didn't touch, either resolve it by extending the edit or file the new tension as an Open question. Do not leave the plan internally contradictory.
- **Honor authoring conventions.** Bold the decision lead in Decisions-log entries. Tag every Decisions-log bullet with `(R<n>)`. Use canonical terms. No marketing words. Identifiers in backticks. Absolute dates only.

When you file an Open question:

- Place it at the end of the existing Open-questions list, numbered sequentially. Do not renumber existing entries.
- Each entry: the question (one sentence), why it's open / what makes it hard, indicative direction (or explicit "deferred until X"), and whether it blocks anything currently in scope.
- Cross-link to the originating review item by quoting a short phrase from the review (in italics) so the human can trace it back.

When you ignore:

- Do not edit the plan. Track the item only in your final chat report.

## Authorization scope for this skill

This skill edits a design plan in place. That is the explicit purpose, so you do not need additional confirmation for body edits, Decisions-log additions, Open-questions additions, or Status appends *within the named plan file*. You **do** need confirmation before:

- Editing any other file (including creating sibling docs, splitting the plan, or moving sections out).
- Renaming the plan file (concern #27 in the example feedback is a rename — propose, don't act).
- Touching code, schemas, migrations, or anything outside the named plan path.

If a review item recommends changes outside the plan file, treat those as Open questions or surface them in the final chat report; do not act on them.

## Workflow

1. **Read both documents end-to-end.** No skimming. Note the plan's current Round count, last Status entry, current Open-questions numbering, and the Decisions-log round tags.
2. **Enumerate the review items.** Most reviews are already itemized (numbered concerns / nits). Treat each numbered item as one unit. If a review item bundles multiple sub-points, split it into sub-units before triaging — each sub-unit gets its own bucket.
3. **Triage each unit** into incorporate / open-question / ignore using the rules above. Keep a running internal table: `unit-id, bucket, one-line rationale`.
4. **Apply incorporations** in dependency order: structural moves (split a bundled question, add a missing section) before within-section edits. Make all edits via the `Edit` tool against the plan file; do not rewrite the whole file unless the volume of edits genuinely warrants it.
5. **Apply Open-question additions** as a single batch at the end of the existing Open-questions list.
6. **Update Decisions log** with one bullet per material incorporation, all tagged with the same new round number.
7. **Append the Status entry** as the last edit.
8. **Re-read the plan** end-to-end one more time, looking for: stale wording your edits orphaned, decisions-log bullets that contradict body edits, open-questions that your edits silently resolved, numbering inconsistencies. Fix what you find inside the same round.
9. **Report back to the user** in chat (format below).

## Report format

Output exactly three sections, in this order, in chat. Keep each one terse — bullets, not paragraphs.

### Incorporated
A bulleted summary of what was incorporated into the body. Each bullet: the review concern (short reference, not a quote dump) + the section of the plan that changed. One line each. If a Decisions-log entry was added, name its bold lead.

### Added to Open questions
A bulleted summary of items filed for the next round. Each bullet: the question (one short phrase) + a one-sentence reason it's a judgment call rather than a mechanical fix.

### Declined
A bulleted summary of review items you ignored. Each bullet: the review item (short reference) + a one-sentence reason. Be honest — if you ignored multiple items for the same reason, group them.

End with one sentence naming the new Round number you wrote and (if any) cross-file actions you flagged but did not perform.

## Tone

- Direct and concrete; no hedging. The reviewer was opinionated; your triage should be too.
- Do not flatter the review or the plan. State what you did.
- Quote the plan and review with backticks/italics when disambiguating; otherwise paraphrase.
- Identifiers in backticks. Absolute dates. Canonical terms.

## Anti-patterns specific to this skill

- **Don't silently resolve open questions.** If a review item proposes an answer to an existing Open question, that is a *planning round* move, not an *incorporation* move. File it as a refinement to the existing Open question or surface it in the report — do not delete the Open-questions entry yourself.
- **Don't promote the round number twice.** Every incorporation in this pass shares one round tag. The plan gets exactly one new Status entry.
- **Don't fabricate Decisions-log entries** for review items you triaged as Open questions. The Decisions log is for resolved calls, not for "we noted this and will think about it."
- **Don't write build-spec specifics** even if a review item asks for them (file paths, full TypeScript signatures, exact function names beyond illustrative). Sketches only.
- **Don't reorder the Open-questions list** to "improve flow." Append-only — preserve the existing numbering so prior conversation references stay valid.
- **Don't skip the final re-read.** It is the cheapest way to catch the inconsistencies your own edits introduced.

---

## Additional Instructions

This skill may have been invoked with additional instructions. Those instructions should take precedence if they conflict with any instruction in this document.