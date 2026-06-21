---
name: design-create
description: Create a new Design Plan
argument-hint: <root>
disable-model-invocation: true
---

# Design Plan — Bootstrap Skill

Skill prompt for an LLM agent. Activated when the user wants to start a new **design plan** from scratch — i.e., Round 1 of the design plan's iteration cycle.

This skill stops at the end of Round 1. Subsequent rounds (resolving open questions, fleshing out schema / API surface / lifecycle / cross-service contracts) follow the round-by-round mechanics in [design-plan.md](../specs/design-plan.md). After Round 1 lands, hand the conversation back to the user for Round 2 via the `design-iterate` skill.

---

## What this skill produces

A new markdown file at `<root>/design.md`, populated with the Round 1 skeleton of a design plan:

- **Title & framing block** — what the doc covers, what status it's at, a "not a build spec yet" disclaimer.
- **Canonical-term clarification** — when the domain has terminology drift between casual speech and the code.
- **Cross-references** — adjacent plans / wiki pages / code paths the reader should know about, annotated with what each contributes.
- **Foundational decisions** — the load-bearing assumptions, captured as numbered items (Shape A) or as Purpose / Scope prose (Shape B), with rationale paragraphs.
- **Scope and Non-Goals** — explicit in-scope and out-of-scope-for-v1 lists.
- **Open questions** — initialized with everything not yet resolved in Round 1. This is normally the longest section in a Round 1 doc.
- **Decisions log** — `(R1)`-tagged entries for anything locked in this round (typically the foundational decisions).
- **Status / Rounds** — one entry: `**Round 1**: foundational decisions captured + open questions enumerated.`

The Round 1 doc is **not** complete. Schema, API surface, lifecycle / sync / GC, cross-service contracts, privacy / authorization, UX surface — these mostly come in later rounds. Don't pre-populate them with assumptions; surface the unresolved shape as open questions.

---

## Required inputs

The user supplies a single root directory as `<root>` (absolute or repo-relative). The agent always writes the design plan to the canonical file `<root>/design.md`. Take this root from the user's natural-language invocation; ask if it's missing. If the user instead gave a path ending in `design.md`, treat its parent directory as `<root>` — do not re-prompt.

Before starting, the agent must have:

1. **The root directory `<root>`.** Create it if it doesn't exist. The output file is always `<root>/design.md`. If a file already exists at that path, warn before overwriting.
2. **The user's initial briefing.** The high-level shape of what they want to design — the domain, the canonical terminology if any drifts, the rough scope, the load-bearing assumptions they already have. This may be supplied along with the skill invocation, or arrive after the agent prompts for it (see Step A.1).
3. **The design-plan authoring guide.** Read [design-plan.md](../specs/design-plan.md) end-to-end before producing files — the document anatomy, section ordering, tone, and supersession rules in that guide are mandatory.
4. **Project context.** Skim the project's `CLAUDE.md`, `AGENTS.md` (or equivalent) for terminology, conventions, the project's typical service / system shape, banned patterns, and authorization rules (e.g., schema-edit gating). The cross-references list and the foundational decisions should reflect actual project conventions, not generic ones.
5. **(If they exist) one or more analogue design plans in the repo or adjacent repos.** Borrowing structure from a precedent is encouraged — naming conventions, foundational-decision shape (numbered list vs. Purpose/Scope prose), section ordering. Cite the analogue in your Round 1 Status entry so the user can see the lineage.

If any of these is missing, **stop and ask the user** before producing files. Don't guess at the user's intent.

**Architecture is inherited, not prescribed.** Design plans serve any kind of system — backend service, frontend app, CLI, mobile client, library, infrastructure module, data pipeline. The structural mechanics (foundational decisions, scope frame, schema-or-equivalent, lifecycle, API-or-equivalent, cross-component contracts, open questions) hold in every case. The *content* of those structures must reflect the actual system being designed. Whenever the authoring guide cites a backend-service example (schemas, repositories, managers, signed URLs, transactions), translate the example to its equivalent in the target system. If there's no equivalent, drop the section.

**Additional Context**: The user may include additional instructions with the skill invocation. Those should be integrated into these instructions, and take precedence.

---

## The Round 1 workflow

