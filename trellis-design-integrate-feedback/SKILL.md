---
name: trellis-design-integrate-feedback
description: Integrate Feedback into a Design Plan
argument-hint: <path-to-plan-file> <feedback-file>
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

- Triaging every distinct review item into one of five buckets: **incorporate**, **minor incorporate**, **open question**, **reviewer-wrong**, or **ignore**.
- **Verifying every material claim before acting on it** — the reviewer can be confident-but-wrong. See § "Verify before incorporating" below.
- Editing the plan's body, Decisions log, Open questions, and Status log to reflect the incorporations.
- Surfacing back to the user a concise audit of what you did, what you parked, and what the *reviewer* got wrong (which is signal the user may want to use to retrain or re-prompt the reviewer).

You are NOT:

- Resolving open questions the plan already had open. Your scope is the review document.
- Auto-resolving review items that hinge on a judgment call the human should make. Those go to Open questions, not into the body.
- Running a new planning round on the underlying domain. If a review item demands fresh design (not just a fix), it belongs in Open questions for the next planning round.
- Fabricating new content beyond what the review item supports. If a review concern is real but the right answer is not derivable from the plan + review, file it as an Open question rather than guessing.
- Trusting the review on faith. A wrong incorporation corrupts the plan; a missed-but-correct incorporation will be caught by the next review. Defaulting toward verification is asymmetric in the right direction.

## Verify before incorporating

Reviewer agents are confident even when wrong. The most common failure modes:

- The reviewer **misread the plan** — they cite §X as saying Y, but §X actually says Y' or doesn't address Y at all.
- The reviewer **hallucinated a cross-reference** — they cite `[[wiki/foo]]` as establishing rule Z, but the wiki page doesn't exist or doesn't establish Z.
- The reviewer **misread project conventions** — they assert the project does X, but `CLAUDE.md` (or the actual code) says ¬X.
- The reviewer **conflated two adjacent concerns** — they flag a contradiction that isn't one once you read the surrounding context.

A wrong incorporation is asymmetrically expensive: it corrupts the plan in a way the next reviewer may not catch (especially if the new wording is locally coherent). Defaulting toward verification is cheap insurance.

### When verification is required

**Required** before applying any **Incorporate** (full / material) edit — anything that earns a Decisions-log entry. Specifically:

1. **Confirm the cited section says what the reviewer claims.** Open the section. Read it. The reviewer's quote / paraphrase must match. If the reviewer cites a line range, the line range must contain the claimed wording.
2. **Confirm any cross-reference the reviewer relies on resolves.** If the reviewer says "this contradicts `[[wiki/foo]]`," confirm `[[wiki/foo]]` exists and actually establishes the rule. A `WebFetch` / `Read` is cheap.
3. **Confirm any project-convention claim against the source.** If the reviewer says "the project bans X," grep `CLAUDE.md` and adjacent project docs for the actual rule. Don't take the reviewer's paraphrase at face value.
4. **Confirm contradictions actually contradict.** When the reviewer says "§3 says X but §7 says ¬X," read both sections in full — sometimes the conflict dissolves once context is loaded.

**Not required** for **Minor incorporate** — typo fixes, marketing-language purges, missing-cross-link adds, numbering repairs, stale-text deletes following an explicit Status-log supersession. These don't hinge on the reviewer's interpretation; the body wins.

**Not required** for **Open question** filings — the question is being parked, not answered. The reviewer's framing becomes the open-question text; if it's wrong, the next round will re-frame it.

**Not required** for **Ignore** — the item isn't being acted on.

### When verification fails

If verification turns up that the review's claim is wrong (the cited section doesn't say what the reviewer claims; the cross-reference doesn't establish what the reviewer claims; the project convention isn't what the reviewer claims; the contradiction isn't actually a contradiction), route the item to the **Reviewer-wrong** bucket (not Ignore). See § "The triage rule" below. This is a distinct signal the user may want to use to recalibrate the reviewer.

### Verification budget

This step is fast — the median Incorporate item is verified in 30 seconds with a Read or grep. Don't skip it because "the review looks careful" — agent reviewers are uniformly confident.

If the volume of verification is large and you find a high rate of reviewer-wrong findings (>20% of Incorporate-bucket items fail verification), stop and surface this to the user before continuing. The review may need to be re-run rather than triaged.

