---
name: trellis-design-iterate
description: Iterate on a Design Plan
argument-hint: <path-to-plan-file>
disable-model-invocation: true
---

# Design Plan — Iterate Skill

Skill prompt for an LLM agent. Activated when a user wants to drive an existing **design plan** through one more planning round (Round 2 onward).

This skill is the most-invoked skill in the design-plan flow. The round mechanics below are **load-bearing** and easy to drop under load — every rule in this prompt has cost the system a round when missed. Read the prompt end-to-end every invocation; do not rely on memory of prior rounds.

---

## Required inputs

The path to the design plan to iterate on is: $0

If `$0` is empty, tell the user they invoked the skill incorrectly (missing path argument) and stop.

Before doing anything, in this order:

1. Read the [Design Plan Documents — Authoring Guide](../trellis-design-create/design-plan.md) end to end. Do not skim.
2. Read the existing design plan at `$0` end to end — every section, top to bottom. Don't rely on prior conversation context.
3. Skim the project's `CLAUDE.md` (or equivalent) for any project conventions, banned patterns, or authorization rules that may affect this round's decisions.

If the user provided no input beyond the skill invocation, stop here and say: *"Ready for Round &lt;next&gt;. Tell me which Open questions to focus on, or share the input you'd like incorporated."* Do not pick questions on your own.

If the user provided input with the invocation, treat that as the round's seed and proceed with the workflow below.

---

## The round workflow

Every round follows the same shape. Skipping any step has cost the system a round at some point — they are all load-bearing.

### Step 1 — Take inventory

Before saying anything substantive, internally note:

- The plan's current Round number (next entry in Status log will be `Round N+1`).
- The current Open questions — count, and the breakdown by tag (`[blocks-v1]` / `[blocks-impl]` / `[deferred]` / `[exploratory]`).
- The most recent Decisions log entries (last 1–2 rounds).
- Any wording the most recent Status entry called out as superseded.
- Any analogue plans or cross-references the plan cites that you may need to spot-check this round.

### Step 2 — Triage which questions this round will address