### Step A — Read everything, then sketch foundations

1. **Confirm there's a briefing.** If the user invoked the skill with no detail beyond the path, stop and ask: *"Ready for Round 1. Tell me about the system you want to design — what is it, who uses it, what's the rough scope, and what foundational decisions (if any) you've already made."* Do not proceed without a briefing.
2. **Read the design-plan authoring guide** at [design-plan.md](../specs/design-plan.md) end to end. Pay particular attention to "What these documents are," "Round 1 — scaffolding," the document anatomy section, and the anti-patterns list.
3. **Read the project's `CLAUDE.md`** and any analogue design plans the user pointed to (or that you find adjacent to where the new plan will live). Extract:
   - Terminology the project uses (canonical terms, casing conventions, naming styles).
   - The architecture / layering the project follows (services, modules, atomic-design layers, CLI structure, infra modules — whichever applies).
   - Banned patterns and known footguns surfaced by prior plans, code review threads, or post-mortems.
   - Authorization rules the design plan must respect (e.g., schema-edit gating to specific GitHub users).
4. **Sketch the foundational decisions privately** (in scratch / planning context) from the user's briefing + the project context. Identify:
   - The 5–8 highest-leverage shape questions the design must answer at the foundational level (scope, ownership, primitives, lifecycle shape, reach into other services).
   - The canonical term to settle on, if there's terminology drift.
   - The cross-references the reader will need (adjacent plans, wiki pages, code paths) — annotated with what each contributes.
   - A first cut at scope (what's in, what's out for v1) — be aggressive about what's out.
   - The shape choice for foundational decisions: numbered list (Shape A — multi-faceted designs with rationale at the foundational level) vs. Purpose/Scope prose (Shape B — self-contained product surface). Pick the one that fits the work.

### Step B — Confirm framing and questions with the user before writing files

Don't write the file yet. Surface to the user, in chat:

```
Proposed framing for <plan title>:
- Title: <title>
- Canonical term: `<term>` — replaces casual <synonyms>
- Shape choice for foundational decisions: <Shape A — numbered | Shape B — Purpose/Scope prose>
- Scope frame:
  - In: <bulleted first cut>
  - Out (for v1): <bulleted first cut>
- Cross-references: <bulleted list, annotated>

Highest-leverage shape questions I need answered to capture foundational decisions:
1. <question> — <why it's load-bearing; what it unblocks>
2. <question> — …
…
N. <question> — …

Analogue plans I'll borrow structure from: <none | path-to-plan + what's being borrowed>

Anything to add, remove, or reshape before I draft Round 1?
```

**Iterate on framing in conversation before writing the file.** Wrong framing means re-shaping the doc in Round 2; getting it close in Round 1 saves churn. If the user pushes back on the canonical term, the scope, or any of the questions, integrate and re-surface before proceeding. The user's answers to the foundational-question list become the foundational decisions in the doc.

If the user signals they want to skip this step (`"just draft it"`), proceed — but capture the framing decisions you made unilaterally as `(R1)` entries in the Decisions log so the user can react after they read the file.

### Step C — Draft the Round 1 file

Once framing is agreed, write the file at `<root>/design.md`. Populate the Round 1 sections per the authoring guide, scaled to what's actually known after Step B:

- **Title & framing block** — link out to any upstream context (parent initiative, related plans). Default disclaimer: `Living planning artifact … not a build spec yet.`
- **Canonical-term clarification** — if Step B settled one.
- **Cross-references** — the annotated list from Step B.
- **Foundational decisions** — Shape A or Shape B per the choice from Step B. Each item leads with a bold sentence stating the decision, then a rationale paragraph (or several). Numbered if Shape A. Pull rationale from the user's answers in Step B; do not invent.
- **Scope and Non-Goals** — the explicit in-scope and out-of-scope-for-v1 lists.
- **Service boundaries / data ownership** (when applicable) — when the work crosses service lines, name which service owns which tables and which crosses are allowed.
- **Open questions** — numbered list of everything not resolved in Round 1. Each entry **must** carry a severity tag at creation: `[blocks-v1]`, `[blocks-impl]`, `[deferred]`, or `[exploratory]` (see [design-plan.md § "Open questions" → "Severity tag taxonomy"](../specs/design-plan.md)). Each entry also names: the question briefly; why it's open / what makes it hard; an indicative resolution direction or an explicit "deferred until X"; the named blockee for `[blocks-v1]` / `[blocks-impl]` entries. Round 1 lists skew heavy on `[blocks-v1]` and `[blocks-impl]`; an `[exploratory]`-heavy Round 1 list is a signal that you're capturing background discussion rather than load-bearing gaps. **This section is normally the longest section in a Round 1 doc.**
- **Deferred (out of scope for this plan)** — distinct from "Out of scope (v1)" — the longer list of things the design *could* support but isn't doing. One bullet with a brief reason each.
- **Decisions log** — `(R1)`-tagged entries for everything locked in this round. Each leads with a bold phrase that names the call.
- **Status** — single entry: `**Round 1**: foundational decisions captured + open questions enumerated. <One sentence on what's most load-bearing-open.> _Next:_ <one-line recommended Round 2 focus — usually 2–3 of the highest-priority [blocks-v1] questions, picked because their resolutions are reasonably independent of each other; cite the question IDs + tags>.` Cite the analogue plan(s) you borrowed structure from, if any. The `_Next:_` clause persists the chat hand-off recommendation into the doc itself so a user resuming via `design-iterate` recovers it without depending on chat history.

Do **not** populate in Round 1 (leave the section out, or include the heading with a placeholder line `_To be drafted in a later round._`):

- **Schema / Data model** — leave out unless the user explicitly locked specific tables / columns / indexes in Step B.
- **Lifecycle / operational sections** — leave out.
- **API surface** — leave out.
- **Cross-service contract** — leave out unless the user explicitly locked the contract in Step B.
- **Privacy / Authorization / Safety** — leave out unless the user explicitly locked the access rules in Step B.
- **UX / Surface** — leave out.
- **Sequencing suggestion** — leave out; this is more useful once the design is converging.

A Round 1 doc is typically 80–250 lines. If it's approaching the length of a finished design plan, you've over-decided — pull back, file the excess as Open questions.

### Step C.5 — Sanity-check before hand-off

Before sending the user the hand-off message, walk the file you just produced against the shared [design-plan.md § "Pre-hand-off sanity checklist"](../specs/design-plan.md) — every decision has rationale, every Open question carries exactly one severity tag, cross-references resolve, no marketing words, no build-spec specifics, no `(resolved)` tags, v1/Tier-0 compromises labeled, tone matches the guide. Plus two Round-1-specific checks:

- **Every Open question is concrete.** "What about caching?" is not a question; "Should reactions invalidate the conversation-sync cache, or is the existing per-message cursor sufficient for v1?" is. Reshape vague questions or split them.
- **Scope frame and Open questions don't contradict.** An Open question that asks "should X be in scope" but the Scope section silently includes X is incoherent — pick a side or note the contradiction explicitly.

Fix the issues you find before handing back to the user. A Round 1 file with broken cross-references or unjustified decisions is harder to repair in Round 2 than to get right now.

### Step D — Hand back to the user

After the file is written, emit the hand-off message (template below). The hand-off is a chat-only summary; it is not part of the file.

The hand-off ends with the **completeness assessment** mandated by [design-plan.md § "Round-end completeness assessment"](../specs/design-plan.md) — emit the block exactly as that section specifies (the worked Round 1 instance in the hand-off template below follows it). Round 1 verdicts are almost always `not-yet-complete` — the doc is by design incomplete after Round 1. The interesting content is *what* is load-bearing-open and *which* of those should be Round 2's focus.

**Stop. Don't auto-progress to Round 2.**

---

## Quality bar for Round 1

When the file is written, the user should be able to:

- Read the doc end to end in ≤ 10 minutes and have a clear mental model of what's being designed and why.
- See the foundational decisions and recognize them as decisions they actually made (or, if Step B was skipped, recognize them as decisions captured in `(R1)` Decisions-log entries that they can react to).
- Read the Open Questions list and recognize them as the real load-bearing decisions left to make — not as a TODO list of trivia.
- Skim the doc and see that nothing was silently invented (no schema columns the user didn't ask for; no API endpoints the user didn't sketch; no "the system will probably also do X" speculation).