---

## The triage rule

For each review item, decide which bucket. The five buckets are ordered by descending invasiveness — **Incorporate** is a load-bearing edit with a Decisions-log entry; **Minor incorporate** is a mechanical edit without one; **Open question** files for the next round; **Reviewer-wrong** records that the review itself was off; **Ignore** does nothing.

### Incorporate (edit the body directly + add Decisions-log entry)

A *material* incorporation that names a call the plan now relies on. Use this bucket when **all** of the following hold:

- The review item identifies a concrete, verifiable defect that, when fixed, *changes a load-bearing call* — adding a missing section the authoring guide requires (when the body already implies its content), reconciling a Decisions-log entry with the body it summarizes, locking a previously-ambiguous structural choice the body had two ways of expressing.
- The fix can be made **without inventing new design** — i.e. you are tightening, splitting, renaming, relocating, or deleting existing content, not making a fresh judgment call.
- You have **high confidence** the fix is right. If you would hesitate to defend the edit to the human in one sentence, it is not high confidence.
- The fix does not implicitly resolve a question the plan currently lists as open. Resolving Open questions is a separate planning round.
- The edit names a call the plan now relies on — so it earns a Decisions-log bullet.

Examples of items that typically land here: missing Schema-section sketch when the plan already proposes columns; missing API-surface table when the plan already commits to a new resource method; an Out-of-scope bullet that contradicts an Open question (you have to pick a side); a foundational-decision rationale paragraph the body discussed but didn't capture in the numbered list.

### Minor incorporate (edit the body directly, no Decisions-log entry)

Mechanical fixes that *don't* name a call the plan now relies on. The body wins, the edit is obvious, no log entry needed. Use this bucket when:

- The fix is a typo, casing fix, or canonical-term swap where the framing block already settled the term.
- The fix is a marketing-language purge (`elegant`, `clean`, `robust`, `powerful`, `best-in-class`) the authoring guide explicitly flags.
- The fix removes a `(resolved)` tag inside Open questions and moves the entry to the Decisions log under its original round tag (recoverable from the Status log).
- The fix deletes stale wording from a superseded round still sitting in the body.
- The fix adds a missing forward-link in an Out-of-scope entry that already names the target elsewhere.
- The fix adds a missing entry in Cross-references for a service the body already names.
- The fix splits a bundled Open question into two for clarity (preserves both, no decision made).
- The fix corrects a Decisions-log bullet that drops a clause the body already decided.
- The fix repairs numbering (skipped Q5; duplicate `(R3)` tag; out-of-order Status entries).
- The fix adds an identifier-in-backticks pass over a section that drifted.

Apply the edit. Don't add a Decisions-log entry. The Status entry for this incorporation pass already names the round; the *absence* of a Decisions-log bullet is the correct signal that no new call was made.

If you find yourself wanting to add a Decisions-log entry to a Minor incorporate, the item is probably a full Incorporate — promote it.

### Open question (file it for the next round)

When the review item is real but resolving it requires a human judgment call, new domain knowledge, or design work beyond mechanical reconciliation. Typical signals:

- The reviewer offers multiple alternatives without one being obviously right.
- The fix requires information not in the plan (a sample external payload, an ops confirmation, a stakeholder decision).
- The fix would change the shape of a foundational decision, the resource contract, the schema, or the lifecycle invariants.
- The fix is a "promote this open question to a foundational decision" recommendation — promotion is itself a planning move the human should make.
- The fix is a category of risk (rollout, observability, partial-failure recovery) the plan has not yet sectioned for. Adding the section is design work; *naming* the gap is an open question.

When filing an Open question from a review item: capture the question, why it's open, the indicative direction the reviewer suggested (if any), and whether it blocks anything currently in scope. Use the standard Open-question shape from the authoring guide. Do **not** copy the review item verbatim — distill it.

### Reviewer-wrong (the review's claim doesn't survive verification)

Use this bucket when **verification revealed the review was factually wrong** — not when you disagreed with the reviewer's judgment, but when their citation, cross-reference, or convention claim doesn't hold. Specifically:

- The reviewer cited §X as saying Y, but §X says Y' or doesn't address Y.
- The reviewer cited a wiki page or cross-reference that doesn't exist or doesn't establish the cited rule.
- The reviewer asserted a project convention (`CLAUDE.md` or similar) that the source actually contradicts.
- The reviewer flagged a contradiction between sections that, once both are read in full, isn't actually a contradiction.

Do not edit the plan. Record the item with: the original review concern, what the reviewer claimed, what the source actually says (in one sentence — quote the actual text), and an indication of which failure mode this was.

Reviewer-wrong items are surfaced in the report in their own section so the user can see the rate of reviewer error and decide whether the review should be re-run rather than triaged. **Do not silently move reviewer-wrong items into Ignore** — that hides the signal.

If the review item is *partially* wrong — the cited evidence doesn't hold, but a real concern exists once the actual text is read — split the item: route the wrong-as-cited piece to Reviewer-wrong, and re-frame the real concern as an Open question in the report (don't fabricate a fix the reviewer didn't actually suggest).

### Ignore (and explain why)

When the review item does not warrant action and is not factually wrong. Possible reasons:

- The concern duplicates another review item you are already handling.
- The concern is out-of-scope for this plan (e.g. a feature the plan explicitly defers).
- The "fix" would over-engineer the plan against a hypothetical the plan deliberately excluded.
- The recommendation contradicts the authoring guide (e.g. asking for build-spec specifics, leaving `(resolved)` tags, or adding speculative future-proofing).
- The item is a stylistic preference the plan's existing voice already handles consistently.

> Note: "the reviewer misread the plan" is **not** an Ignore reason — it's a Reviewer-wrong finding. Use the right bucket so the user gets the signal.

Ignoring is legitimate. Do not pad the incorporate bucket out of politeness to the reviewer. But every ignore must come with a one-sentence reason you can defend.

### When in doubt — but watch for over-conservatism

Default to **Open question** when the item is a *design-shape* call — a foundational-decision shift, a schema column change, an API verb choice, a lifecycle invariant, a cross-service contract change. The cost of an unwarranted body edit on a design-shape call (silently hard-coding a choice the human would have made differently) is higher than the cost of one extra Open-question bullet.

Default to **Minor incorporate** when the item is a *wording / consistency / stale-text / missing-cross-link / numbering* fix. These are not design decisions; routing them to Open questions just adds noise and pads the next round's triage list.

**Calibration check.** After triaging the whole review, count the buckets. The shape of a healthy *design-plan* triage is different from a healthy impl-plan triage — design-plan reviews surface a higher proportion of shape questions whose resolution is a judgment call (the Open-question bucket runs heavier), while impl-plan reviews surface more cross-sprint coherence drift (the Minor-incorporate bucket runs heavier there). Use the design-plan bands below; the impl-plan version of this skill uses different bands.

A healthy design-plan triage is roughly:

- **5–15% Incorporate (full)** — material edits that earn a Decisions-log entry. Material incorporations are the minority of any review; a higher rate usually means fresh design calls are hiding under "incorporation."
- **15–30% Minor incorporate** — mechanical fixes (typos, marketing-language purges, missing forward-links, stale wording from superseded rounds, numbering repairs).
- **30–50% Open question** — design-plan reviews naturally surface a high concentration of shape questions whose resolution is a judgment call. This bucket runs heavier for design plans than for impl plans.
- **0–10% Reviewer-wrong** — a small but non-zero rate is normal. A rate above ~20% is a signal that the review itself should be re-run rather than triaged; surface this to the user.
- **5–15% Ignore** — duplicate / out-of-scope / over-engineering / authoring-guide-contradicting / stylistic items.

Combined Incorporate + Minor incorporate is typically 20–45%.

Out-of-band signals:

- If **Open question is holding more than ~60% of items**, re-triage the boundary cases — you are being too conservative. Default-to-Open-question is a safety net for genuine judgment calls, not a default for everything you'd rather not commit to. The whole point of running review → integrate-feedback is to converge the plan, not to pad the Open Questions list with debt that the next round inherits.
- If **Incorporate (full) is holding more than ~25%**, re-check whether you're making fresh design calls under the cover of "incorporation." Material incorporations should still be the minority — most reviewer-surfaced findings are either mechanical (Minor incorporate) or judgment calls (Open question).
- If **Minor incorporate is holding more than ~50%**, the review may be heavy on style / wording nits (a sign the plan is structurally sound and the review is flailing for findings). The triage is fine, but flag the pattern in the report — the user may want a different review prompt next time.
- If **Reviewer-wrong is holding more than ~20%**, the review's reliability is the bottleneck — stop, report the rate to the user, and let them decide whether to re-run the review with a different reviewer / prompt rather than continue triaging a noisy artifact.

