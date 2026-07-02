# Implementation Plan — Round Mechanics & Cross-File Checks

Companion to [implementation-plan.md](./implementation-plan.md) (document anatomy and authoring conventions) and [implementation-plan-templates.md](./implementation-plan-templates.md) (skeleton templates). This file holds the mechanics that drive an implementation plan forward and keep its files coherent: the round-by-round iteration model, supersession, deviations applied during execution, driving a round with the user, the round-end completeness assessment, the multi-file edit discipline, the cross-sprint coherence checklist, the around-corner concern checklist, and the execution handoff.

Operations that run a planning round over an implementation plan (impl-create, impl-iterate, impl-integrate-feedback, resolve-open-questions on an impl file) and the impl-plan reviewer load this file. The per-step execution subagents (step-executor, step-reviewer) and the impl-execute orchestrator do **not** need it.

---


## How the document set is produced

### The iteration model

Like design plans, implementation plans are produced through repeated conversation rounds with a human collaborator. Each round resolves a small number of open questions, updates the affected files, and (where helpful) appends an entry to the plan-level `decisions.md` / `status.md` files or to a sprint-scoped log inside a sprint file. The agent should never try to "finish" the plan in one pass — the rounds keep cognitive load low and make trade-offs visible.

**Where the logs live.** Plan-level (cross-cutting) decisions and the round-by-round audit trail live directly in the feature root as `decisions.md` and `status.md`. They are **not** sections inside `overview.md`. Each sprint file may additionally carry its own sprint-scoped Decisions log section (for calls that affect only that sprint) and an optional sprint-scoped Status / Feedback-incorporated section (for round entries that materially changed just that sprint). The split is deliberate: `decisions.md` and `status.md` are the plan's single-page indexes for "what got decided across the whole feature" and "how did this plan get here"; per-sprint logs are the local record inside the sprint that owns them.

A round generally goes:

1. **Read the entire plan**, every implementation-plan file in the feature root, before saying anything. The overview frames the philosophy; each sprint encodes a slice; the progress file shows what's been completed. Don't rely on prior conversation context.
2. **Triage Open Questions** — pick 1 to ~5 questions that can plausibly be resolved in this round without spawning many more.
3. **Surface alternatives clearly** to the user — present 2–3 viable options for each question with trade-offs, recommend one, and let the user pick or redirect.
4. **Update the affected files** for each resolution:
   - Rewrite the affected sprint section (or overview section) to reflect the chosen approach.
   - Add a tagged bullet to the relevant **Decisions log** — cross-cutting calls go in the plan-level **`decisions.md`**; sprint-specific calls go in the affected sprint file's Decisions log section.
   - Remove the resolved entry from **Open questions**.
   - Add any newly-surfaced sub-questions to Open questions for next round.
5. **Re-shape sprint boundaries when warranted.** A resolution may show that a sprint is too large (split it), too small (fold it into a neighbor), or sequenced wrong (swap order). Sprint files are renamed and the index updated; the progress file is regenerated.
6. **Purge obsolete prose.** When a decision invalidates earlier wording, delete the old wording — don't leave both versions side-by-side.
7. **Append a Status entry** for the round summarizing what changed — to the plan-level **`status.md`**, and (when a single sprint was materially changed) optionally to that sprint file's sprint-scoped Status / Feedback-incorporated section.
8. **Ensure internal consistency.** Rewriting one sprint may leak inconsistency into adjacent sprints (a new dependency, a deferred concern, a renamed helper). Either fix the consequence in the same round or add a note to Open questions for next round.
9. **Emit a completeness assessment to chat** at the end of the round (see § "Round-end completeness assessment"). This is the agent's recommendation on how close the plan is to "the next sprint can be picked up cold and shipped," what's still load-bearing-open, and which nits the user may want to resolve before handing the plan off to an implementer.

### Round 1 — scaffolding

Round 1 is initiated by a human pointing the agent at a design plan and (usually) sketching the high-level slicing they have in mind. The first round produces the skeleton, not the fully-detailed plan:

- The implementation-plan files are created in the feature root.
- `overview.md` is drafted — title, framing, link back to the source design plan, philosophy, locked feature-wide decisions, sprint roster (titles + one-line outcomes), dependency graph. **No Decisions log or Status section inside `overview.md`** — those live in `decisions.md` and `status.md`.
- `decisions.md` is initialized — usually empty, or with `(R1)`-tagged entries capturing the slicing decisions agreed in Round 1.
- `status.md` is initialized with a single Round 1 entry: `**Round 1**: scaffolding + sprint roster + open questions enumerated. _Next:_ <recommended Round 2 focus>.`
- One stub file per sprint is created (`01-<topic>.md`, `02-<topic>.md`, …) with at minimum the Goal, Prerequisites, and an initial Open Questions list scoped to that sprint.
- `progress.md` is initialized with a master checklist where every sprint shows its sprint-level title and (if known) its tentative steps. Each item starts unchecked.

Detailed step breakdowns within each sprint mostly come in later rounds. Don't pre-populate them with assumptions; surface unresolved shape as Open questions.

### Subsequent rounds — fleshing out sprints

Each subsequent round picks one (or a small number) of sprint files and pushes it from "stub" to "ready-to-execute":

- Lock the sprint's Key Design Decisions table.
- Add Architecture notes ("Why X" subsections) for non-obvious choices.
- Sketch the Public surface, request/response shapes, and error tables (when applicable).
- Decompose the sprint into ordered steps with concrete Actions / Deliverables / Verification.
- Add a Step Dependency Chart and an Acceptance Checklist.
- Update the master `progress.md` with the new step list for that sprint.

A sprint is "ready to hand off" when a junior engineer could pick it up cold and execute it without consulting the design plan for anything load-bearing.

### Supersession

When a later round invalidates a prior decision (cross-sprint or within a sprint):

1. Update the affected file(s) to reflect the new design.
2. Either edit the existing Decisions-log bullet (preserving its round tag) or add a new one tagged with the current round. Cross-cutting calls live in `decisions.md`; sprint-scoped calls live in the affected sprint file's Decisions log section.
3. Note the supersession explicitly in `status.md`: "Round 5: re-sliced Sprint 06; cross-cutting helper now lands in Sprint 04 instead. R3's helper-in-Sprint-06 decision superseded." If the supersession is sprint-scoped and that sprint has its own Status section, also note it there.
4. **Purge stale wording** from all files where it appears. Old paragraphs left in place become land mines.
5. **Update `progress.md`** if step ordering or step count changed.

### Living during execution: Deviations applied during implementation

Sprint files are also living during execution, not just during planning. When the *implementation* of a step diverges from what the sprint doc said — a column rename, a different idempotency-recovery shape, a swapped error-mapping layer, an extra constraint discovered during coding — the **sprint doc is updated in the same PR** that ships the divergence. There are two complementary surfaces for this:

- **Deviations applied during implementation** — a section near the top of the sprint doc listing the deltas between the original plan and what shipped. The semantic contract from the design plan stays preserved; only the implementation shape is allowed to shift.
- **Per-step Deviations / Post Mortem** — short subsections inline with the affected step, capturing detail too granular for the top-of-doc list (test count, parameter rename, regex tweak, etc.).

A sprint doc that no longer matches the code actively misleads the next reviewer. The PR that introduces the divergence owns the doc update.

### Honest framing for v1 / Tier 0 compromises

Like design plans, implementation plans ship compromises. Label them in place:
- `v1 ships with X — semantics improve once Y lands`.
- `Tier 0 trade-off: …`.
- `Acceptable at this scale; revisit if Z becomes hot.`
- `Schema-additive when we want it.`
- `Defers the chat fan-out to Sprint 11; Sprint 07 ships locally-correct behavior with a known cross-service gap.`

A reader should never have to guess whether a piece of a sprint is the long-term answer or a deliberate stop-gap.