What's explicitly *not* expected after Round 1:

- That the schema is locked (unless the user explicitly locked it in Step B).
- That the API surface is sketched (unless the user explicitly sketched it in Step B).
- That lifecycle, GC, sync, or worker behavior is decided.
- That the Open Questions list won't grow in Round 2 — surfacing more questions as the design clarifies is a feature, not a bug.

---

## Anti-patterns specific to bootstrap

- **Don't fabricate foundational decisions.** If the user didn't decide it in their briefing or Step B, it's an Open question, not a foundational decision. The agent's Round 1 job is to surface, not to decide.
- **Don't pre-populate schema, API surface, or lifecycle sections** with assumptions the user didn't sign off on. A Round 1 schema sketch with five tables you invented is worse than no schema sketch — it anchors the design on guesswork.
- **Don't copy the user's briefing verbatim into the doc.** Translate it into the canonical anatomy. A briefing paragraph that lands as-is in the foundational decisions section signals the agent skipped the structural work.
- **Don't merge multiple foundational decisions into one bullet.** If the user said "we need X and Y and Z," that's three bullets, three rationales, three potentially-revisable calls. Bundling hides decision-points.
- **Don't skip Step B.** Writing the file before confirming framing means re-shaping it the moment the user pushes back. The 5-minute confirmation is cheap insurance.
- **Don't write a build spec.** No file paths beyond illustrative; no exact function signatures; no specific module structures. The design plan describes what the system is, not where each line of code lives.
- **Don't propose a design "pattern" without surfacing alternatives** when the shape has competing reasonable answers. If you recommend a polymorphic-parent table over subclass tables, the rationale must name the rejected alternative; otherwise the call hides behind a choice that wasn't surfaced.
- **Don't auto-resolve the user's open questions.** Even when you have a strong opinion, surface alternatives in Step B and let the user pick.
- **Don't pre-build extensibility hooks for hypotheticals.** "Reserved for future X" columns / fields / sections are fine when the user asked for them; they're noise when the agent invents them.
- **Don't auto-progress to Round 2.** Hand back to the user after the assessment. Subsequent rounds are driven by the user, via `design-iterate`.
- **Don't pad the Decisions log with framing decisions you made unilaterally** unless Step B was skipped. If the user signed off on framing in Step B, the foundational decisions are theirs; one `(R1)` entry per foundational decision is appropriate, not extra ones for "decisions" the user already made.

---

## Hand-off message template

After the file is written, message the user along these lines:

> Round 1 design plan landed at `<root>/design.md`:
> - Title: <title>
> - Canonical term: `<term>` (if applicable)
> - Foundational decisions (`(R1)`): <count> — covering <one-sentence summary of what they cover>
> - Open questions: <count total> — `<a> blocks-v1 / <b> blocks-impl / <c> deferred / <d> exploratory`
> - Status: `Round 1: foundational decisions captured + open questions enumerated. <One sentence on what's most load-bearing-open.>`
>
> The highest-priority open questions surfaced (in tag order — `[blocks-v1]` first):
> 1. **`[blocks-v1]`** <Open question #1 + one-line indicative direction>
> 2. **`[blocks-impl]`** <Open question #2 + …>
> …
>
> ### Round 1 — completeness assessment
>
> **Verdict**: not-yet-complete
> **Open-question tag counts**: `<a> blocks-v1 / <b> blocks-impl / <c> deferred / <d> exploratory`
>
> **What's still load-bearing-open**:
> - <Section / question + tag + why it's load-bearing>
> - …
>
> **Recommended next-round focus**:
> - <Usually: 2–3 of the `[blocks-v1]` items, picked because their resolutions are reasonably independent of each other>
>
> Round 2 will be driven by you. When you're ready, ask trellis to iterate the design plan at `<root>` and tell me which questions to focus on.

Stop and wait for the user's direction.

---

## Additional Instructions

The **Trellis instruction precedence chain** (`~/.trellis/instructions.md` → `<repo-root>/.trellis/instructions.md` → instructions supplied with this invocation) is applied by `SKILL.md` before this file is dispatched, so it is already in effect; honor it. Canonical statement — the single source shared by every instruction file and subagent brief: [instruction-precedence.md](../specs/instruction-precedence.md).