## Edit discipline

When you do incorporate:

- **Edit the relevant body section directly.** Do not leave both old and new wording side-by-side; purge the obsolete phrasing per the authoring guide's supersession rule.
- **Add a tagged bullet to the Decisions log** for each material incorporation, using the next round number (`(R<n>)`). For pure typo / wording / structural-cleanup edits, a Decisions-log entry is optional — but for anything that names a call the plan now relies on, log it. Lead with a bold phrase that names the call.
- **Update Open questions**: remove an entry if the incorporation resolves it (rare in this skill — usually means you should have triaged it as Open question instead); add an entry for each item triaged into the Open-question bucket.
- **Append exactly one new Status entry** for this incorporation pass. Format: `**Round <n>**: incorporated review feedback — <one-line summary of what changed>; <count> items added to Open questions; <count> items declined. _Next:_ <one-line recommended focus for the next planning round, citing Open question IDs + tags>.` Round number is the plan's next integer. The `_Next:_` clause persists the recommendation into the doc itself — a user resuming the plan via `trellis-design-iterate` reads it to recover where to pick up. When all newly-filed Open questions are `[deferred]` / `[exploratory]` and no `[blocks-v1]` / `[blocks-impl]` items remain anywhere in the plan, the `_Next:_` clause is `graduate to implementation plan`.
- **Re-read affected sections** after editing to catch wording from earlier rounds that your edits silently invalidated. If you find any, fix them in the same pass and note the supersession in the Status entry.
- **Preserve internal consistency.** If incorporating one review item creates a tension with another part of the plan you didn't touch, either resolve it by extending the edit or file the new tension as an Open question. Do not leave the plan internally contradictory.
- **Honor authoring conventions.** Bold the decision lead in Decisions-log entries. Tag every Decisions-log bullet with `(R<n>)`. Use canonical terms. No marketing words. Identifiers in backticks. Absolute dates only.

When you file an Open question:

- Place it at the end of the existing Open-questions list, numbered sequentially. Do not renumber existing entries.
- **Tag at creation** with exactly one of `[blocks-v1]`, `[blocks-impl]`, `[deferred]`, `[exploratory]` per [design-plan.md § "Open questions" → "Severity tag taxonomy"](../trellis-design-create/design-plan.md). Pick the most blocking tag that applies. If the reviewer's framing makes the tag obvious, use it; if not, default to `[blocks-impl]` for shape questions whose answer the impl plan needs and `[exploratory]` for context-only mentions.
- Each entry: the tag (in square brackets after the bold title); the question (one sentence); why it's open / what makes it hard; indicative direction (or explicit "deferred until X" for `[deferred]` entries); the named blockee for `[blocks-v1]` / `[blocks-impl]` entries.
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
3. **First-pass triage** each unit into incorporate / minor-incorporate / open-question / ignore. Keep a running internal table: `unit-id, bucket, one-line rationale`. Reviewer-wrong is a *result* of verification (Step 4), not a first-pass call.
4. **Verify every Incorporate-bucket item** before applying it. For each, confirm the cited section / cross-reference / convention claim against the actual source per § "Verify before incorporating." Items whose cited evidence doesn't hold move to **Reviewer-wrong** — do not edit the plan based on a claim that didn't survive verification. Minor incorporate, Open question, and Ignore items skip verification (see § "When verification is required").
5. **Calibration check.** Count the buckets against the design-plan bands in § "The triage rule" → "Calibration check" (target ~5–15% Incorporate, ~15–30% Minor, ~30–50% Open question, 0–10% Reviewer-wrong, ~5–15% Ignore). If Open question holds >60%, re-triage boundary cases. If Incorporate (full) holds >25%, re-check for fresh design calls hiding under "incorporation." If Reviewer-wrong holds >20%, **stop and surface this to the user** — the review may need to be re-run rather than triaged.
6. **Apply incorporations** in dependency order: structural moves (split a bundled question, add a missing section) before within-section edits. Apply Incorporate and Minor incorporate items in the same pass; the only difference at edit time is whether a Decisions-log entry is added afterward. Make all edits via the `Edit` tool against the plan file; do not rewrite the whole file unless the volume of edits genuinely warrants it.
7. **Apply Open-question additions** as a single batch at the end of the existing Open-questions list.
8. **Update Decisions log** with one bullet per *material* incorporation (Incorporate bucket only; Minor incorporates do not earn entries), all tagged with the same new round number.
9. **Append the Status entry** as the last edit.
10. **Re-read the plan** end-to-end one more time, looking for: stale wording your edits orphaned, decisions-log bullets that contradict body edits, open-questions that your edits silently resolved, numbering inconsistencies. Fix what you find inside the same round.
11. **Report back to the user** in chat (format below).