---

## Driving a round with the user

### Continuing an existing implementation plan

1. **Read the feature root's implementation-plan files, end to end.** `overview.md`, `decisions.md`, `status.md`, `progress.md`, every sprint file, and `post-mortem.md` if it exists. Identify Open questions (overview-level + sprint-level), recent `decisions.md` entries (plus sprint-scoped Decisions logs), and the most recent `status.md` round.
2. **Pick 1–3 questions or 1 sprint** to focus this round. Prefer questions whose resolutions are reasonably independent.
3. **For each, frame the choice** for the user: 2–3 viable options with trade-offs, your recommendation, and one sentence on what gets unblocked by the answer.
4. **After the user answers**, update the affected files:
   - Sprint body gets the new design (purge the old wording).
   - Decisions log gets a tagged bullet — cross-cutting calls in `decisions.md`; sprint-scoped calls in the affected sprint file's Decisions log section.
   - Open questions loses the resolved entry; gains any newly-surfaced sub-questions.
   - `status.md` gets a new numbered entry (plus, optionally, a sprint-scoped Status entry inside the affected sprint file).
   - `progress.md` is updated if step ordering or step count changed.
5. **Re-read the affected sprint(s)** to catch any wording from earlier rounds that the resolution silently invalidated. Cross-check against the overview's locked-decisions table — sprint-level decisions never override overview-level ones.

### Starting a new implementation plan

The bootstrap workflow — taking a finished design plan and producing the initial scaffold — is captured in [`../instructions/impl-create.md`](../instructions/impl-create.md). That instruction file is the agent-facing brief for Round 1 specifically.

### What to do when the user doesn't know

