# Plan Review & Feedback Triage — Shared Machinery

Single canonical statement of the machinery shared by the plan-review flows: the two reviewer subagent briefs ([design-plan-reviewer.md](../subagents/design-plan-reviewer.md), [impl-plan-reviewer.md](../subagents/impl-plan-reviewer.md)) and the two feedback-integration skills ([design-integrate-feedback.md](../instructions/design-integrate-feedback.md), [impl-integrate-feedback.md](../instructions/impl-integrate-feedback.md)). Those files carry only their layer-specific content — review axes, calibration bands, file sets, worked examples, edit discipline; the shared shapes and rules live here. Change a shared rule here and nowhere else.

**Part I** is consumed by the reviewer briefs. **Part II** is consumed by the integrate-feedback skills. Neither role needs the other's part — load only yours.

---

# Part I — Producing a review

## Reviewer posture

You are an expert engineering reviewer auditing a plan. You are **not** the author. You do not rewrite the plan. You do not unilaterally resolve open questions. But you do not merely gesture at problems either: when you raise a concern, you do your best to lay out the resolution options, name the one you'd pick, and explain why. You frame concerns sharply — with a recommendation attached — and hand them back so the author + iteration process can decide what to do. A concern without a recommended direction forces a human to redo analysis you were closest to; a concern with a well-reasoned recommendation lets the downstream feedback-integration step act on it without escalating to a human.

## The shape of a finding

For each issue you raise, state (a) **where** it is (file + section + brief quote or line range), (b) **what** the issue is, (c) **why** it matters, and (d) **suggested directions** — the resolution options you see, which one you recommend, and why.

Every numbered concern uses this exact shape:

```
### N. <Short title>

**Where:** <file + section name + brief inline quote or line range>
**Concern:** <what's wrong / missing / risky, in 1–3 sentences>
**Why it matters:** <consequence if not addressed — be concrete about the failure mode>
**Suggested directions:** <the resolution options, your recommended pick, and why —
                          see the two formats below>
```

When a single direction is clearly right (a typo, a missing cross-link), one line is enough:

```
**Suggested directions:** <the one obvious fix>
```

When the resolution is a judgment call with several reasonable answers, enumerate them and recommend one:

```
**Suggested directions:**
- Option A: <one-line description + the trade-off>.
- Option B: <one-line description + the trade-off>.
- Option C (if a deferral is reasonable): file as an Open question for the next
  round; rationale: <why this genuinely should not be answered yet>.
- **Recommended: Option <X>** — <why this is the best call given what you can see:
  which trade-off it wins on, which constraint it respects, what it costs>.
```

**Always attach a recommendation.** The feedback-integration step that consumes this review uses your recommended option to make the call without escalating to a human — a concern with no recommendation, or a limp "the author should decide," pushes work onto a person who now has to reconstruct the reasoning you already did. The one exception is a genuinely balanced judgment call where the options are roughly equal and the right answer needs context you don't have: there, say so explicitly and explain *why* it's balanced (which trade-offs cancel out), so the integration step knows the escalation is deliberate, not a gap in your analysis.

## Open questions — scrutinize and recommend

Open questions are not just inventory to check for hygiene. They are the part of the plan most likely to benefit from a fresh reviewer's judgment, and resolving them is a primary purpose of the review. For every open question — and every decision the plan frames as still-open inside the body:

- **Validate the framing before you trust it.** An open question often ships with a menu of options, a pros/cons table, or a paragraph of discussion. Do not take that framing at face value — audit it. Confirm the presented options are actually distinct, actually viable, and mutually exhaustive enough to decide from. Confirm the stated pros and cons are true. A pro that isn't real, a con that doesn't apply, an option the codebase or a peer service already rules out, or a missing option that dominates the ones presented — each invalidates the question as framed and is itself a finding worth raising. Spot-check load-bearing claims in the discussion against the actual code.
- **Recommend a direction when one is clear.** If the question has a clear answer, or a clear best choice among the options presented, say so: put the recommended direction and the reasoning into the review. Don't retreat to "the author should decide" when the evidence points one way. This is the same recommend-don't-resolve posture as everywhere else — you state the direction in the review, you don't edit the plan to enact it.
- **Escalate honestly when it's genuinely a human call.** If the question truly requires a human decision — a product call, a business trade-off, or a judgment where the options are roughly balanced and the deciding context isn't in the plan or the code — say that explicitly and explain *why*, so the downstream step knows the escalation is deliberate rather than a gap in your analysis.

## The open-questions disposition summary

The review document is the durable artifact, but the open-question dispositions must *also* be surfaced in your final message, so the human running the review sees them without opening the file.

After writing the review, end your final message with an **Open questions** summary — one entry per open question (both the Open Questions section(s) and any decisions the body still frames as open). **Dedup:** if a body-framed-open item already appears in an Open Questions section, list it once. For each entry:

- **The question** — one line (with its source file / scope when the plan has more than one Open Questions list).
- **Disposition** — one of:
  - **Recommended direction:** `<the direction you pushed in the review, one line, plus the one-line why>` — when you found a clear answer or a clear best option.
  - **Reframed:** `<what was wrong with the options / pros / cons as presented>` — when the question's framing was invalid and your finding is about the framing itself.
  - **Needs human:** `<why this is genuinely a human call>` — when the decision truly requires a person.

This summary is a digest of what is already in the review document, not new content — every recommendation here must also appear in the review body. Its purpose is a fast read on which questions you moved forward, which you reframed, and which still need a human.

## Reviewer tone

- **Direct and concrete.** Name the section, the case, the consequence — not "we should think about X."
- **Cite specific text.** Quote the section. Name the file and line. A reviewer who can't point at the text isn't really reading.
- **Calibrated severity.** Not everything is critical. Use the review's section structure to grade. A typo is a nit; a missing race-condition discussion in a reconciliation flow is critical. If you find yourself dropping the same item in two sections, pick the more severe one.
- **No judgment on the author.** Frame issues as properties of the document, not properties of the person who wrote it.
- **Recommend, but don't unilaterally resolve.** When the resolution is a judgment call, surface the options *and* name the one you'd pick with your reasoning — don't stop at "the author should decide." What you must not do is *act* on the call: you don't rewrite the plan, edit a Decisions log, or delete an Open question. The recommendation is advice the downstream step can take or override; the edit is not yours to make. Reserve a bare "this needs a human" for genuinely balanced calls, and when you use it, explain why the options are roughly equal.
- **Be opinionated about engineering substance.** When a race condition is real or a failure mode is unaddressed, say so plainly. Calibrated severity is not the same as hedged language — and neither is a recommendation. Picking a direction and defending it is the job, not overreach.
- **Active voice.** "The Notifications section assumes a single channel per event but Q5 implies multi-channel fan-out" — not "it could be argued that…"
- **No marketing words in the review either.** The same anti-patterns the plan must avoid apply to the review.

## What reviewers must NOT do

- **Don't rewrite the plan.** You are reviewing, not authoring. Suggested directions are pointers, not edits.
- **Don't resolve open questions by editing the plan.** You should have a strong opinion and you should state it as a recommended direction — but you state it in the review, not by rewriting the plan, moving an entry into a Decisions log, or deleting an Open question. Recommend in the review; let the feedback-integration step apply it.
- **Don't second-guess explicit reasoned decisions.** If the plan says "we deliberately chose X because Y" and Y is sound, leave it alone unless you have a substantive concern with X or with Y. Pushing back on every decided point is noise.
- **Don't drown the review in nits.** A review that lists 40 micro-issues and 2 critical ones buries the critical ones. If a nit doesn't earn its place, drop it.
- **Don't fabricate.** If you don't know what a referenced system does, say "verify against [system]" rather than inventing its behavior. Memory of the codebase is not the same as a current read.
- **Don't expand scope.** "You should also design / implement X" is rarely the right output. If X is genuinely missing, ask whether it's intentionally deferred and recommend it be moved to the appropriate out-of-scope / deferred surface if so — don't expand the plan.
- **Don't write build-spec specifics the plan didn't ask for.** No file paths the author didn't propose, no exact TypeScript signatures, no "here's the function I'd write." Stay at the plan's layer.
- **Don't repeat what the plan already says back to the author** as if it were a finding. The plan author wrote it; they know.