Pick **1 to ~5** questions that can plausibly be resolved in this round without spawning many more. Prefer questions whose resolutions are reasonably independent of each other (so they don't snowball). Bias toward `[blocks-v1]` and `[blocks-impl]` entries before `[deferred]` / `[exploratory]`.

If the user named specific questions, those are the round's focus. Don't expand beyond what they asked for unless you flag the expansion explicitly.

### Step 3 — Frame each choice with alternatives

For each question this round will address:

- Surface **2–3 viable options** with their trade-offs.
- Recommend one — be opinionated; don't hedge.
- Name what gets unblocked if the question resolves this way.

Then **wait for the user**. Do not auto-resolve. Even when you have a strong opinion, the user picks. The agent's job is to frame the call sharply, not to make it.

If the user pushes back, integrate their pushback and re-frame. If the user says "you pick," capture each call as a `(R<n>)` Decisions log entry tagged with a one-line rationale, and surface the unilateral decisions in the round hand-off so the user can react.

### Step 4 — Update the doc for each resolution

For every question the user resolved this round, in the same edit pass:

1. **Rewrite the affected body section** to reflect the chosen design. Purge the obsolete wording — do not leave both old and new versions side-by-side. Supersession discipline is described in [design-plan.md § "Supersession"](../trellis-design-create/design-plan.md).
2. **Add a tagged bullet to the Decisions log** with the next round number, e.g. `(R<n>)`. Each bullet leads with a **bold phrase** that names the call.
3. **Remove the resolved entry from Open questions.** Do not append `(resolved)` in place — the entry **moves** to Decisions log. Do not renumber remaining entries.
4. **Add any newly-surfaced sub-questions** to Open questions for the next round. Tag each at creation with exactly one of `[blocks-v1]` / `[blocks-impl]` / `[deferred]` / `[exploratory]` per [design-plan.md § "Severity tag taxonomy"](../trellis-design-create/design-plan.md). For `[blocks-v1]` / `[blocks-impl]` entries, name the specific blockee.

### Step 5 — Re-read affected sections

Rewriting one section can silently invalidate wording elsewhere. Re-read every section the round's resolutions touched (and the sections those sections cross-reference) and fix any stale wording in the **same pass**. Do not leave the plan internally contradictory.

If a resolution invalidated a prior `(R<m>)` Decisions-log entry, note the supersession explicitly in the round's Status entry (Step 7).

### Step 6 — Sanity-check before stopping

Walk these checks before sending the hand-off message. Each one has cost the system a round when missed.

- **Every new Decisions-log entry has rationale.** A bold lead with no paragraph following it is half a decision.
- **Every new Open question carries exactly one severity tag.** Untagged entries are a process bug.
- **Every cross-reference resolves.** Each link points at a real adjacent doc / wiki page / code path. Mark broken or unauthored references explicitly (`(TBD — not yet authored)`) rather than letting them rot silently.
- **No marketing words.** Walk for `elegant`, `clean`, `robust`, `powerful`, `best-in-class`. Purge.
- **No build-spec specifics.** No exact file paths beyond illustrative ones, no exact TypeScript implementations, no specific function names beyond illustrative.
- **No `(resolved)` tags** inside Open questions. Resolved entries move to Decisions log.
- **No fabricated decisions.** Anything you decided unilaterally (because the user said "you pick") is logged and called out in the hand-off so the user can react.
- **v1 / Tier-0 compromises are labeled in place** — `Tier 0 trade-off`, `v1 ships with X — semantics improve once Y lands`, `acceptable at this scale; revisit if Z becomes hot`. A future reader should never have to guess whether a piece is the long-term answer or a deliberate stop-gap.
- **Tone matches the guide** — active voice about the system; identifiers in backticks; em-dashes for asides; absolute dates only.

Fix what you find here, in this round.

### Step 7 — Append exactly one Status entry

Format (two clauses — what changed, then a `_Next:_` clause naming the recommended next-round focus):

```
- **Round <n>**: <one-line summary of what this round resolved>; <one phrase about what was superseded, if anything>. _Next:_ <one-line recommended focus for the next round, citing Open question IDs + tags>.
```

Examples:

```
- **Round 5**: cross-conversation sync redesigned — replaced GET /chat/sync with POST /chat/conversations/sync. R3's contiguity-cache rule superseded. _Next:_ lock the privacy predicate for participant invites (Q12 [blocks-v1]).
- **Round 8**: schema for `chat.reactions` finalized; soft-delete cascade documented. _Next:_ graduate to implementation plan — no [blocks-v1] / [blocks-impl] entries remain.
```

Why the `_Next:_` clause: it persists the chat-only hand-off recommendation into the doc itself. A user resuming the plan a week later via `trellis-design-iterate` reads the latest `_Next:_` and knows where to pick up — without depending on prior chat history.

Pick the `_Next:_` clause from your completeness assessment's "Recommended next-round focus" (Step 8) — they should agree. If the plan is complete, the `_Next:_` clause is `graduate to implementation plan` (or `run trellis-design-review for an external check first` if the user asked for one).

Cite supersessions explicitly when they happened (e.g., `R7 supersedes R5's contiguity-cache rule`). Status entries are append-only — never edit prior entries.

### Step 8 — Emit the completeness assessment to chat

This is **mandatory** at the end of every round, including small ones. The assessment is a chat-only output (not part of the doc body). It forces the agent to take a position each round, which surfaces drift before it compounds.

Use this exact structure:

```markdown
### Round <N> — completeness assessment

**Verdict**: <not-yet-complete | substantially-complete | complete>
**Open-question tag counts**: `<a> blocks-v1 / <b> blocks-impl / <c> deferred / <d> exploratory`

**What's still load-bearing-open** *(if not complete; omit otherwise)*:
- <one-line item naming the section / question + tag + why it's load-bearing>
- …

**Top nits worth resolving before declaring done** *(if substantially-complete or complete; omit otherwise)*:
- <one-line item naming a section + the wording / consistency / coverage issue>
- …

**Recommended next-round focus**:
- <one-line recommendation; usually the highest-priority blocks-v1 / blocks-impl items, or "graduate to implementation plan" if complete>
```

Verdict semantics (full definitions in [design-plan.md § "Round-end completeness assessment"](../trellis-design-create/design-plan.md)):

- **`not-yet-complete`** — at least one load-bearing item is still open or undecided. Default to this verdict whenever any `[blocks-v1]` or `[blocks-impl]` entry remains open. The "What's still load-bearing-open" block enumerates the gaps.
- **`substantially-complete`** — all load-bearing items are decided, but rough edges remain (stale wording, marketing language to purge, a missing tag, a Decisions log entry that drops a clause). The "Top nits" block enumerates them.
- **`complete`** — load-bearing decisions are made *and* the doc is internally clean. Recommend graduating to an implementation plan (or running `trellis-design-review` first if the user wants an external check).

Calibration:

- **Default to `not-yet-complete`** until *every* item from the "When a design plan is 'complete'" checklist in `design-plan.md` is true. The bar for `substantially-complete` is high — if you haven't ticked the foundational + scope + schema + lifecycle + API + auth checks mentally, the verdict is `not-yet-complete`.
- **Don't oscillate.** If round N said `substantially-complete` and round N+1 only changed a comment, the verdict shouldn't drop back to `not-yet-complete`.
- **Don't pad either side.** A 3-bullet "load-bearing-open" list is a real list. A 12-bullet "top nits" list is a sign the plan is not actually substantially-complete — promote the most material items into "load-bearing-open" and re-grade.
- **Cite the section** in every bullet (`Schema § \`chat.messages\``, `Open questions Q4`) so the user can navigate without re-reading the doc.

### Step 9 — Stop. Wait for the user

Don't auto-progress to the next round. Each round is a discrete user-driven step.

---

## Posture rules (read every round)

- **Surface alternatives, don't decide unilaterally.** When the user has skin in the game, the agent's job is to frame the call sharply. Auto-resolution silently hard-codes choices the user might have made differently.
- **Don't fabricate.** If the user didn't decide it, it's an Open question, not a foundational decision — even if you have a confident default.
- **Don't speculate beyond scope.** "This will probably also be useful for X" is a comment for Open Questions or the Deferred section, not load-bearing prose in the body.
- **Don't pre-build extensibility hooks for hypotheticals.** Reserved columns / enum values / fields whose only justification is "for future X" without concrete demand are noise.
- **Active voice about the system.** "The worker scans," "the manager partitions" — not "the system performs reconciliation."
- **Bold the decision lead.** Every Decisions-log bullet, every foundational-decision item, leads with a bold phrase that names the call.
- **Promote `[exploratory]` → `[blocks-impl]` / `[blocks-v1]`** when a question crystallizes; **demote `[blocks-v1]` → `[deferred]`** when the round downgrades it. Note the promotion / demotion in the Status entry.
- **What to do when the user doesn't know.** Surface analogue resolutions from similar systems / prior plans. Default to v1 simplicity — the smallest answer that doesn't paint into a corner. Document the punt: "Deferred — revisit when concrete demand emerges" is a legitimate resolution.

---

## Anti-patterns specific to iteration

- **Don't skip the completeness assessment.** It is mandatory every round, including small ones. Without it, the user has no honest read on whether the plan is converging.
- **Don't skip the Status entry** when the round was small. Every round earns one entry.
- **Don't skip the sanity-check pass.** It is the cheapest way to catch the inconsistencies your own edits introduced.
- **Don't append `(resolved)` tags** inside Open questions. Move the entry to Decisions log.
- **Don't auto-progress to the next round.** Hand back to the user after the assessment.
- **Don't renumber Open questions.** Append-only — preserve numbering so prior conversation references stay valid.
- **Don't write a build spec.** Design plans describe what the system is, not where each line of code lives.

---

## Additional Instructions

Before executing this skill, read and apply Trellis instructions from these sources, in ascending order of precedence:

1. `~/.trellis/instructions.md`
2. `<repo-root>/.trellis/instructions.md`
3. Additional instructions included with this skill invocation

`<repo-root>` is the root of the current Git repository. Missing instruction files are normal; skip them silently.

If instructions conflict, later sources override earlier sources. Invocation-specific instructions apply only to the current run and have the highest precedence.

These instructions may override this skill document, but they must not override system, developer, tool, safety, or repository policy instructions.

