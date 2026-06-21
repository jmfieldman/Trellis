---
name: design-integrate-feedback
description: Integrate Feedback into a Design Plan
argument-hint: <root> <feedback-path>
disable-model-invocation: true
---

# Skill — Incorporate review feedback into a design plan

You are continuing an in-progress design plan by reconciling it against a separate review document. The plan is the living artifact (anatomy and rules described in [Design Plan Documents — Authoring Guide](../specs/design-plan.md); the review document is one round of external critique against it. Your job in this invocation is **triage and incorporation**, not full re-design — you are processing the review, not running a fresh planning round.

## Inputs

Extract two paths from the user's natural-language invocation:

1. **`<root>`** — the directory containing the canonical design plan file `<root>/design.md`. If the user gave a path ending in `design.md`, treat its parent directory as `<root>`.
2. **`<feedback-path>`** — a review of that plan.

Each review item typically has a *Where*, *Concern*, *Why it matters*, and *Suggested direction*.

If either is missing, ask the user for it and stop. If `<root>/design.md` does not exist, stop and report the missing file.

## What you are doing (and not doing)

You ARE:

- Triaging every distinct review item into one of five buckets: **incorporate**, **minor incorporate**, **open question**, **reviewer-wrong**, or **ignore**.
- **Verifying every material claim before acting on it** — the reviewer can be confident-but-wrong. See § "Verify before incorporating" below.
- Editing the plan's body, Decisions log, Open questions, and Status log to reflect the incorporations.
- Surfacing back to the user a concise audit of what you did, what you parked, and what the *reviewer* got wrong (which is signal the user may want to use to retrain or re-prompt the reviewer).

You are NOT:

- Resolving open questions the plan already had open. Your scope is the review document.
- Auto-resolving review items that hinge on a judgment call *only the human* can make. When the reviewer laid out the options and recommended one with sound reasoning, adopting that recommendation **is** your job — that is the decision the review → integrate-feedback loop exists to make. What you don't do is invent a resolution the reviewer didn't propose, or override a sound recommendation on a whim. Items go to Open questions only when the choice is genuinely more nuanced than the reviewer's framing, or genuinely needs a human operator — see § "The triage rule".
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

When the Incorporate item is an **adopted reviewer recommendation** on a judgment call, verification has a second half: beyond confirming the cited evidence (steps 1–4 above), confirm the reviewer's *reasoning* for the recommendation holds — the trade-off they cite is real, the constraint they lean on actually applies, and you don't see a material consideration they missed. If the evidence checks out but the reasoning doesn't, the item is an **Open question** (the choice is more nuanced than the reviewer's framing), not an Incorporate — and not Reviewer-wrong, since Reviewer-wrong is for failed *factual* claims, not for a recommendation you'd weigh differently.

**Not required** for **Minor incorporate** — typo fixes, marketing-language purges, missing-cross-link adds, numbering repairs, stale-text deletes following an explicit Status-log supersession. These don't hinge on the reviewer's interpretation; the body wins.

**Not required** for **Open question** filings — the question is being parked, not answered. The reviewer's framing becomes the open-question text; if it's wrong, the next round will re-frame it.

**Not required** for **Ignore** — the item isn't being acted on.

### When verification fails

If verification turns up that the review's claim is wrong (the cited section doesn't say what the reviewer claims; the cross-reference doesn't establish what the reviewer claims; the project convention isn't what the reviewer claims; the contradiction isn't actually a contradiction), route the item to the **Reviewer-wrong** bucket (not Ignore). See § "The triage rule" below. This is a distinct signal the user may want to use to recalibrate the reviewer.

### Verification budget

This step is fast — the median Incorporate item is verified in 30 seconds with a Read or grep. Don't skip it because "the review looks careful" — agent reviewers are uniformly confident.

If the volume of verification is large and reviewer-wrong findings are piling up — more than ~20% of the Incorporate-bucket items you've verified so far are failing — stop and surface this to the user before continuing. The review may need to be re-run rather than triaged. This is the **same ~20% threshold** as the post-triage Reviewer-wrong calibration band (§ "The triage rule" → "Calibration check"), measured against a different denominator at a different moment: this check runs *during* verification against the Incorporate-bucket items verified so far (the earliest signal), while the calibration band runs *after* triage against all review items (the final tally). Either crossing ~20% is cause to stop.

---

## The triage rule

For each review item, decide which bucket. The five buckets are ordered by descending invasiveness — **Incorporate** is a load-bearing edit with a Decisions-log entry; **Minor incorporate** is a mechanical edit without one; **Open question** files for the next round; **Reviewer-wrong** records that the review itself was off; **Ignore** does nothing.

### Incorporate (edit the body directly + add Decisions-log entry)

A *material* incorporation that names a call the plan now relies on. Use this bucket when **all** of the following hold:

- The review item identifies a concrete, verifiable defect that, when fixed, *changes a load-bearing call* — adding a missing section the authoring guide requires (when the body already implies its content), reconciling a Decisions-log entry with the body it summarizes, locking a previously-ambiguous structural choice the body had two ways of expressing.
- The fix can be made **without inventing new design** — i.e. you are tightening, splitting, renaming, relocating, or deleting existing content, *or adopting a resolution the reviewer explicitly recommended*. Adopting the reviewer's recommended option is not "inventing new design" — it is the call the review → integrate-feedback loop exists to make. What stays out of this bucket is a fresh judgment call *neither the plan nor the reviewer* supplies an answer to.
- You have **high confidence** the fix is right. If you would hesitate to defend the edit to the human in one sentence, it is not high confidence. When you are adopting a reviewer's recommended option, "high confidence" means: you verified the reviewer's cited evidence (per § "Verify before incorporating"), the reviewer's reasoning for the recommendation holds up, and you don't see a material consideration the reviewer missed. If the reasoning holds, adopt it — don't downgrade a sound recommendation to an Open question out of caution.
- The fix does not implicitly resolve a question the plan *currently lists* as open. Resolving pre-existing Open questions is a separate planning round. (This is distinct from acting on a reviewer's recommendation about a *new* concern the review surfaced — that is in scope.)
- The edit names a call the plan now relies on — so it earns a Decisions-log bullet.

Examples of items that typically land here: missing Schema-section sketch when the plan already proposes columns; missing API-surface table when the plan already commits to a new resource method; an Out-of-scope bullet that contradicts an Open question (you have to pick a side); a foundational-decision rationale paragraph the body discussed but didn't capture in the numbered list; a shape question (an API verb, a column type, a lifecycle choice) where the reviewer enumerated the options and recommended one whose reasoning holds up under verification.

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

When the review item is real but resolving it **genuinely needs a human operator** — and the reviewer's recommendation is not enough to act on. The bar here has changed: the reviewer now hands you a menu of options *and* a recommended pick with reasoning. Your default is to **adopt that recommendation** (route to Incorporate). You file an Open question only when one of these holds:

- **The choice is more nuanced than the reviewer's framing.** You see a material consideration the reviewer missed — a constraint, a downstream dependency, an interaction with another part of the plan — that makes the reviewer's recommended option no longer obviously right. (Note what the reviewer missed in the Open-question entry.)
- **The options are genuinely balanced and the call needs context you don't have.** Once you've thought it through, the trade-offs really do cancel out, and picking correctly depends on product priorities, stakeholder input, or domain knowledge that isn't in the plan or the review.
- **The fix requires information not in the plan or the review** — a sample external payload, an ops confirmation, a stakeholder decision — that no amount of reasoning from the current artifacts can supply.
- **The fix is a "promote this open question to a foundational decision" recommendation** — promotion is itself a planning move the human should make.

What is **no longer** an automatic Open-question signal: "the reviewer offered multiple alternatives." The reviewer is *supposed* to offer alternatives — and a recommendation among them. A menu with a sound recommendation is something you act on, not something you punt.

When you do file an Open question from a review item, the entry must let a human operator make the call fast — see § "When you file an Open question" below for the required shape (the options, why the human is needed, and the pros/cons that make the options roughly equal). Use the standard Open-question shape from the authoring guide. Do **not** copy the review item verbatim — distill it.

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

### When in doubt — adopt the recommendation, don't hoard Open questions

The reviewer hands you a recommended option with reasoning for every judgment-call finding. Your job is to *use* it. The failure mode this skill is calibrated against is over-conservatism — routing a finding to Open questions when the reviewer already did the thinking and the recommendation holds up. Every Open question you file instead of resolving is debt the next round inherits and a human operator has to clear.

So: when the item is a *design-shape* call — a foundational-decision shift, a schema column change, an API verb choice, a lifecycle invariant, a cross-service contract change — and **the reviewer recommended an option whose reasoning survives your verification**, default to **Incorporate**. Adopt the recommendation. Only fall back to **Open question** when the choice is genuinely more nuanced than the reviewer's framing (you see something the reviewer missed) or genuinely needs a human operator (balanced trade-offs, missing information) — per the Open-question bucket's signals above.

Default to **Minor incorporate** when the item is a *wording / consistency / stale-text / missing-cross-link / numbering* fix. These are not design decisions; routing them to Open questions just adds noise and pads the next round's triage list.

**Calibration check.** After triaging the whole review, count the buckets. Because the reviewer now supplies a recommendation for every judgment call and your default is to adopt it, the Incorporate bucket runs heavier than it used to and the Open-question bucket runs lighter — Open questions are now the *exception* (the genuinely-needs-a-human residue), not the default home for every shape question. Use the design-plan bands below; the impl-plan version of this skill uses different bands.

**These bands assume a review with ≳8 distinct items.** On a smaller review (a handful of findings), the percentages are statistical noise — judge each item on its merits and don't force the distribution to match a band.

A healthy design-plan triage is roughly:

- **20–40% Incorporate (full)** — material edits that earn a Decisions-log entry, including adopted reviewer recommendations on shape questions. This bucket runs heavier than under the old "frame, don't resolve" posture — that is intended.
- **15–30% Minor incorporate** — mechanical fixes (typos, marketing-language purges, missing forward-links, stale wording from superseded rounds, numbering repairs).
- **10–25% Open question** — only the genuinely-needs-a-human residue: shape questions where the reviewer's recommendation didn't survive scrutiny, the trade-offs are truly balanced, or required information is missing. If this bucket is large, you are probably being too conservative — re-check that you adopted every sound recommendation.
- **0–10% Reviewer-wrong** — a small but non-zero rate is normal. A rate above ~20% is a signal that the review itself should be re-run rather than triaged; surface this to the user.
- **5–15% Ignore** — duplicate / out-of-scope / over-engineering / authoring-guide-contradicting / stylistic items.

Combined Incorporate + Minor incorporate is typically 35–70%.

Out-of-band signals:

- If **Open question is holding more than ~35% of items**, re-triage the boundary cases — you are being too conservative. For each Open-question item, ask: *did the reviewer recommend an option, and does its reasoning survive verification?* If yes, it belongs in Incorporate. Open questions are the residue of genuine human-needed calls — not the default home for findings you'd rather not commit to. The whole point of running review → integrate-feedback is to converge the plan with minimal human intervention.
- If **Incorporate (full) is holding more than ~45%**, re-check whether you're making fresh design calls the reviewer did *not* recommend — adopting a reviewer's recommendation is in scope, inventing your own resolution is not. Also check you're not promoting Minor incorporates.
- If **Minor incorporate is holding more than ~50%**, the review may be heavy on style / wording nits (a sign the plan is structurally sound and the review is flailing for findings). The triage is fine, but flag the pattern in the report — the user may want a different review prompt next time.
- If **Reviewer-wrong is holding more than ~20%**, the review's reliability is the bottleneck — stop, report the rate to the user, and let them decide whether to re-run the review with a different reviewer / prompt rather than continue triaging a noisy artifact.

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

When you file an Open question:

- Place it at the end of the existing Open-questions list, numbered sequentially. Do not renumber existing entries.
- **Tag at creation** with exactly one of `[blocks-v1]`, `[blocks-impl]`, `[deferred]`, `[exploratory]` per [design-plan.md § "Open questions" → "Severity tag taxonomy"](../specs/design-plan.md). Pick the most blocking tag that applies. If the reviewer's framing makes the tag obvious, use it; if not, default to `[blocks-impl]` for shape questions whose answer the impl plan needs and `[exploratory]` for context-only mentions.
- **Make the entry decision-ready.** Because you are surfacing this *instead of* deciding it, the entry must give a human operator everything they need to make the call in one sitting. Include:
  - **The options.** Enumerate the realistic resolutions (carry the reviewer's menu forward, refined by your own analysis — don't just paste it).
  - **Why this needs a human.** State plainly why you didn't adopt the reviewer's recommendation: the consideration the reviewer missed, the missing information, or the fact that the trade-offs are genuinely balanced. Be specific — "needs product input on X" beats "judgment call."
  - **Pros and cons per option.** For each option, the concrete upside and the concrete cost — enough that the human can see *why the options are roughly equal* and what tips the balance. If one option is mildly preferable, say so and say what would have to be true for the other to win.
- Also include: the tag (in square brackets after the bold title); the question (one sentence); the named blockee for `[blocks-v1]` / `[blocks-impl]` entries; or explicit "deferred until X" for `[deferred]` entries.
- Cross-link to the originating review item by quoting a short phrase from the review (in italics) so the human can trace it back.

When you ignore:

- Do not edit the plan. Track the item only in your final chat report.

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
4. **Verify every Incorporate-bucket item** before applying it. For each, confirm the cited section / cross-reference / convention claim against the actual source per § "Verify before incorporating." Items whose cited evidence doesn't hold move to **Reviewer-wrong** — do not edit the plan based on a claim that didn't survive verification. Minor incorporate, Open question, and Ignore items skip verification (see § "When verification is required").
5. **Calibration check.** Count the buckets against the design-plan bands in § "The triage rule" → "Calibration check" (target ~20–40% Incorporate, ~15–30% Minor, ~10–25% Open question, 0–10% Reviewer-wrong, ~5–15% Ignore). If Open question holds >35%, re-triage boundary cases — for each, check whether the reviewer recommended an option whose reasoning survives verification; if so, it belongs in Incorporate. If Incorporate (full) holds >45%, re-check for fresh design calls the reviewer did not recommend. If Reviewer-wrong holds >20%, **stop and surface this to the user** — the review may need to be re-run rather than triaged.
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

A bulleted summary of items filed for the next round — these should be the exception, not the bulk of the triage. Each bullet: the question (one short phrase) + why it needed a human rather than the reviewer's recommendation (the consideration the reviewer missed, the missing information, or the genuinely-balanced trade-off). If this section is long, double-check you didn't punt recommendations you could have adopted.

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

The **Trellis instruction precedence chain** (`~/.trellis/instructions.md` → `<repo-root>/.trellis/instructions.md` → instructions supplied with this invocation) is applied by `SKILL.md` before this file is dispatched, so it is already in effect; honor it. Canonical statement — the single source shared by every instruction file and subagent brief: [instruction-precedence.md](../specs/instruction-precedence.md).