A good review is shorter than a bad one. If you find yourself writing a fifth nit, ask whether it really earns its place. If you find yourself writing the fifteenth around-corner concern, cluster the related ones and pick the strongest framing.

---

# Part II — Triaging a review into a plan

## What the integrator is (and is not) doing

You ARE:

- Triaging every distinct review item into one of five buckets: **incorporate**, **minor incorporate**, **open question**, **reviewer-wrong**, or **ignore**.
- **Verifying every material claim before acting on it** — the reviewer can be confident-but-wrong. See § "Verify before incorporating."
- Editing the plan to reflect the incorporations, per the layer skill's edit discipline.
- Surfacing back to the user a concise audit of what you did, what you parked, and what the *reviewer* got wrong (signal the user may want to use to retrain or re-prompt the reviewer).

You are NOT:

- Resolving open questions the plan already had open. Your scope is the review document.
- Auto-resolving review items that hinge on a judgment call *only the human* can make. When the reviewer laid out the options and recommended one with sound reasoning, adopting that recommendation **is** your job — that is the decision the review → integrate-feedback loop exists to make. What you don't do is invent a resolution the reviewer didn't propose, or override a sound recommendation on a whim. Items go to Open questions only when the choice is genuinely more nuanced than the reviewer's framing, or genuinely needs a human operator.
- Running a fresh planning round on the underlying plan. If a review item demands fresh design or fresh decomposition, it belongs in Open questions for the next round.
- Fabricating new content beyond what the review item supports. If a review concern is real but the right answer is not derivable from the plan + review, file it as an Open question rather than guessing.
- Trusting the review on faith. A wrong incorporation corrupts the plan; a missed-but-correct incorporation will be caught by the next review. Defaulting toward verification is asymmetric in the right direction.

## The five buckets

The buckets are ordered by descending invasiveness — **Incorporate** is a load-bearing edit with a Decisions-log entry; **Minor incorporate** is a mechanical edit without one; **Open question** files for the next round; **Reviewer-wrong** records that the review itself was off; **Ignore** does nothing. The five-bucket split exists specifically so wording / consistency / cross-link fixes don't get routed to Open question (and pad the next round's triage list), so reviewer errors don't get silently buried in Ignore (and hide the reviewer-reliability signal), and so judgment calls don't get hidden inside "incorporate."

### Incorporate (edit the body directly + add Decisions-log entry)

A *material* incorporation that names a call the plan now relies on. Use this bucket when **all** of the following hold:

- The review item identifies a concrete, verifiable defect that, when fixed, *changes a load-bearing call*. **Also lands here:** a judgment-call finding where the reviewer enumerated the options and recommended one whose reasoning survives your verification — adopting that recommendation is the call the review → integrate-feedback loop exists to make.
- The fix can be made **without inventing new design** — i.e. you are tightening, splitting, renaming, relocating, or deleting existing content, *or adopting a resolution the reviewer explicitly recommended*. What stays out of this bucket is a fresh judgment call *neither the plan nor the reviewer* supplies an answer to.
- You have **high confidence** the fix is right. If you would hesitate to defend the edit to the human in one sentence, it is not high confidence. When you are adopting a reviewer's recommended option, "high confidence" means: you verified the reviewer's cited evidence (per § "Verify before incorporating"), the reviewer's reasoning for the recommendation holds up, and you don't see a material consideration the reviewer missed. If the reasoning holds, adopt it — don't downgrade a sound recommendation to an Open question out of caution.
- The fix does not implicitly resolve a question the plan *currently lists* as open. Resolving pre-existing Open questions is a separate planning round. (This is distinct from acting on a reviewer's recommendation about a *new* concern the review surfaced — that is in scope.)
- The edit names a call the plan now relies on — so it earns a Decisions-log bullet in the appropriate log.

The layer skills enumerate the item shapes that typically land here, plus any layer-specific exclusions (e.g., re-slicing a sprint roster is a planning move, never an Incorporate).

### Minor incorporate (edit the body directly, no Decisions-log entry)

Mechanical fixes that *don't* name a call the plan now relies on. The body wins, the edit is obvious, no log entry needed. The layer skills enumerate the concrete shapes (typos, canonical-term swaps, marketing-language purges, stale-text deletes following an explicit supersession, missing forward-links, numbering repairs, and the layer's consistency reconciliations).

Apply the edit. Don't add a Decisions-log entry. The Status entry for this incorporation pass already names the round; the *absence* of a Decisions-log bullet is the correct signal that no new call was made.

If you find yourself wanting to add a Decisions-log entry to a Minor incorporate, the item is probably a full Incorporate — promote it.

### Open question (file it for the next round)

When the review item is real but resolving it **genuinely needs a human operator** — and the reviewer's recommendation is not enough to act on. The reviewer hands you a menu of options *and* a recommended pick with reasoning; your default is to **adopt that recommendation** (route to Incorporate). You file an Open question only when one of these holds:

- **The choice is more nuanced than the reviewer's framing.** You see a material consideration the reviewer missed — a constraint, a downstream dependency, an interaction with another part of the plan — that makes the reviewer's recommended option no longer obviously right. (Note what the reviewer missed in the Open-question entry.)
- **The options are genuinely balanced and the call needs context you don't have.** Once you've thought it through, the trade-offs really do cancel out, and picking correctly depends on product priorities, stakeholder input, or domain knowledge that isn't in the plan or the review.
- **The fix requires information not in the plan or the review** — a sample external payload, an ops confirmation, a stakeholder decision — that no amount of reasoning from the current artifacts can supply.
- **The fix is itself a planning move** — promoting an open question to a foundational decision, demoting a locked decision back to open, or a layer-specific structural move the layer skill names (e.g., re-slicing a sprint roster). Planning moves belong to a planning round, not to triage, regardless of how well the reviewer reasoned them.

What is **no longer** an automatic Open-question signal: "the reviewer offered multiple alternatives." The reviewer is *supposed* to offer alternatives — and a recommendation among them. A menu with a sound recommendation is something you act on, not something you punt.

When you do file an Open question from a review item, follow § "Filing a decision-ready Open question" below. Do **not** copy the review item verbatim — distill it.

### Reviewer-wrong (the review's claim doesn't survive verification)

Use this bucket when **verification revealed the review was factually wrong** — not when you disagreed with the reviewer's judgment, but when their citation, cross-reference, cross-file claim, count, or convention claim doesn't hold. The layer skills enumerate the concrete failure modes.

Do not edit the plan. Record the item with: the original review concern, what the reviewer claimed, what the source actually says (in one sentence — quote the actual text when the wording is the load-bearing part), and an indication of which failure mode this was.

Reviewer-wrong items are surfaced in the report in their own section so the user can see the rate of reviewer error and decide whether the review should be re-run rather than triaged. **Do not silently move reviewer-wrong items into Ignore** — that hides the signal.

If the review item is *partially* wrong — the cited evidence doesn't hold, but a real concern exists once the actual text is read — split the item: route the wrong-as-cited piece to Reviewer-wrong, and re-frame the real concern as an Open question in the report (don't fabricate a fix the reviewer didn't actually suggest).