- **Surface the analogues.** Find similar implementation plans in the repo (or the precedents under `plans/`) and present how they resolved the same shape question. Borrow generously; explain what you're borrowing.
- **Default to v1 simplicity.** When the user is genuinely undecided, prefer the smallest answer that doesn't paint into a corner — and note the alternative as deferred.
- **Document the punt.** "Deferred — revisit when concrete demand emerges" is a legitimate resolution. Tag it in the appropriate Decisions log (`decisions.md` for cross-cutting; the sprint file's Decisions log for sprint-scoped) so a future round can reopen with proper context.

### When sprint slicing changes mid-stream

Re-slicing is normal. When it happens:

- Update `overview.md`'s sprint roster, dependency graph, and any rationale paragraphs that reference the old shape.
- Rename / split / fold sprint files. Renumber if the file ordering would otherwise lie about execution order.
- **Shipped sprints are frozen from renumbering.** A sprint whose final step is `[x]` in `progress.md` keeps its number, filename, and checkboxes as-is — its identity is part of the shipped record, and renaming it would orphan the `post-mortem.md` section, the git history, and any external reference keyed to that number. Re-slicing renumbers only unshipped sprints; when a re-slice would collide with a shipped sprint's number, renumber the unshipped sprints around it rather than renaming the shipped file.
- Update `progress.md` to reflect the new structure.
- Add a Status entry **to `status.md`**: "Round 4: re-sliced 06 into 06-resolve-type and 07-deduplicate; re-numbered 07–10 → 08–11. R3's combined-worker decision superseded."
- Cross-references in adjacent sprints get updated in the same round.

---

## Round-end completeness assessment

After every round (including Round 1), the agent emits a completeness assessment to chat. This is **not** part of any plan file — it's a chat-only recommendation that helps the user decide whether to keep iterating, run an external review, or hand the plan off to an implementer.

The assessment is the agent's honest read on the plan's state. It is allowed to be opinionated; the user is allowed to disagree. Do not pad the "complete" verdict to be polite, and do not list every minor wording quirk to look thorough.

Implementation plans differ from design plans in two important ways for completeness:

1. **Implementation plans are a set of files directly in the feature root**, not a single doc. "Complete" requires every required file to be present and consistent with the others.
2. **Implementation plans are sequential by design**. Sprint 01 may be execution-ready while Sprint 06 is still a stub — that's normal and not a defect. The completeness assessment grades the plan's *current readiness for the next unshipped sprint*, not the readiness of every file equally.

### When an implementation plan is "complete enough"

There are three different "complete enough" thresholds; the assessment picks the most relevant one for the round just finished.

**Scaffold-complete** — Round 1 has just landed:

- `overview.md` is populated per the spec — title, framing, philosophy, architectural invariants, module/directory layout, cross-sprint conventions, sprint organization rationale, sprint roster (with one-line outcomes), dependency graph, feature-wide locked decisions table, out-of-scope across all sprints, open questions, what this plan does *not* try to do, how to read each sprint. **No Decisions log or Status section inside `overview.md`.**
- `decisions.md` exists directly in the feature root — empty, or with `(R1)`-tagged entries capturing slicing decisions agreed in Round 1.
- `status.md` exists directly in the feature root with a single entry: `**Round 1**: scaffolding + sprint roster + open questions enumerated. _Next:_ …`.
- One stub sprint file per roster entry, each with at least Goal, Prerequisites, Deliverables (best-effort first cut), Out of scope (forward-linked to owner sprints), and Open questions.
- `progress.md` has one section per sprint, no step bullets yet.
- No `post-mortem.md` (it's created lazily when the first sprint ships).
- The sprint roster slicing is agreed with the user.

**Sprint-NN-execution-ready** — the round just locked Sprint NN:

- Sprint NN's stub has graduated to a full sprint file: Locked Decisions table populated (10–25 rows; not a single row), Architecture notes ("Why X" subsections) for non-obvious shape decisions, Public surface sketched (when applicable) with request/response shapes and an error table, Progress checklist, Implementation Steps (5–10) each with Goal/Actions/Deliverables/Verification, Step Dependency Chart, Acceptance checklist.
- Every step is sliced as a compilable/testable boundary, or explicitly includes a Build/test boundary note explaining the unavoidable temporary breakage and the later step that restores it.
- Every step's Verification names specific assertions, commands, or test names — not "tests pass" / "compiles."
- `progress.md`'s Sprint NN section is in lockstep with the sprint file's Progress section.
- Sprint NN's Prerequisites all match a prior sprint's Deliverables (or a peer-service surface that exists, or an audit-or-build branch).
- Sprint NN's Out-of-scope items each forward-link to a sprint that owns them.
- Sprint NN's Open questions are empty *or* contain only items that don't block this sprint's execution.

**Plan-complete** — every sprint is execution-ready:

- Every sprint file has cleared the Sprint-NN-execution-ready bar.
- The cross-sprint coherence checks all pass — every item in § "Cross-sprint coherence checklist" (Prerequisite ↔ Deliverable matching, dependency-graph honesty, overview-vs-sprint decisions, module-tree drift, conventions drift, invariants honored, roster completeness, `progress.md` lockstep, `status.md` supersessions, open-question ownership, round-numbering, acceptance coverage).
- `progress.md` reflects the full step structure.
- Open Questions at every scope are empty *or* contain only items explicitly deferred with rationale.
- `decisions.md` and `status.md` are coherent with every sprint file's sprint-scoped Decisions / Status sections — round numbering consistent, supersessions paired with body edits, no orphaned references.
- Tone and conventions clean — no marketing words, no design-level rationale leaking into sprints, no implementation-spec specifics leaking up to the overview.

If the round didn't aim at a specific sprint (e.g., it was a feedback-incorporation round across many files), grade against whichever threshold the plan's overall state best matches.

### What to emit after each round

The chat output uses this exact structure (one block, terse, no preamble):

```markdown
### Round <N> — completeness assessment

**Verdict**: <not-yet-complete | substantially-complete | complete>
**Threshold graded**: <scaffold-complete | sprint-NN-execution-ready | plan-complete>

**What's still load-bearing-open** *(if not complete; omit otherwise)*:
- <file or sprint + section + the gap>
- …

**Top nits worth resolving before declaring done** *(if substantially-complete or complete; omit otherwise)*:
- <file + section + the wording / consistency / coverage issue>
- …

**Recommended next-round focus**:
- <one-line recommendation; usually the load-bearing-open items, or "lock Sprint NN+1" if the current sprint is execution-ready, or "hand off to implementation" if plan-complete>
```

Verdict semantics (relative to the chosen threshold):

- **`not-yet-complete`**: At least one load-bearing item from the threshold's checklist is still open or undecided. The "What's still load-bearing-open" block enumerates the gaps with file + section pointers.
- **`substantially-complete`**: All load-bearing items at this threshold are decided, but rough edges remain — wording inconsistencies, a `progress.md` ↔ per-sprint Progress mismatch, a Decisions log missing a recent decision's tag, marketing language to purge, an Out-of-scope entry without a forward-link. The plan *could* graduate at this threshold; the user may want to clean up first. The "Top nits" block enumerates them.
- **`complete`**: Load-bearing items at this threshold are decided *and* the affected files are internally clean. Recommend the next move — usually locking the next sprint, running `impl-review` for an external check, or handing the plan off to an implementer if `plan-complete`.

Calibration:

- **Always name the threshold.** A round that locks Sprint 03 doesn't grade against `plan-complete` — it grades against `sprint-03-execution-ready`. Listing "Sprint 04–11 are still stubs" as "load-bearing-open" against a Sprint 03 round is noise; that's the next round's work.
- **Don't oscillate.** If round N said `substantially-complete` at the same threshold and round N+1 only changed a comment, the verdict shouldn't drop back to `not-yet-complete`.
- **Cross-sprint coherence is load-bearing at every threshold above scaffold-complete.** A Prerequisite that names a Deliverable spelled differently in a prior sprint is a load-bearing-open item, not a nit.
- **Don't pad either side.** A 3-bullet "load-bearing-open" list is a real list. A 12-bullet "top nits" list is a sign the plan is not actually substantially-complete — promote the most material items into "load-bearing-open" and re-grade.
- **Cite the file + section.** Every bullet names the file *and* the section so the user can navigate (`Sprint 04 § Locked Decisions`, `overview.md § Dependency graph`, `progress.md § Sprint 06`). Bare-section references are too ambiguous across multiple files.
- **One block per round.** Don't emit multiple completeness assessments in the same round; the user reads the latest as authoritative.

This assessment is the agent's recommendation, not a gate. The user decides when to graduate. But the assessment forces the agent to take a position each round, which surfaces drift before it compounds across sprint files.

---

## Multi-file edit discipline

An implementation plan is a set of files that must agree at their seams. Most planning failures hide *between* files, not inside any one — a Prerequisite renamed in one sprint but left stale in the sprint that consumes it produces a plan where every file is internally coherent and only the seam is wrong. Any round that edits across files (an `impl-iterate` round, an `impl-integrate-feedback` pass, a `resolve-open-questions` integration) follows this discipline:

1. **List the file cluster before editing any file.** Which file *owns* the canonical wording the edit changes? Which files *reference* it (consume, forward-link, mirror, or restate)? Does `overview.md` need an update (module-tree, dependency-graph one-liner, sprint roster, feature-wide locked decisions)? Do `decisions.md` / `status.md` need an entry? Does `progress.md` — and the affected sprint's per-sprint Progress section — need updating? Write the cluster down before opening any file; a cluster you didn't enumerate is one you'll forget to finish.
2. **Edit canonical-owner first, consumers second.** The file that *owns* the renamed Deliverable / locked decision / module-tree entry is updated before any file that *references* it. Editing a consumer before its owner opens a window where both old and new wording coexist — and an incomplete pass leaves the plan in exactly that broken state.
3. **Grep the feature root for the rejected wording after the edit pass.** `grep -rn '<old wording>' <root>` — zero hits is the only acceptable result; any hit means a consumer was missed. This is the cheapest way to catch the silent drift the incorporation itself introduced.

`progress.md` and the per-sprint Progress sections stay in lockstep: edit one, edit the other in the same pass. The master `progress.md` wins on conflict; reconcile the per-sprint copy to it (unless the master is the broken side).

---

## Cross-sprint coherence checklist

The implementation plan is a *system* of files that must agree with each other. These are the checks that catch the failure modes living in the seams between sprint files — the highest-leverage class of planning bug, because each one is invisible to a reader (or reviewer) looking at any single file. Every round's sanity pass (`impl-iterate`), every plan review (`impl-review`), and the `plan-complete` completeness threshold walk this same list.

- **Prerequisite ↔ Deliverable matching.** Every sprint's Prerequisite is produced by a prior sprint's Deliverable, names a peer-service surface that exists, or is flagged as an audit-or-build branch. Every "deferred to Sprint NN" out-of-scope entry is owned by Sprint NN's Deliverables. Promises and obligations come in pairs — flag any pair missing a side.
- **Dependency-graph honesty.** `overview.md`'s dependency chart agrees with the per-sprint Prerequisites lists. A sprint that lists Sprint 04 as a prerequisite sits downstream of 04 in the graph. Cycles are slicing bugs, not graphs.
- **Overview vs. sprint-level decisions.** Sprint-level Locked Decisions tables *refine* the overview's Feature-wide locked decisions; they never *override* them. A sprint that silently picks a different soft-delete posture, idempotency-key shape, or test-location than the overview pinned is a critical inconsistency.
- **Sprint-vs-sprint agreement.** Two sprints that touch the same surface agree on its shape — a column, an error code, or an idempotency strategy fixed in one sprint isn't re-decided differently in another.
- **Module / directory layout drift.** Every file a sprint commits to creating appears in `overview.md`'s Module/Directory tree; no sprint proposes a path the tree contradicts.
- **Cross-sprint conventions drift.** Every sprint uses the test framework, error vocabulary, logger, branching workflow, and migration policy `overview.md`'s Cross-sprint conventions pinned — not a quietly different one.
- **Architectural invariants honored everywhere.** No sprint violates an invariant from `overview.md`'s "Architectural invariants" list; each sprint that touches one restates the relevant verification in its Acceptance checklist.
- **Sprint roster completeness.** Every roster row points at a real sprint file; every sprint file appears in the roster; numbering matches file order; titles match.
- **`progress.md` ↔ per-sprint Progress lockstep.** The master checklist and each sprint file's per-sprint Progress section are identical for that sprint's slice — no step-presence, ordering, or `[x]`/`[ ]` drift.
- **`status.md` supersessions paired with body edits.** A re-slice or supersession announced in `status.md` actually shows in the sprint files: old wording purged, new files in place, dependent Prerequisites updated, `progress.md` regenerated.
- **Open-question ownership.** Each open question (overview-level + sprint-level) names the sprint(s) it blocks; it doesn't cite a sprint that doesn't exist, and the blocked sprint acknowledges it.
- **Round-numbering coherence.** Decisions-log round tags across `decisions.md` and the sprint files use one coherent plan-level scheme (the counter is incremented in `status.md`); flag impossible tags (a sprint `(R6)` entry when `status.md` never reached Round 6).
- **Acceptance-checklist coverage.** Each sprint's Acceptance checklist covers every documented failure mode in its Public surface, every architectural invariant it touched, every "Subtle bug" gotcha in its steps, and the cross-sprint convention items (build / type / test / lint green; spec sync; tests at the right layer).

---

## Around-corner concern checklist

A stack-agnostic checklist of the failure modes a reviewer (and a planning round's sanity pass) should walk for any sprint that ships lifecycle, integration, or async behavior. These are the substance concerns that hide *inside* a sprint's design — distinct from the seam-level drift the cross-sprint coherence checklist catches. The list is shared: `impl-review` walks it per sprint, and a flesh-out round should self-check against it before declaring a sprint execution-ready.

- **Race conditions.** Two events arriving in either order. Two workers running the same task. A request landing during a state transition. Webhook + sync racing the same external state change.
- **Idempotency gaps.** Operations that re-fire on retry, pod restart, debounce flush, or manual re-sync, where the second fire isn't a no-op. Especially: external API calls or worker tasks not gated by a deterministic idempotency key.
- **Concurrency / transaction boundaries.** Read-before-BEGIN / write-after-COMMIT discipline. Cross-service calls inside transactions. Locks held across IO. Eager + worker-safety-net patterns where one is missing.
- **Partial success.** Financial action ✓, notification ✗ — vs. — financial action ✗, notification ✓ — vs. — both succeed but the row write failed. Each combination needs an answer.
- **External-system assumptions.** Behavior assumed about a partner API that isn't pinned in their docs / sandbox / contract. Webhook delivery semantics taken for granted (ordering, at-least-once, debounce).
- **Migration / rollout risk.** Schema changes without deploy-order discipline. Backfill assumptions. Missing feature-flag / kill-switch. Unsafe column adds (NOT NULL on populated tables, etc.).
- **Backwards compatibility.** Old clients seeing new state; new clients seeing old state. API contract changes without versioning. Data-shape evolution without a migration story.
- **Privacy / authorization.** Who can see what. Cross-tenant leakage. Soft-deleted rows reappearing. Data leaving its owning service. Access scope of new methods (`internal_function` vs. `public_api`, or the project's equivalent).
- **Observability.** Can support / oncall debug a failure from logs alone? Structured event-log fields named? Error paths emit useful context? Metrics for the cases that matter?
- **Scale boundaries.** "Acceptable at Tier 0" claims — are they quantified? What's the breaking point? What's the trigger to revisit?
- **GC / orphan rows.** Eager cleanup paths plus a worker safety net? What happens to attached child rows when the parent transitions terminal? What happens to references in peer services?

The *categories* are stack-agnostic; the *content* of each in your project depends on the architecture extracted from the project's conventions. A frontend or CLI project re-reads these through its own failure modes (stale closures, signal-handler reentrancy, cache-key collisions) — the posture translates even when the examples don't.

---

## How to use the plan to execute

A sprint is "execution-ready" when a junior engineer can:

1. Open `overview.md`, scan the philosophy + locked-decisions table + sprint roster (≤ 5 minutes).
2. Open the target sprint file, read Goal → Prerequisites → Deliverables → Out of scope → Locked decisions → Architecture notes (≤ 15 minutes).
3. Pick up Step 1 from the Implementation Steps section and execute it without consulting the design plan unless an explicit pointer says to.
4. After each step, mark its checkbox `[x]` in **both** the per-sprint Progress section **and** the master `progress.md`. (Or just the master if the per-sprint section has been removed for compactness — but typically both.)
5. Record any Deviations applied during implementation **in the same PR** that ships the divergence — top-of-doc list for cross-cutting deltas, inline `### Step N deviations` subsection for granular ones.
6. After the last step, walk the Acceptance checklist; only when every box is `[x]` is the sprint shippable.
7. In the same PR that checks off the final step, distill the sprint's Post Mortem section into a sprint-keyed entry in the feature root's `post-mortem.md` (creating the file if it doesn't yet exist). If there's nothing post-mortem-worthy, the entry is a single `_No post-mortem-worthy observations._` bullet — don't omit the sprint heading.

If the engineer hits a question the sprint doc doesn't answer:

- **Easy answer**: it's in `overview.md`'s locked-decisions table, `decisions.md`, or the design plan. Resolve and continue.
- **Mid-difficulty**: it's a sprint-level call the planning rounds missed. Surface to the planning collaborator (the user / the senior engineer) to add a sprint-level locked decision. Don't make the call inside a step.
- **Hard answer**: it's a design-level call the design plan didn't make. Stop the sprint, log the question into the design plan's Open questions, escalate. Don't re-decide architecture inside an implementation plan.