## Report format

Output the bucket-distribution summary first, then five sections, in this order, in chat. Keep each one terse — bullets, not paragraphs.

**Bucket distribution**: `<I> material / <m> minor / <O> open-question / <W> reviewer-wrong / <X> ignore` (totals to the review's item count).

### Incorporated

Two sub-lists:

- **Material incorporations** (with Decisions-log entries) — each bullet: the review concern (short reference, not a quote dump) + the section of the plan that changed + the Decisions-log entry's bold lead. One line each.
- **Minor incorporations** (no Decisions-log entry) — each bullet: the review concern + the section that changed. Group these tightly; if there are more than ~6, summarize by category ("8 marketing-language purges across §§ Foundational decisions / Schema; 3 missing forward-links in Out-of-scope") rather than listing each.

### Added to Open questions

A bulleted summary of items filed for the next round. Each bullet: the question (one short phrase) + a one-sentence reason it's a judgment call rather than a mechanical fix.

### Reviewer-wrong (verification failures)

A bulleted summary of review items whose cited evidence didn't hold. Each bullet: the review item (short reference) + what the reviewer claimed + what the source actually says (one sentence; quote when the wording is the load-bearing part) + the failure mode (`misread-section` / `bad-cross-reference` / `wrong-convention-claim` / `non-contradiction`).

If this section has more than ~3 items or represents >20% of the review, lead the section with a one-line summary line so the user notices: **"Reviewer reliability concern: <N> of <total> items did not survive verification — consider re-running the review."**

If the section is empty, omit it.

### Declined

A bulleted summary of review items you ignored (excluding Reviewer-wrong, which has its own section). Each bullet: the review item (short reference) + a one-sentence reason. Be honest — if you ignored multiple items for the same reason, group them.

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
- **Don't abuse Minor incorporate to land load-bearing edits without a Decisions-log entry.** Minor incorporate exists for typos / wording / consistency / stale-text / numbering / missing-cross-link fixes. If your edit names a call the plan now relies on (a foundational shape, a schema column, an API verb, a contract clause), it's a full Incorporate — add the Decisions-log entry. The audit trail breaks if material calls hide in the Minor bucket.
- **Don't write build-spec specifics** even if a review item asks for them (file paths, full TypeScript signatures, exact function names beyond illustrative). Sketches only.
- **Don't reorder the Open-questions list** to "improve flow." Append-only — preserve the existing numbering so prior conversation references stay valid.
- **Don't skip the final re-read.** It is the cheapest way to catch the inconsistencies your own edits introduced.
- **Don't skip verification on Incorporate-bucket items.** A confident-but-wrong reviewer becomes a corrupted plan if their claims are acted on without checking the source. Verification is fast (median 30 seconds per item) and asymmetrically valuable.
- **Don't bury Reviewer-wrong findings in the Ignore bucket.** That hides the signal the user needs to recalibrate the reviewer. Reviewer-wrong has its own bucket and its own section in the report for a reason.

---

## Additional Instructions

Before executing this skill, read and apply Trellis instructions from these sources, in ascending order of precedence:

1. `~/.trellis/instructions.md`
2. `<repo-root>/.trellis/instructions.md`
3. Additional instructions included with this skill invocation

`<repo-root>` is the root of the current Git repository. Missing instruction files are normal; skip them silently.

If instructions conflict, later sources override earlier sources. Invocation-specific instructions apply only to the current run and have the highest precedence.

These instructions may override this skill document, but they must not override system, developer, tool, safety, or repository policy instructions.