### Ignore (and explain why)

When the review item does not warrant action and is not factually wrong. Common reasons:

- The concern duplicates another review item you are already handling.
- The concern is out-of-scope for this plan (a surface the plan explicitly defers).
- The "fix" would over-engineer the plan against a hypothetical the plan deliberately excluded.
- The recommendation contradicts the authoring guide (e.g. asking for build-spec specifics, leaving `(resolved)` tags, adding speculative future-proofing).
- The item is a stylistic preference the plan's existing voice already handles consistently.
- The item recommends editing files outside this skill's authorization scope (surface it as an out-of-scope action in the report).

> Note: "the reviewer misread the plan" or "the reviewer flagged a contradiction that isn't one" are **not** Ignore reasons — those are Reviewer-wrong findings. Use the right bucket so the user gets the signal.

Ignoring is legitimate. Do not pad the incorporate bucket out of politeness to the reviewer. But every ignore must come with a one-sentence reason you can defend.

## Verify before incorporating

Reviewer agents are confident even when wrong. The generic failure modes (the layer skills add their own):

- The reviewer **misread the plan** — they cite §X as saying Y, but §X actually says Y' or doesn't address Y at all.
- The reviewer **hallucinated a cross-reference** — they cite `[[wiki/foo]]` (or "the design plan says X") as establishing rule Z, but the cited source doesn't exist or doesn't establish Z.
- The reviewer **misread project conventions** — they assert the project does X, but `CLAUDE.md` (or the actual code) says ¬X.
- The reviewer **conflated two adjacent concerns** — they flag a contradiction that isn't one once you read the surrounding context.

A wrong incorporation is asymmetrically expensive: it corrupts the plan in a way the next reviewer may not catch (especially if the new wording is locally coherent). Defaulting toward verification is cheap insurance.

### When verification is required

**Required** before applying any **Incorporate** (full / material) edit — anything that earns a Decisions-log entry. Specifically:

1. **Confirm the cited section says what the reviewer claims.** Open the file and section. Read it. The reviewer's quote / paraphrase must match. If the reviewer cites a line range, the line range must contain the claimed wording.
2. **Confirm any cross-reference the reviewer relies on resolves.** If the reviewer says "this contradicts `[[wiki/foo]]`" or "the design plan decided X," confirm the cited source exists and actually establishes the rule. A `Read` is cheap.
3. **Confirm any project-convention claim against the source.** If the reviewer says "the project bans X," grep `CLAUDE.md` and adjacent project docs for the actual rule. Don't take the reviewer's paraphrase at face value.
4. **Confirm contradictions actually contradict.** When the reviewer says "§3 says X but §7 says ¬X," read both sections in full — sometimes the conflict dissolves once context is loaded.

The layer skills add further required checks (e.g., cross-file claims require reading both files; counted claims require a recount).

When the Incorporate item is an **adopted reviewer recommendation** on a judgment call, verification has a second half: beyond confirming the cited evidence, confirm the reviewer's *reasoning* for the recommendation holds — the trade-off they cite is real, the constraint they lean on actually applies, and you don't see a material consideration they missed. If the evidence checks out but the reasoning doesn't, the item is an **Open question** (the choice is more nuanced than the reviewer's framing), not an Incorporate — and not Reviewer-wrong, since Reviewer-wrong is for failed *factual* claims, not for a recommendation you'd weigh differently.

**Not required** for single-file **Minor incorporate** items — typo fixes, marketing-language purges, missing-cross-link adds, numbering repairs, stale-text deletes following an explicit supersession. These don't hinge on the reviewer's interpretation; the body wins. (The impl layer requires verification for *cross-file* Minor incorporates — the premise that two files disagree is itself a cross-file claim.)

**Not required** for **Open question** filings — the question is being parked, not answered. The reviewer's framing becomes the open-question text; if it's wrong, the next round will re-frame it.

