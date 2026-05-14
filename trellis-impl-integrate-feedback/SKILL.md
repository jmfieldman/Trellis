---
name: trellis-impl-integrate-feedback
description: Integrate Feedback into an Implementation Plan
argument-hint: <path-to-impl-plan-directory> <feedback-file>
disable-model-invocation: true
---

# Skill — Incorporate review feedback into an implementation plan

You are continuing an in-progress implementation plan by reconciling it against a separate review document. The plan is the living artifact (anatomy and rules described in [Implementation Plan Documents — Authoring Guide](../trellis-impl-create/implementation-plan.md)); the review document is one round of external critique against it. Your job in this invocation is **triage and incorporation**, not full re-design or fresh sprint planning — you are processing the review, not running a fresh planning round.

The implementation plan is a *directory* of files (`overview.md`, sprint files, `progress.md`, optionally `post-mortem.md`), not a single document. Many incorporations touch multiple files; the discipline below treats the directory as a coherent system.

## Inputs

You will be given two file paths:

1. **`<plan-dir>`** — the directory of the implementation plan in progress: $0
2. **`<feedback-path>`** — a review of that plan: $1

The review is typically organized as: Executive summary, High-level concerns, Cross-sprint coherence, Per-sprint findings (one block per sprint), optional Per-step findings, Substance/authoring-guide deviations, Minor nits, Suggested next-round focus. Each item typically has a *Where*, *Concern*, *Why it matters*, and *Suggested directions* (often a menu of options).

If one of the paths is not provided, tell the user they forgot an argument to this skill invocation, and stop.

## What you are doing (and not doing)

You ARE:

- Reading every file in the plan directory before doing anything else.
- Triaging every distinct review item into one of five buckets: **incorporate**, **minor incorporate**, **open question**, **reviewer-wrong**, or **ignore**.
- **Verifying every material claim before acting on it** — the reviewer can be confident-but-wrong. See § "Verify before incorporating" below. For impl plan reviews this is especially load-bearing because cross-sprint coherence claims often hinge on the reviewer correctly reading two files at once.
- Editing the plan's body (overview, sprint files, `progress.md`), Decisions logs (overview-level + per-sprint), Open questions (overview-level + per-sprint), and Status log to reflect the incorporations.
- Keeping `progress.md` and per-sprint Progress sections in lockstep when step structure changes.
- Surfacing back to the user a concise audit of what you did, what you parked, and what the *reviewer* got wrong (which is signal the user may want to use to retrain or re-prompt the reviewer).

You are NOT:

- Resolving open questions the plan already had open. Your scope is the review document.
- Auto-resolving review items that hinge on a judgment call *only the human* can make. When the reviewer laid out the options and recommended one with sound reasoning, adopting that recommendation **is** your job — that is the decision the review → integrate-feedback loop exists to make. What you don't do is invent a resolution the reviewer didn't propose, or override a sound recommendation on a whim. Items go to Open questions only when the choice is genuinely more nuanced than the reviewer's framing, or genuinely needs a human operator — see § "The triage rule".
- Running a new planning round on the underlying design or fleshing out a stub sprint to execution-ready. If a review item demands fresh design or fresh sprint-step decomposition, file it as an Open question for the relevant sprint.
- Re-slicing the sprint roster on your own. If a review item recommends a split / fold / re-order, file it as an Open question (or in the report) — re-slicing is a planning-round move, not a triage move. The single exception: a review item that flags a clear naming or numbering bug (e.g., a sprint file that doesn't match the roster) can be fixed mechanically.
- Editing the source design plan. If a review item routes back to a design-level call, surface it in the report — don't touch the design plan from here.
- Fabricating new content beyond what the review item supports. If a review concern is real but the right answer isn't derivable from the plan + review, file it as an Open question rather than guessing.
- Trusting the review on faith. A wrong incorporation corrupts the plan; a missed-but-correct incorporation will be caught by the next review. Defaulting toward verification is asymmetric in the right direction.

## Verify before incorporating

Reviewer agents are confident even when wrong. Impl plan reviews are *especially* prone to confident-but-wrong findings because the cross-sprint coherence axis (the highest-leverage axis in the review skill) requires the reviewer to read two or more files in lockstep — a hard task that's easy to get subtly wrong. The most common failure modes:

- The reviewer **misread one of the two files** in a cross-sprint claim — they say "Sprint 04's Prerequisites lists `X` but Sprint 02's Deliverables don't include it," but Sprint 02 actually does include it (under a different heading or with slightly different wording the reviewer didn't recognize as the same thing).
- The reviewer **conflated the overview's Locked Decisions with a sprint's Locked Decisions** — they flag a "contradiction" that's actually a refinement (the sprint pinning a value the overview left general).
- The reviewer **hallucinated a cross-reference** — they cite `[[wiki/foo]]` or claim "the design plan says X" but the cited source doesn't exist or doesn't establish X.
- The reviewer **misread project conventions** — they assert `CLAUDE.md` bans Y, but `CLAUDE.md` actually says Y is fine in context Z.
- The reviewer **counted wrong** — they say "Sprint 06's Implementation Steps lists 8 steps but `progress.md` lists 6," but a quick recount shows both have the same number under different formatting.
- The reviewer **flagged stale wording that was already purged** — they're reading an older copy of the plan than what's in the directory.

A wrong incorporation is asymmetrically expensive in the impl-plan setting: a "fix" applied to the wrong sprint based on a misread cross-sprint claim corrupts that sprint's coherence in a way the next reviewer may not catch. Defaulting toward verification is cheap insurance.

### When verification is required

**Required** before applying any **Incorporate** (full / material) edit — anything that earns a Decisions-log entry. Specifically:

1. **Confirm the cited section says what the reviewer claims.** Open the file and section. Read it. The reviewer's quote / paraphrase must match. If the reviewer cites a line range, the line range must contain the claimed wording.
2. **Confirm cross-file claims by reading both files.** When the review item asserts a contradiction or drift between two files (Prerequisite vs. Deliverable, overview Locked Decision vs. sprint Locked Decision, `progress.md` vs. per-sprint Progress, sprint-A surface vs. sprint-B surface), open both files and confirm the claim. Cross-file claims are the highest-error category — verify them with extra care.
3. **Confirm any cross-reference the reviewer relies on resolves.** If the reviewer says "this contradicts the design plan's foundational decision #5," confirm decision #5 says what the reviewer claims. A `Read` of the design plan is cheap.
4. **Confirm any project-convention claim against the source.** If the reviewer says "the project bans X," grep `CLAUDE.md` and adjacent project docs for the actual rule.
5. **Re-count when the review counts.** If the review claims "Sprint NN has 8 steps but `progress.md` has 6," count both yourself before acting.

When the Incorporate item is an **adopted reviewer recommendation** on a judgment call, verification has a second half: beyond confirming the cited evidence (steps 1–5 above), confirm the reviewer's *reasoning* for the recommendation holds — the trade-off they cite is real, the constraint they lean on actually applies, and you don't see a material consideration they missed (a cross-sprint dependency, a locked decision the recommendation collides with). If the evidence checks out but the reasoning doesn't, the item is an **Open question** (the choice is more nuanced than the reviewer's framing), not an Incorporate — and not Reviewer-wrong, since Reviewer-wrong is for failed *factual* claims, not for a recommendation you'd weigh differently.

**Required** before applying **Minor incorporate** items that span multiple files (e.g., a name-drift fix in Sprint 04's Prerequisites that mirrors a Deliverable in Sprint 02). Even though the edit is mechanical, the *premise* — that the two files actually disagree — is a cross-file claim and must be confirmed. Single-file Minor incorporates (typo, marketing-language purge, missing forward-link the body already names) skip verification.

**Not required** for **Open question** filings — the question is being parked, not answered. The reviewer's framing becomes the open-question text; if it's wrong, the next round will re-frame it.

**Not required** for **Ignore** — the item isn't being acted on.

### When verification fails

If verification reveals the review's claim is wrong (the cited section doesn't say what the reviewer claims; the cross-file contradiction isn't actually a contradiction; the cited cross-reference doesn't establish what the reviewer claims; the project convention isn't what the reviewer claims; the count is wrong), route the item to the **Reviewer-wrong** bucket (not Ignore). See § "The triage rule" below. This is a distinct signal the user may want to use to recalibrate the reviewer.

If the review item is *partially* wrong — the cited evidence doesn't hold, but a real concern exists once the actual files are read — split the item: route the wrong-as-cited piece to Reviewer-wrong, and re-frame the real concern as an Open question in the report (don't fabricate a fix the reviewer didn't actually suggest).

### Verification budget

This step is fast — the median Incorporate item is verified in 30–60 seconds with one or two `Read` / `grep` calls. Don't skip it because "the review looks careful" — agent reviewers are uniformly confident regardless of accuracy.

If the volume of verification is large and you find a high rate of reviewer-wrong findings (>20% of Incorporate-bucket + cross-file Minor-incorporate items fail verification), stop and surface this to the user before continuing. The review may need to be re-run rather than triaged.

---

## The triage rule

For each review item, decide which bucket. The five buckets are ordered by descending invasiveness — **Incorporate** is a load-bearing edit with a Decisions-log entry; **Minor incorporate** is a mechanical edit without one; **Open question** files for the next round; **Reviewer-wrong** records that the review itself was off; **Ignore** does nothing. The five-bucket split exists specifically so wording / consistency / cross-link fixes don't get routed to Open question (and pad the next round's triage list), so reviewer errors don't get silently buried in Ignore (and hide the reviewer-reliability signal), and so judgment calls don't get hidden inside "incorporate."

### Incorporate (edit the body directly + add Decisions-log entry)

A *material* incorporation that names a call the plan now relies on. Use this bucket when **all** of the following hold:

- The review item identifies a concrete, verifiable defect that, when fixed, *changes a load-bearing call* — locking a sprint-level decision the body had two ways of expressing; reconciling a Locked-Decisions row with a Public-surface error-table row that disagrees with it; resolving a contradiction between a sprint's Goal and its Steps; adding a missing required section the authoring guide demands when the body already implies its content (e.g., an "Out of scope" forward-link the body names elsewhere). **Also lands here:** a judgment-call finding where the reviewer enumerated the options and recommended one whose reasoning survives your verification — adopting that recommendation is the call the review → integrate-feedback loop exists to make.
- The fix can be made **without inventing new design** — i.e. you are tightening, splitting, renaming, relocating, or deleting existing content, *or adopting a resolution the reviewer explicitly recommended*. What stays out of this bucket is a fresh judgment call *neither the plan nor the reviewer* supplies an answer to.
- You have **high confidence** the fix is right. If you would hesitate to defend the edit to the human in one sentence, it is not high confidence. When you are adopting a reviewer's recommended option, "high confidence" means: you verified the reviewer's cited evidence (per § "Verify before incorporating"), the reviewer's reasoning for the recommendation holds up, and you don't see a material consideration the reviewer missed. If the reasoning holds, adopt it — don't downgrade a sound recommendation to an Open question out of caution.
- The fix does not implicitly resolve a question the plan *currently lists* as open at any scope. Resolving pre-existing Open questions is a separate planning round. (This is distinct from acting on a reviewer's recommendation about a *new* concern the review surfaced — that is in scope.)
- The fix does not require re-slicing the sprint roster. (Renaming a sprint file to match its title in the roster is mechanical and lives in Minor incorporate; splitting a sprint is a planning move.)
- The edit names a call the plan now relies on — so it earns a Decisions-log bullet (overview-level for cross-cutting calls; sprint-level for sprint-scoped calls).

Examples of items that typically land here: a contradiction between a Locked-Decisions row and a Public-surface error table (you have to pick a side); a missing `progress.md` entry for a sprint already in the roster *with steps the sprint file enumerates*; a foundational invariant restated in `overview.md`'s Architectural-invariants list when an existing sprint already verifies it but the list was missing the entry.

### Minor incorporate (edit the body directly, no Decisions-log entry)

Mechanical fixes that *don't* name a call the plan now relies on. The body wins, the edit is obvious, no log entry needed. Use this bucket when:

- The fix is a typo, casing fix, or canonical-term swap where `overview.md`'s framing block already settled the term.
- The fix is a marketing-language purge (`elegant`, `clean`, `robust`, `powerful`, `best-in-class`) the authoring guide explicitly flags.
- The fix removes a `(resolved)` tag inside Open questions and moves the entry to the Decisions log under its original round tag.
- The fix deletes stale wording from a superseded round still sitting in any file.
- The fix adds a missing forward-link in an Out-of-scope entry that already names the target sprint elsewhere.
- The fix adds a missing entry in `overview.md`'s Cross-references / dependency-graph one-liner list for an edge already drawn in the chart.
- The fix adds a missing entry in `overview.md`'s Module/Directory tree for a file an existing sprint already commits to creating.
- The fix splits a bundled Open question into two for clarity (preserves both, no decision made).
- The fix corrects a sprint Decisions-log bullet that drops a clause the body already decided.
- The fix repairs numbering (skipped Q5; duplicate `(R3)` tag; out-of-order Status entries; sprint roster row missing for an existing file).
- The fix reconciles `progress.md` ↔ per-sprint Progress drift (the master file is the source of truth; the per-sprint copy is reconciled to it, or the master is fixed if it's the broken side).
- The fix corrects a `post-mortem.md` violation that's mechanical (missing section heading for a shipped sprint; section out of sprint order). *Note*: deleting a `post-mortem.md` that was pre-created at scaffold time is a Minor incorporate; creating one when no sprint has shipped is forbidden — see Authorization scope.
- The fix renames a sprint file to match the title already used in `overview.md`'s sprint roster.
- The fix is a Status-log entry that omits the supersession callout for a re-slice it announces (add the callout, don't reopen the slice).

Apply the edit. Don't add a Decisions-log entry. The Status entry for this incorporation pass already names the round; the *absence* of a Decisions-log bullet is the correct signal that no new call was made.

If you find yourself wanting to add a Decisions-log entry to a Minor incorporate, the item is probably a full Incorporate — promote it.

### Open question (file it for the next round)

When the review item is real but resolving it **genuinely needs a human operator** — and the reviewer's recommendation is not enough to act on. The bar here has changed: the reviewer now hands you a menu of options *and* a recommended pick with reasoning. Your default is to **adopt that recommendation** (route to Incorporate). You file an Open question — at the **right scope** (overview-level for cross-sprint / feature-wide questions; sprint-level for sprint-scoped ones) — only when one of these holds:

- **The choice is more nuanced than the reviewer's framing.** You see a material consideration the reviewer missed — a cross-sprint dependency, an interaction with a locked decision, a downstream sprint the recommendation would destabilize — that makes the reviewer's recommended option no longer obviously right. (Note what the reviewer missed in the Open-question entry.)
- **The options are genuinely balanced and the call needs context you don't have.** Once you've thought it through, the trade-offs really do cancel out, and picking correctly depends on product priorities, stakeholder input, or domain knowledge that isn't in the plan or the review.
- **The fix requires information not in the plan or the review** — a sample external payload, an ops confirmation, a stakeholder decision, a peer-service contract — that no amount of reasoning from the current artifacts can supply.
- **The fix is "promote this open question to a foundational decision" or "demote this locked decision back to an open question"** — promotion / demotion is itself a planning move the human should make.
- **The fix is "re-slice these sprints"** — re-slicing is a planning-round move, out of scope for triage regardless of how well-reasoned the reviewer's recommendation is.

What is **no longer** an automatic Open-question signal: "the reviewer offered multiple alternatives" / "the Suggested directions block lists Options A/B/C." The reviewer is *supposed* to offer alternatives — and a recommendation among them. A menu with a sound recommendation is something you act on, not something you punt.

When you do file an Open question from a review item, the entry must let a human operator make the call fast — see § "When you file an Open question" below for the required shape (the options, why the human is needed, and the pros/cons that make the options roughly equal). Use the standard Open-question shape from the authoring guide. Do **not** copy the review item verbatim — distill it.

### Reviewer-wrong (the review's claim doesn't survive verification)

Use this bucket when **verification revealed the review was factually wrong** — not when you disagreed with the reviewer's judgment, but when their citation, cross-file claim, cross-reference, or convention claim doesn't hold. Specifically:

- The reviewer cited Sprint NN § X as saying Y, but Sprint NN § X says Y' or doesn't address Y.
- The reviewer asserted a cross-sprint contradiction that, once both files are read in full, isn't actually a contradiction.
- The reviewer cited the design plan or a wiki page as establishing rule Z, but the source doesn't exist or doesn't establish Z.
- The reviewer asserted a project convention (`CLAUDE.md` or similar) that the source actually contradicts.
- The reviewer's count is wrong (e.g., "Sprint 06 has 8 steps but `progress.md` has 6" when both actually have the same count).
- The reviewer flagged stale wording that was already purged in a prior round.

Do not edit the plan. Record the item with: the original review concern, what the reviewer claimed, what the source actually says (in one sentence — quote the actual text when wording is the load-bearing part), and an indication of which failure mode this was.

Reviewer-wrong items are surfaced in the report in their own section so the user can see the rate of reviewer error and decide whether the review should be re-run rather than triaged. **Do not silently move reviewer-wrong items into Ignore** — that hides the signal.

If the review item is *partially* wrong — the cited evidence doesn't hold, but a real concern exists once the actual files are read — split the item: route the wrong-as-cited piece to Reviewer-wrong, and re-frame the real concern as an Open question in the report (don't fabricate a fix the reviewer didn't actually suggest).

### Ignore (and explain why)

When the review item does not warrant action and is not factually wrong. Possible reasons:

- The concern duplicates another review item you are already handling.
- The concern is out-of-scope for this plan (e.g. a feature the plan explicitly defers via "What this plan does not try to do").
- The "fix" would over-engineer the plan against a hypothetical the plan deliberately excluded.
- The recommendation contradicts the authoring guide (e.g. asking for build-spec specifics in `overview.md`, leaving `(resolved)` tags, adding speculative future-proofing).
- The recommendation contradicts the project's `CLAUDE.md` / conventions — but verify the convention claim against the actual `CLAUDE.md` before placing the item here. If `CLAUDE.md` says something different from what the reviewer assumed, the bucket is **Reviewer-wrong**, not Ignore.
- The item is a stylistic preference the plan's existing voice already handles consistently.
- The item recommends editing the design plan or other files outside the implementation plan directory (surface it as an out-of-scope action in the report).

> Note: "the reviewer misread the plan" or "the reviewer flagged a contradiction that isn't one" are **not** Ignore reasons — those are Reviewer-wrong findings. Use the right bucket so the user gets the signal.

Ignoring is legitimate. Do not pad the incorporate bucket out of politeness to the reviewer. But every ignore must come with a one-sentence reason you can defend.

### When in doubt — adopt the recommendation, don't hoard Open questions

The reviewer hands you a recommended option with reasoning for every judgment-call finding. Your job is to *use* it. The failure mode this skill is calibrated against is over-conservatism — routing a finding to Open questions when the reviewer already did the thinking and the recommendation holds up. Every Open question you file instead of resolving is debt the next round inherits and a human operator has to clear.

So: when the item is a *design-shape or sprint-shape* call — a foundational-decision shift, a schema column change, an API verb choice, a lifecycle invariant, a cross-service contract change, an idempotency-key shape change, a sprint dependency rewire — and **the reviewer recommended an option whose reasoning survives your verification**, default to **Incorporate**. Adopt the recommendation. Only fall back to **Open question** when the choice is genuinely more nuanced than the reviewer's framing (you see a cross-sprint consideration the reviewer missed) or genuinely needs a human operator (balanced trade-offs, missing information) — per the Open-question bucket's signals above. The one shape call that stays in Open question regardless of how well the reviewer reasoned it: a **re-slicing recommendation** — re-slicing the sprint roster is a planning-round move, not a triage move.

Default to **Minor incorporate** when the item is a *wording / consistency / stale-text / missing-cross-link / numbering / progress.md drift* fix. These are not design or sprint-shape decisions; routing them to Open questions just adds noise and pads the next round's triage list.

**Calibration check.** After triaging the whole review, count the buckets across all sources (Executive summary, High-level concerns, Cross-sprint coherence, per-sprint findings, per-step findings, Substance, Minor nits). Two things shape an impl-plan triage: impl-plan reviews surface a high concentration of cross-sprint coherence drift (Prerequisite ↔ Deliverable name drift, dependency-graph one-liner alignment, `progress.md` step-list mismatch), most of which are mechanical Minor incorporates — *and*, because the reviewer now supplies a recommendation for every judgment call and your default is to adopt it, the Incorporate bucket runs heavier and the Open-question bucket runs lighter than under the old "frame, don't resolve" posture. Open questions are now the *exception* (the genuinely-needs-a-human residue), not the default home for shape questions. Use the impl-plan bands below; the design-plan version of this skill uses different bands.

A healthy impl-plan triage is roughly:

- **10–25% Incorporate (full)** — material edits that earn a Decisions-log entry (overview-level for cross-cutting calls; sprint-level for sprint-scoped calls), including adopted reviewer recommendations on shape questions. This bucket runs heavier than under the old posture — that is intended.
- **40–60% Minor incorporate** — cross-sprint coherence fixes plus the usual mechanical fixes (typos, marketing-language purges, missing forward-links, stale wording, numbering, `progress.md` ↔ per-sprint Progress reconciliation). This bucket still runs heaviest for impl plans because the directory-of-files shape produces many seam-drift findings, most of which are mechanical.
- **5–15% Open question** — only the genuinely-needs-a-human residue: shape questions where the reviewer's recommendation didn't survive scrutiny, the trade-offs are truly balanced, required information is missing, or the fix is a re-slicing / promotion-demotion planning move. If this bucket is large, you are probably being too conservative — re-check that you adopted every sound recommendation.
- **0–10% Reviewer-wrong** — a small but non-zero rate is normal, especially on cross-file coherence claims (the highest-error category for reviewer agents). A rate above ~20% is a signal that the review itself should be re-run rather than triaged; surface this to the user.
- **5–15% Ignore** — duplicate / out-of-scope / over-engineering / authoring-guide-contradicting / stylistic / out-of-directory items.

Combined Incorporate + Minor incorporate is typically 50–85%.

Out-of-band signals:

- If **Open question is holding more than ~20% of items**, re-triage the boundary cases — for each Open-question item, ask: *did the reviewer recommend an option, and does its reasoning survive verification?* If yes, it belongs in Incorporate (unless it's a re-slicing / promotion-demotion planning move). Most reviewer-surfaced wording / consistency / cross-sprint-coherence drift belongs in Minor incorporate. Open questions are the residue of genuine human-needed calls — not the default home for findings you'd rather not commit to.
- If **Incorporate (full) is holding more than ~30%**, re-check whether you're making fresh design or sprint-shape calls the reviewer did *not* recommend — adopting a reviewer's recommendation is in scope, inventing your own resolution is not. Also check you're not promoting Minor incorporates.
- If **Minor incorporate is holding more than ~70%**, the review may be flailing — heavy on micro-coherence findings without a load-bearing shape concern. The triage is fine, but flag the pattern in the report — the user may want a different review prompt next time, or to skip the next review entirely if the plan has stabilized.
- If **Reviewer-wrong is holding more than ~20%**, the review's reliability is the bottleneck — stop, report the rate to the user, and let them decide whether to re-run the review with a different reviewer / prompt rather than continue triaging a noisy artifact. Cross-sprint coherence findings are the most likely source of reviewer error in impl-plan reviews; if the high Reviewer-wrong rate is concentrated there, that's worth calling out to the user separately.

## Edit discipline

### Multi-file incorporations (the high-risk path — read this first)

Most cross-sprint-coherence findings are multi-file by nature. They are also the highest-error path in this skill: a Prerequisite renamed in Sprint 04 but left untouched in the consuming Sprint 06 produces a *silently-broken* plan — every file looks internally coherent, only the seam between them is wrong. The seam is invisible to a reviewer reading any single file. Treat multi-file incorporations as their own discipline.

#### Step 1 — List every file the incorporation will touch *before* editing any of them

For each multi-file finding, enumerate the file cluster:

- Which file owns the canonical wording the edit corrects?
- Which files reference (consume, forward-link, mirror, or restate) that wording?
- Does `overview.md` need an update too — module-tree, dependency-graph one-liner, sprint roster, feature-wide locked-decisions table?
- Does `progress.md` need an update — step rename, step add/remove, sprint title change?
- Does the affected sprint's per-sprint Progress section need to mirror a `progress.md` change?

Write the cluster down (in scratch / planning context) before opening any file in the editor. A cluster you didn't enumerate is a cluster you'll forget to finish.

#### Step 2 — Apply edits in dependency order: rename targets before consumers

The canonical-wording change lands first; consumers follow. Specifically:

1. **Rename / re-shape the canonical wording** in the file that owns it (the sprint that ships the Deliverable, the file that holds the locked decision, the module-tree entry that names the new file).
2. **Update each consumer** in turn (Prerequisites lists, Out-of-scope forward-links, dependency-graph one-liners, error-table entries that reference the renamed code, `progress.md` step labels).
3. **Update `overview.md`** if the change crosses a structural surface (module-tree, sprint roster, dependency graph, feature-wide locked decisions).
4. **Update `progress.md` and the per-sprint Progress section together** — never one without the other. Master `progress.md` wins on conflict; reconcile the per-sprint copy to it (unless the master is the broken side, in which case fix the master).

If the order is wrong (consumer edited before target is renamed), you'll generate a window where both old and new wording exist in the directory simultaneously — and any incomplete pass through the cluster leaves the plan in that broken state.

#### Step 3 — After every multi-file incorporation, grep the cluster

Before moving to the next finding, run a literal text search for the *old* canonical wording across the entire directory. Zero hits is the only acceptable result; one or more hits means a consumer was missed. (`grep -rn '<old wording>' <plan-dir>` is the canonical check.) This is the cheapest way to catch silent drift introduced by the incorporation itself.

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
6. Grep the directory for the rejected name. Zero hits expected.

#### Worked example 2: a sprint adds a file the overview's module-tree doesn't predict

Reviewer finding: *"Sprint 03 introduces `src/services/measurement/managers/measurement-resolver.ts` but `overview.md` § Module/Directory layout doesn't list it."*

File cluster:
- `03-manager-core.md` — already names the file in its Deliverables / Steps. No edit needed there.
- `overview.md` § Module/Directory layout — needs the file added in the right tree position.

Order:
1. Add the file to `overview.md`'s tree.
2. Confirm Sprint 03's name for the file matches exactly (canonical wording lives in the sprint that ships it, since the sprint is closer to the actual code).
3. Grep the directory for any other reference to the same module — sometimes a downstream sprint references the file too.

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

A multi-file edit touches more files but does *not* expand the skill's authorization. Edits stay within the named plan directory. If the cluster names a file outside the directory (the source design plan, a sibling doc, code), surface that in the report as an out-of-directory action — do not act on it.

---

### Single-file edit discipline

When you do incorporate (whether single-file or multi-file):

- **Edit the relevant body section directly**, in whichever file owns it. Do not leave both old and new wording side-by-side; purge the obsolete phrasing per the authoring guide's supersession rule.
- **Keep `progress.md` and per-sprint Progress in lockstep.** If a step is renamed, re-ordered, added, or removed, update both surfaces in the same pass. The master `progress.md` wins on conflict; the per-sprint section is reconciled to it (unless the master is the broken one, in which case correct the master).
- **Add a tagged bullet to the relevant Decisions log** for each material incorporation, using the next round number (`(R<n>)`). Cross-cutting calls go in `overview.md`'s log; sprint-scoped calls go in the affected sprint file's log. For pure typo / wording / structural-cleanup edits, a Decisions-log entry is optional — but for anything that names a call the plan now relies on, log it. Lead with a bold phrase that names the call.
- **Update Open questions**: remove an entry if the incorporation resolves it (rare in this skill — usually means you should have triaged it as Open question instead); add an entry to the right scope (overview vs. sprint) for each item triaged into the Open-question bucket.
- **Append exactly one new Status entry** in `overview.md` for this incorporation pass. Format: `**Round <n>**: incorporated review feedback — <one-line summary spanning the most material edits>; <count> items added to Open questions; <count> items declined. _Next:_ <one-line recommended focus for the next planning round, citing Open question IDs + tags + scope where relevant>.` Round number is the plan's next integer at overview level. The `_Next:_` clause persists the recommendation into the doc itself — a user resuming the plan via `trellis-impl-iterate` reads it to recover where to pick up. Typical `_Next:_` patterns: `lock Sprint <NN>`, `resolve overview Q<n> [blocks-v1] before touching Sprint <NN>`, `re-slice Sprints <06>/<07> — seam unstable`, or `hand off to implementation` when no load-bearing items remain. If the incorporation supersedes earlier-round wording in any sprint, call out the supersession explicitly in the same Status line ("R3's helper-in-Sprint-06 wording superseded").
- **Sprint-level Status / Deviations.** If a sprint file has its own Status log or "Feedback incorporated" section, append a tagged entry there too for sprint-scoped incorporations. If a sprint shipped before this review (its final step is `[x]`), prefer the "Feedback incorporated (post-review)" / "Deviations applied during implementation" surfaces over editing the body of an already-shipped sprint — the shipped code is the source of truth, and the doc captures the divergence rather than pretending it didn't happen.
- **Re-read affected files** after editing to catch wording from earlier rounds that your edits silently invalidated. If you find any, fix them in the same pass and note the supersession in the Status entry.
- **Preserve cross-file consistency.** If incorporating one review item creates a tension with another part of the plan you didn't touch (a sibling sprint that no longer agrees, a Prerequisite that no longer matches a renamed Deliverable), either resolve it by extending the edit or file the new tension as an Open question. Do not leave the plan internally contradictory.
- **Honor authoring conventions.** Bold the decision lead in Decisions-log entries. Tag every Decisions-log bullet with `(R<n>)`. Use canonical terms. No marketing words. Identifiers in backticks. Absolute dates only. File paths real, not invented.

When you file an Open question:

- **Pick the right scope.** Cross-sprint / feature-wide → `overview.md`'s Open questions. Sprint-scoped → the affected sprint file's Open questions. When in doubt, prefer the sprint level (lower scope is cheaper to relocate later).
- Place it at the end of the existing Open-questions list, numbered sequentially. Do not renumber existing entries.
- **Make the entry decision-ready.** Because you are surfacing this *instead of* deciding it, the entry must give a human operator everything they need to make the call in one sitting. Include:
  - **The options.** Enumerate the realistic resolutions (carry the reviewer's menu forward, refined by your own analysis — don't just paste it).
  - **Why this needs a human.** State plainly why you didn't adopt the reviewer's recommendation: the consideration the reviewer missed, the missing information, or the fact that the trade-offs are genuinely balanced. Be specific — "needs product input on retention window" beats "judgment call."
  - **Pros and cons per option.** For each option, the concrete upside and the concrete cost — enough that the human can see *why the options are roughly equal* and what tips the balance. If one option is mildly preferable, say so and say what would have to be true for the other to win.
- Also include: the question (one sentence), whether it blocks anything currently in scope, and the sprint(s) it touches.
- Cross-link to the originating review item by quoting a short phrase from the review (in italics) so the human can trace it back.

When you ignore:

- Do not edit the plan. Track the item only in your final chat report.

## Authorization scope for this skill

This skill edits an implementation plan in place — across `overview.md`, every sprint file, `progress.md`, and (when present) `post-mortem.md` within the named directory. That is the explicit purpose, so you do not need additional confirmation for body edits, Decisions-log additions, Open-questions additions, Status appends, or `progress.md` reconciliation *within the named plan directory*. You **do** need confirmation before:

- Editing any file outside the named plan directory (including the source design plan, sibling docs, or project-level conventions files).
- Renaming files within the directory (sprint file renames, overview-file renames, etc. — propose, don't act).
- Re-slicing the sprint roster (splitting a file, folding two files together, re-numbering). These are planning-round moves; surface them in the report or as Open questions instead.
- Touching code, schemas, migrations, or anything outside the plan directory.
- Creating `post-mortem.md` if no sprint has shipped yet (the file appears the first time a sprint ships, not at triage time). If a review item asks for a `post-mortem.md` entry but no sprint has shipped, decline it and explain.

If a review item recommends changes outside the plan directory, treat those as Open questions or surface them in the final chat report; do not act on them.

## Workflow

1. **Read every file in the plan directory, end-to-end.** No skimming. `overview.md` first; then `progress.md`; then each sprint file in numeric order; then `post-mortem.md` if present. Note the plan's current Round count in `overview.md`'s Status log, the latest Decisions-log round tags (overview-level + per sprint), the current Open-questions numbering at each scope, and which sprints have shipped per `progress.md`.
2. **Read the review document end-to-end.** Note its overall structure and which sections you'll process.
3. **Enumerate the review items.** Most reviews are already itemized (numbered concerns / nits). Treat each numbered item as one unit. If a review item bundles multiple sub-points, split it into sub-units before triaging — each sub-unit gets its own bucket. Per-sprint and per-step findings are each their own units.
4. **First-pass triage** each unit into incorporate / minor-incorporate / open-question / ignore. Keep a running internal table: `unit-id, source section, target file(s), bucket, one-line rationale`. Note when an incorporation will require multi-file edits. Reviewer-wrong is a *result* of verification (Step 5), not a first-pass call.
5. **Verify every Incorporate-bucket item, plus every cross-file Minor-incorporate item.** For each, confirm the cited section / cross-file claim / cross-reference / convention claim against the actual source per § "Verify before incorporating." For cross-sprint claims (the most error-prone category), open both files. Items whose cited evidence doesn't hold move to **Reviewer-wrong** — do not edit the plan based on a claim that didn't survive verification. Single-file Minor incorporate, Open question, and Ignore items skip verification (see § "When verification is required").
6. **Calibration check.** Count the buckets against the impl-plan bands in § "The triage rule" → "Calibration check" (target ~10–25% Incorporate, ~40–60% Minor, ~5–15% Open question, 0–10% Reviewer-wrong, ~5–15% Ignore). If Open question holds >20%, re-triage boundary cases — for each, check whether the reviewer recommended an option whose reasoning survives verification; if so, it belongs in Incorporate (unless it's a re-slicing / promotion-demotion planning move). If Incorporate (full) holds >30%, re-check for fresh design or sprint-shape calls the reviewer did not recommend. If Reviewer-wrong holds >20%, **stop and surface this to the user** — the review may need to be re-run rather than triaged.
7. **Group incorporations by file** so you can apply edits to each file in one pass instead of opening it repeatedly. Apply Incorporate and Minor incorporate items in the same pass; the only difference at edit time is whether a Decisions-log entry is added afterward. Apply incorporations in dependency order: structural moves (renaming a referenced helper, fixing a `progress.md` ↔ sprint mismatch) before within-section edits that depend on them. **For any multi-file incorporation, follow § "Multi-file incorporations (the high-risk path)" — list the cluster, edit canonical-wording first, consumers second, then grep the cluster for the rejected wording.** Make all edits via the `Edit` tool against the relevant file; do not rewrite a whole file unless the volume of edits genuinely warrants it.
8. **Apply Open-question additions** as a single batch at the end of each affected Open-questions list (overview-level batch first, then per-sprint batches). Preserve existing numbering — append-only.
9. **Update Decisions logs** with one bullet per *material* incorporation (Incorporate bucket only; Minor incorporates do not earn entries), all tagged with the same new round number for overview-level entries. Sprint-level entries can use the same round number even though the sprint file's prior round count may differ — what matters is that *this incorporation pass* is one coherent round.
10. **Update `progress.md`** if any step structure changed. Reconcile each affected sprint's per-sprint Progress section with the master.
11. **Append the overview Status entry** as the last edit. If any sprint had its own Status / Feedback-incorporated section updated, those are already in place from step 7.
12. **Re-read the directory** end-to-end one more time, looking for: stale wording your edits orphaned across files; Decisions-log bullets that contradict body edits in a sibling file; Open-questions entries your edits silently resolved; numbering inconsistencies; cross-sprint references that broke (a Prerequisite that names a Deliverable you renamed; a forward-link to a sprint whose section you re-titled). Fix what you find inside the same round.
13. **Report back to the user** in chat (format below).

## Report format

Output the bucket-distribution summary first, then five sections (Incorporated, Added to Open questions, Reviewer-wrong, Declined, Out-of-directory actions surfaced), in this order, in chat. Keep each one terse — bullets, not paragraphs. Group bullets by file or by sprint when the volume warrants it.

**Bucket distribution**: `<I> material / <m> minor / <O> open-question / <W> reviewer-wrong / <X> ignore` (totals to the review's item count).

### Incorporated

Two sub-lists:

- **Material incorporations** (with Decisions-log entries) — each bullet: the review concern (short reference, not a quote dump) + the file(s) and section(s) that changed + the Decisions-log entry's bold lead and which log (overview vs. which sprint). If a multi-file incorporation, name all touched files. Group by sprint or by file when there are more than ~6 entries.
- **Minor incorporations** (no Decisions-log entry) — each bullet: the review concern + the file(s) and section(s) that changed. Group these tightly; if there are many, summarize by category ("9 cross-sprint coherence fixes — Prerequisite ↔ Deliverable naming alignment across Sprints 03/04/06; 4 marketing-language purges in `overview.md` § Philosophy") rather than listing each individually. Cross-sprint coherence Minor incorporations should be grouped by the seam they crossed, not by file.

### Added to Open questions

A bulleted summary of items filed for the next round — these should be the exception, not the bulk of the triage. Each bullet: the question (one short phrase) + the scope where it was filed (overview-level / Sprint NN) + why it needed a human rather than the reviewer's recommendation (the consideration the reviewer missed, the missing information, the genuinely-balanced trade-off, or that it's a re-slicing / promotion-demotion planning move). If this section is long, double-check you didn't punt recommendations you could have adopted.

### Reviewer-wrong (verification failures)

A bulleted summary of review items whose cited evidence didn't hold. Each bullet: the review item (short reference) + the file(s) the reviewer cited + what the reviewer claimed + what the source actually says (one sentence; quote when the wording is the load-bearing part) + the failure mode (`misread-section` / `bad-cross-file-claim` / `bad-cross-reference` / `wrong-convention-claim` / `wrong-count` / `stale-wording-already-purged` / `non-contradiction`).

If this section has more than ~3 items or represents >20% of the review, lead the section with a one-line summary line so the user notices: **"Reviewer reliability concern: <N> of <total> items did not survive verification — consider re-running the review."** When the failures are concentrated in cross-sprint coherence findings, call that out specifically.

If the section is empty, omit it.

### Declined

A bulleted summary of review items you ignored (excluding Reviewer-wrong, which has its own section). Each bullet: the review item (short reference) + a one-sentence reason. Be honest — if you declined multiple items for the same reason, group them.

### Out-of-directory actions surfaced (optional)

Only if the review recommended changes outside the plan directory (edits to the design plan, code, configs, conventions files, sprint-roster re-slicing, file renames). Each bullet: the action + which file / decision it affects + that you did **not** perform it. Omit this section entirely if there are none.

End with one sentence naming the new overview Round number you wrote and (if any) cross-file or cross-directory actions you flagged but did not perform.

## Tone

- Direct and concrete; no hedging. The reviewer was opinionated; your triage should be too.
- Do not flatter the review or the plan. State what you did.
- Quote the plan and review with backticks/italics when disambiguating; otherwise paraphrase.
- Identifiers in backticks. Absolute dates. Canonical terms. Real file paths in the report (`Sprint 04 (`./04-manager-core.md`)`, not "the manager sprint").

## Anti-patterns specific to this skill

- **Don't silently resolve open questions.** If a review item proposes an answer to an existing Open question (overview or sprint-level), that is a *planning round* move, not an *incorporation* move. File it as a refinement to the existing Open question or surface it in the report — do not delete the Open-questions entry yourself.
- **Don't promote the round number twice.** Every incorporation in this pass shares one overview-level round tag. The plan gets exactly one new overview Status entry.
- **Don't fabricate Decisions-log entries** for review items you triaged as Open questions. The Decisions log is for resolved calls, not for "we noted this and will think about it."
- **Don't abuse Minor incorporate to land load-bearing edits without a Decisions-log entry.** Minor incorporate exists for typos / wording / consistency / stale-text / numbering / missing-cross-link / `progress.md`-drift fixes. If your edit names a call the plan now relies on (a sprint-level locked decision, an idempotency-key shape, an API error code, a contract clause, a sprint dependency), it's a full Incorporate — add the Decisions-log entry to the appropriate log (overview-level for cross-cutting; sprint-level for sprint-scoped). The audit trail breaks if material calls hide in the Minor bucket.
- **Don't write build-spec specifics** even if a review item asks for them (file paths the author didn't propose, full TypeScript signatures, exact function names beyond illustrative). Sketches only.
- **Don't reorder Open-questions lists** to "improve flow." Append-only — preserve the existing numbering so prior conversation references stay valid.
- **Don't re-slice the sprint roster** as part of triage. Splitting / folding / re-numbering sprints is a planning-round move; surface it in the report or as an Open question.
- **Don't edit a shipped sprint's body** as if the code didn't exist. If a sprint's final step is `[x]` in `progress.md`, the implementation is the source of truth — capture the review's correction in "Feedback incorporated (post-review)" or "Deviations applied during implementation" instead of pretending the original plan said the new thing all along.
- **Don't pre-create `post-mortem.md`.** If the review asks for an entry but no sprint has shipped, decline the item.
- **Don't drift `progress.md` and per-sprint Progress.** If you edit one, edit the other in the same pass.
- **Don't edit the design plan** even when a review item routes back to a design-level call. Surface it in the report.
- **Don't skip the final re-read.** It is the cheapest way to catch the cross-file inconsistencies your own edits introduced.
- **Don't skip verification on Incorporate-bucket items, or on cross-file Minor-incorporate items.** A confident-but-wrong reviewer becomes a corrupted plan if their claims are acted on without checking the source. Cross-sprint coherence findings are the highest-error category in impl-plan reviews — verify them with extra care. Verification is fast (median 30–60 seconds per item) and asymmetrically valuable.
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