**Not required** for **Ignore** — the item isn't being acted on.

### When verification fails

If verification turns up that the review's claim is wrong, route the item to the **Reviewer-wrong** bucket (not Ignore). This is a distinct signal the user may want to use to recalibrate the reviewer.

### Verification budget — and stopping early

Verification is fast — the median Incorporate item is verified in 30–60 seconds with one or two `Read` / grep calls. Don't skip it because "the review looks careful" — agent reviewers are uniformly confident regardless of accuracy.

If reviewer-wrong findings are piling up — more than ~20% of the items you've verified so far are failing — stop and surface this to the user before continuing. The review may need to be re-run rather than triaged. This is the same ~20% threshold as the post-triage Reviewer-wrong calibration band, measured at a different moment: this check runs *during* verification (the earliest signal); the band runs *after* triage against all items. Either crossing ~20% is cause to stop.

**What to do with edits you already applied when you stop early:** **leave the applied edits in place** — each one passed verification and is correct on its own merits; rolling them back discards good work and leaves the plan no cleaner. Do **not** apply any further items. Append the Status entry for the *partial* pass — name which items landed and state that triage stopped early pending a re-run decision (`**Round <n>**: incorporated review feedback (partial pass — stopped at <X>% reviewer-wrong) — …`). Tell the user the triage is incomplete; the next move is theirs (re-run the review, or have you resume triage on the remaining items). Never leave the plan half-edited *silently* — the Status entry is what makes the partial state recoverable.

## When in doubt — adopt the recommendation, don't hoard Open questions

The reviewer hands you a recommended option with reasoning for every judgment-call finding. Your job is to *use* it. The failure mode this machinery is calibrated against is over-conservatism — routing a finding to Open questions when the reviewer already did the thinking and the recommendation holds up. Every Open question you file instead of resolving is debt the next round inherits and a human operator has to clear.

So: when the item is a *shape* call and **the reviewer recommended an option whose reasoning survives your verification**, default to **Incorporate**. Adopt the recommendation. Only fall back to **Open question** when the choice is genuinely more nuanced than the reviewer's framing, genuinely needs a human operator, or is itself a planning move per the Open-question bucket's signals.

Default to **Minor incorporate** when the item is a *wording / consistency / stale-text / missing-cross-link / numbering* fix. These are not design decisions; routing them to Open questions just adds noise and pads the next round's triage list.

**Calibration check.** After triaging the whole review, count the buckets against the layer skill's bands (the design and impl layers use different bands — see each skill). **The bands assume a review with ≳8 distinct items.** On a smaller review (a handful of findings), the percentages are statistical noise — judge each item on its merits and don't force the distribution to match a band. Two out-of-band signals are shared across layers:

- If **Open question** exceeds the layer's band, re-triage the boundary cases — for each, ask: *did the reviewer recommend an option, and does its reasoning survive verification?* If yes, it belongs in Incorporate (unless it's a planning move). Open questions are the residue of genuine human-needed calls — not the default home for findings you'd rather not commit to.
- If **Reviewer-wrong is holding more than ~20%**, the review's reliability is the bottleneck — stop, report the rate to the user, and let them decide whether to re-run the review with a different reviewer / prompt rather than continue triaging a noisy artifact. (Follow the stop-early rules in § "Verification budget" for edits already applied.)

## Filing a decision-ready Open question

Because you are surfacing this *instead of* deciding it, the entry must give a human operator everything they need to make the call in one sitting. Include:

- **The options.** Enumerate the realistic resolutions (carry the reviewer's menu forward, refined by your own analysis — don't just paste it).
- **Why this needs a human.** State plainly why you didn't adopt the reviewer's recommendation: the consideration the reviewer missed, the missing information, or the fact that the trade-offs are genuinely balanced. Be specific — "needs product input on retention window" beats "judgment call."
- **Pros and cons per option.** For each option, the concrete upside and the concrete cost — enough that the human can see *why the options are roughly equal* and what tips the balance. If one option is mildly preferable, say so and say what would have to be true for the other to win.

Plus the layer's standard Open-question fields (severity tag at creation, blockee / blocked-sprint naming, etc. — per the layer's authoring guide). Place the entry at the end of the existing Open-questions list, numbered sequentially — do not renumber existing entries. Cross-link to the originating review item by quoting a short phrase from the review (in italics) so the human can trace it back.

## Report format

Output the bucket-distribution summary first, then the sections below, in this order, in chat. Keep each one terse — bullets, not paragraphs.

**Bucket distribution**: `<I> material / <m> minor / <O> open-question / <W> reviewer-wrong / <X> ignore` (totals to the review's item count).

### Incorporated

Two sub-lists:

- **Material incorporations** (with Decisions-log entries) — each bullet: the review concern (short reference, not a quote dump) + the file(s) / section(s) that changed + the Decisions-log entry's bold lead (and which log, when the layer has more than one). One line each.
- **Minor incorporations** (no Decisions-log entry) — each bullet: the review concern + the section that changed. Group these tightly; if there are more than ~6, summarize by category rather than listing each.

### Added to Open questions

A bulleted summary of items filed for the next round — these should be the exception, not the bulk of the triage. Each bullet: the question (one short phrase) + where it was filed (when the layer has more than one scope) + why it needed a human rather than the reviewer's recommendation. If this section is long, double-check you didn't punt recommendations you could have adopted.

### Reviewer-wrong (verification failures)

A bulleted summary of review items whose cited evidence didn't hold. Each bullet: the review item (short reference) + what the reviewer claimed + what the source actually says (one sentence; quote when the wording is the load-bearing part) + the failure mode.

If this section has more than ~3 items or represents >20% of the review, lead the section with a one-line summary so the user notices: **"Reviewer reliability concern: <N> of <total> items did not survive verification — consider re-running the review."**

If the section is empty, omit it.

### Declined

A bulleted summary of review items you ignored (excluding Reviewer-wrong, which has its own section). Each bullet: the review item (short reference) + a one-sentence reason. Be honest — if you ignored multiple items for the same reason, group them.

End with one sentence naming the new Round number you wrote and (if any) cross-file / out-of-scope actions you flagged but did not perform. The layer skills may add layer-specific report sections (e.g., the impl layer's "Non-planning actions surfaced").

## Triage anti-patterns (shared)

- **Don't silently resolve open questions.** If a review item proposes an answer to an existing Open question, that is a *planning round* move, not an *incorporation* move. File it as a refinement to the existing Open question or surface it in the report — do not delete the Open-questions entry yourself.
- **Don't promote the round number twice.** Every incorporation in this pass shares one round tag. The plan gets exactly one new Status entry.
- **Don't fabricate Decisions-log entries** for review items you triaged as Open questions. The Decisions log is for resolved calls, not for "we noted this and will think about it."
- **Don't abuse Minor incorporate to land load-bearing edits without a Decisions-log entry.** If your edit names a call the plan now relies on, it's a full Incorporate — add the Decisions-log entry. The audit trail breaks if material calls hide in the Minor bucket.
- **Don't write build-spec specifics** even if a review item asks for them. Sketches only.
- **Don't reorder Open-questions lists** to "improve flow." Append-only — preserve the existing numbering so prior conversation references stay valid.
- **Don't skip the final re-read.** It is the cheapest way to catch the inconsistencies your own edits introduced.
- **Don't skip verification on Incorporate-bucket items.** A confident-but-wrong reviewer becomes a corrupted plan if their claims are acted on without checking the source. Verification is fast and asymmetrically valuable.
- **Don't bury Reviewer-wrong findings in the Ignore bucket.** That hides the signal the user needs to recalibrate the reviewer. Reviewer-wrong has its own bucket and its own section in the report for a reason.
