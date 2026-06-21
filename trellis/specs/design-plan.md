# Design Plan Documents — Authoring Guide

Instructions for an LLM agent on producing **design plan documents**. This guide is the agent-facing brief for collaborating with a human user to produce one of these documents from scratch — or to drive an existing one forward through another planning round.

---

## What these documents are

A **design plan** is a living engineering artifact that captures how a feature, service, or subsystem is conceived, structured, and bounded — anchored to the *why* behind every decision. A finished plan lets another engineer come in cold and recover (a) what the system does, (b) why each shape was chosen, (c) what was deliberately excluded, (d) what is still open.

A design plan IS:
- A high-fidelity description of a system's data model, lifecycle, API shape, and cross-service contracts.
- A record of the decisions that resolved the design, each tagged with the round in which it landed.
- A record of the open questions that still need resolution.
- A round-by-round audit trail (the "Status" log) showing how the doc evolved.

A design plan is NOT:
- A build/sprint spec — it's the substrate from which sprint plans are later derived. Implementation specifics (sprint sequencing, file paths, exact types) are downstream.
- A user-facing product brief — it speaks engineering vocabulary and assumes the reader knows the stack.
- A finished/frozen reference — it's expected to evolve through rounds and be partially superseded over time.

The reader you are writing for: another engineer with no prior context on this design, joining the project today, who needs to ship code against it.

---

## How the document is produced

### The iteration model

These documents are produced through repeated conversation rounds with a human collaborator. Each round resolves a small number of open questions, updates the body of the doc accordingly, and appends an entry to the Status log. The agent should never try to "finish" a doc in one pass — the rounds are a feature, not a workaround. They keep cognitive load low and make the trade-offs visible.

A round generally goes:

1. **Read the entire doc**, every section, before saying anything. Don't rely on prior conversation context.
2. **Triage Open Questions** — pick 1 to ~5 questions that can plausibly be resolved in this round without spawning many more.
3. **Surface alternatives clearly** to the user — present 2–3 viable options for each question with trade-offs, recommend one, and let the user pick or redirect.
4. **Update the doc** for each resolution:
   - Rewrite the affected body section to reflect the chosen design.
   - Add a tagged bullet to the **Decisions log** (e.g. `(R7)`).
   - Remove the resolved entry from **Open questions**.
   - Add any newly-surfaced sub-questions to Open questions for next round.
5. **Purge obsolete prose.** When a decision invalidates earlier wording, delete the old wording — don't leave both versions side-by-side.
6. **Append a Status entry** for the round summarizing what changed.
7. **Ensure internal consistency**. Rewriting sections based on decisions may lead to inconsistencies with other areas of the document. If it is unclear how to resolve, add the new inconsistencies to the open questions section.
8. **Emit a completeness assessment to chat** at the end of the round (see § "Round-end completeness assessment"). This is the agent's recommendation on whether the plan is ready to graduate to an implementation plan, what's still load-bearing-open, and which nits the user may want to resolve before declaring the plan done.

### Round 1 — scaffolding

Round 1 should be initiated by a human explaining some initial high-level decisions and foundational ideas.

The first round produces the skeleton, not the design:
- Title and one-paragraph framing.
- Canonical-term clarification if the domain has terminology drift.
- Cross-references to adjacent plans/wiki.
- A first cut at **Foundational decisions** (or Purpose / Scope) — the load-bearing assumptions.
- **Scope and non-goals** — what's in, what's out (be aggressive about what's out).
- An initial **Open questions** list of everything not yet resolved, **each tagged at creation** with `[blocks-v1]`, `[blocks-impl]`, `[deferred]`, or `[exploratory]` per § "Open questions" → "Severity tag taxonomy." Round 1 lists are usually heavy on `[blocks-v1]` and `[blocks-impl]`; few or no `[deferred]` or `[exploratory]` is normal at this stage.
- A `Round 1` entry in Status saying "foundational decisions captured + open questions enumerated."

Schema, API surface, lifecycle, worker logic, GC — these mostly come in later rounds. Don't pre-populate them with assumptions; surface the unresolved shape as questions.

### Supersession

When a later round invalidates a prior decision:

1. Update the body to reflect the new design.
2. Either edit the existing Decisions-log bullet (preserving its round tag) or add a new one tagged with the current round.
3. Note the supersession explicitly in the Status log: `**Round 7**: cross-conversation sync redesigned — replaced GET /chat/sync with POST /chat/conversations/sync. R5's contiguity-cache rule superseded.`
4. **Purge stale wording** from the body. Old paragraphs left in place become land mines for future readers.

### Honest framing for v1 / Tier 0 compromises

Every plan ships compromises. Label them in place:
- `v1 ships with X — semantics improve once Y lands`.
- `Tier 0 trade-off: …`.
- `Acceptable at this scale; revisit if Z becomes hot.`
- `Schema-additive when we want it.` (For deferred features that the schema is reserving room for without committing to.)

A reader should never have to guess whether a piece of design is the long-term answer or a deliberate stop-gap.

---

## Document anatomy

Drop sections that don't apply (e.g. no Schema if the design is purely architectural). Add domain-specific sections where the design demands them (e.g. `Privacy: 1:1 vs group` in chat, `Type Resolution` in measurements, `Garbage collection` in media). The list below is the canonical superset, in roughly the order it should appear.

### 1. Title & framing block

```
# <Subsystem> — planning doc          (or)  # <Domain Title>
```

Followed by 1–2 sentences saying what the doc is and what status it's at:

> Living planning artifact for how the `chat` service represents conversations, messages, reactions, edits, deletes, and how delivery / push fan-out is wired. Updated on a rolling basis as we agree on details. **Not a build spec yet** — design conversation in progress.

If the domain has terminology drift between casual speech and the code, fix it here:

> **Canonical term:** `conversation`. We may say "thread" in discussion but the data model, schemas, code, and API all use `conversation`.

### 2. Cross-references

A short bulleted list of adjacent plans, wiki pages, or code paths the reader should be aware of, each annotated with what it contributes:

```
- [[backend/service-overview]] — `chat` sits below `content`/`social`/`auth`/`safety` in the DAG…
- [[backend/db-conventions]] — UUIDv7 PKs, `timestamps` helper, `set_updated_at` trigger…
```

This is not exhaustive linking — it's the set the reader needs to follow this plan. Don't recapitulate the linked content here.

### 3. Foundational decisions (or Purpose / Scope)

The most important section. Two common shapes:

**Shape A — numbered foundational decisions list.** A flat 1..N list. Each item leads with a bold sentence stating the decision, then a paragraph (or several) of rationale. Numbered so they can be referenced by ID elsewhere ("per decision #6, the asset is `scope='pending'` until attached"). Use this when the design is multi-faceted and the rationale lives at the foundational level. Examples in chat-and-notifications.md, contact-book-ordering.md, media-service.md.

**Shape B — prose Purpose + Scope.** Title-style headers (`## Purpose`, `## Scope and Non-Goals`, `## Service Boundaries`) explaining what the system does, what's in vs. out, and which service owns what. Use this when the design is a self-contained product surface. Example: measurements.md.

Most plans benefit from elements of both. Pick the primary structure that matches the work and add the other where it helps.

### 4. Scope and Non-Goals

Explicit. **In scope** + **Out of scope (for v1)**, each as a bulleted list. Be aggressive about declaring things out of scope — this is the highest-leverage section for keeping the rest of the doc tight.

### 5. Service boundaries / data ownership

When the work crosses service lines, name which service owns which tables and which crosses are allowed. Cite the project rule (e.g. "no cross-schema FKs", "data *about* a patient belongs in the patient service"). One paragraph is usually enough.

### 6. Schema / Data model

If the work touches the database, the doc spells out:

- **Per-table column tables** in markdown. Each row's "Notes" cell carries the column's nullability conditions, what it references (and explicitly whether or not it's an FK), discriminator semantics, and any domain constraints:

```
| Column | Type | Notes |
|---|---|---|
| `id` | `uuid` (v7) PK | ... |
| `target_id` | `uuid` nullable | References rows in peer schemas. **No FK declaration** — cross-schema FKs are banned per [[backend/social-schema#no-cross-schema-fks]]. |
```

- **Indexes & constraints** in fenced SQL:

```sql
CREATE UNIQUE INDEX uniq_group_lab_order_laboratory
  ON measurement_groups (labOrderId, laboratory)
  WHERE labOrderId IS NOT NULL AND deletedAt IS NULL;
```

- **CHECK constraints** as plain rules.
- **Polymorphic-parent / discriminated-union** semantics as inline rules + Zod sketches.
- **"Why X" subsections** for non-obvious shape decisions: *Why two parent tables*, *Why a storage_providers table instead of env config*, *Why polymorphic parent over subclass tables*. The Notes column is for per-column rationale; "Why X" subsections are for table-shape rationale.
- **Relationships table** when the cardinality web is non-trivial:

```
| Relationship | Cardinality | Notes |
|---|---|---|
| Patient : MeasurementGroup | 1:N | Event-shaped sources |
| MeasurementGroup : Measurement | 1:N | Polymorphic parent; mutually exclusive with series |
```

### 7. Lifecycle / operational sections

The longest sections. Cover whatever lifecycle questions the system raises:

- Creation flows — who calls what, idempotency keys, race conditions.
- Mutation flows — append-only? in-place updates? supersession via new rows?
- Ingestion / re-ingestion / soft-delete / hard-delete.
- Sync / cache coherence / cursor design.
- Worker tasks — what runs async, on what cadence, against what query.
- Garbage collection — eager + worker safety net.

Use concrete code-shaped pseudocode or SQL when it disambiguates intent:

```sql
WITH client_state AS (
  SELECT * FROM jsonb_to_recordset(:client_map_json)
    AS t(conversation_id uuid, last_message_id uuid)
), ...
```

### 8. API surface (sketch — not normative)

A markdown table is the standard form:

```
| HTTP | Path | Notes |
|---|---|---|
| `POST` | `/chat/conversations` | Create conversation row + participant rows. … |
```

Distinguish `internal_function` (service-to-service) vs. `public_api` (client-callable) endpoints. The job here is shape, not field-by-field detail; field-level shape lives in transport-schema sketches:

```ts
const labResultPayload = z.discriminatedUnion('valueType', [...]);
```

### 9. Cross-service contract / Integration points

When the design touches more than one service, dedicate a section to the contract:
- What helpers each peer must expose, with `accessScope` annotations.
- The read-before-BEGIN / write-after-COMMIT discipline (or the project's equivalent).
- What happens if a post-commit call fails (eager + worker GC pattern).

### 10. Privacy / Authorization / Safety

For features with non-trivial access-control stories: state the predicate for each access scope, distinguish DB-layer vs. resource-layer vs. caller enforcement, and walk the "what if X" cases (user leaves a conversation, post is soft-deleted with attached media, user is flagged).

### 11. UX / Surface

If the design has a visible product surface, sketch it: layout, filters, group-by toggles, empty states, cross-links to other surfaces. Doesn't have to be Figma-precise — text-and-bullet descriptions of the surface are enough to ground the data model.

### 12. Open questions

A numbered list of items not yet decided. Each entry includes:
- A **severity tag** in square brackets (see taxonomy below) — placed immediately after the bold title so it surfaces in any skim.
- The question, briefly.
- Why it's open / what makes it hard.
- An indicative resolution direction or an explicit "deferred until X."
- Whether it blocks anything currently in scope (named explicitly when the tag is `[blocks-v1]` or `[blocks-impl]` — e.g., "blocks the schema sketch in §6," "blocks impl-plan Sprint 01").

#### Severity tag taxonomy

Every Open question carries exactly one tag. Pick the most blocking one if multiple apply:

- **`[blocks-v1]`** — must be resolved before v1 ships. The design has a hole this question fills; without an answer, the system can't be built or can't be built correctly. Includes most foundational shape questions, schema decisions the lifecycle depends on, contract clauses with peer services, and access-control predicates.
- **`[blocks-impl]`** — must be resolved before an implementation plan can start (or before a specific sprint can be locked, if an impl plan exists). Less load-bearing than `[blocks-v1]` — the design *concept* survives without it, but the impl-plan author can't sequence sprints or lock decisions without the answer. Includes most "how do we sequence the rollout" questions and "which surface lands first" questions.
- **`[deferred]`** — explicitly punted to a later round / phase / version. The design *could* answer it now but is deliberately not. The entry includes the punt rationale ("deferred until X concrete demand emerges," "deferred to v2 — schema-additive when we want it," "deferred until <peer team> confirms <thing>"). Does not block v1 ship.
- **`[exploratory]`** — surfaced for context but not yet shaped into a question that needs an answer. The doc records the open thread so a future round can pick it up; the current design does not depend on it. Lowest-priority bucket. Should be the smallest portion of any Open questions list.

The tag is the user's triage handle. A 30-item list with proper tags can be scanned in 30 seconds; the same list without tags requires reading every entry. When the user asks "what blocks impl?" the answer is `grep -c '\[blocks-impl\]'`.

#### Round mechanics for tags

- **Tag at creation.** Every new Open question gets a tag the moment it's filed. Untagged entries are a process bug.
- **Promote when shape clarifies.** An `[exploratory]` question that crystallizes into a real shape question gets promoted to `[blocks-impl]` or `[blocks-v1]` in the round that crystallizes it. Note the promotion in the Status entry.
- **Demote when scope shrinks.** A `[blocks-v1]` question that the round downgrades to "we can ship without this" becomes `[deferred]` with a punt rationale. Note the demotion in the Status entry.
- **Resolution is bucket-agnostic.** When a round resolves an entry — at any tag — the entry **moves to** the Decisions log; it does not stay in Open questions with a "(resolved)" tag.

### 13. Deferred (out of scope for this plan)

Distinct from "Out of scope (v1)" in the Scope section: this is the longer list of things the design *could* support but *isn't doing* in this plan. Often forward-looking — "schema-additive when we want it" entries. Each item is one bullet with a brief reason.

### 14. Decisions log (resolved)

A bulleted list of decisions made during prior rounds, each tagged with the round number:

```
- **Term:** `conversation` is canonical. (R2)
- **`unreaction` is its own append-only row** referencing the prior `reaction` row by id. (R2)
- **`preempted_by` denormalization on creation rows.** … (R9)
```

Each bullet leads with a bold phrase that names the call. The list lets a returning collaborator scan to current state without re-reading the body. It also prevents re-litigation: when a future round wants to revisit a decision, the bullet is the anchor it argues against.

#### Compressing older entries

The Decisions log grows monotonically as rounds accumulate. Without active pruning, R20's reader is wading through ten rounds of obsolete detail to find what's load-bearing now. When you append a new entry, also walk the older entries and condense any whose detail no longer carries weight:

- **Superseded entries.** A bullet whose call has been replaced by a later round keeps its round tag and bold lead but drops the rationale paragraph: `- **<lead>.** Superseded by R<m>. (R<n>)`. The audit trail (round number + supersession pointer) survives; the now-irrelevant prose does not.
- **Resolved-and-irrelevant sub-decisions.** A bullet that locked a detail whose parent decision is itself now superseded can be collapsed to the same one-line marker, or removed entirely if the parent's supersession entry already names the sub-decision's death.
- **Ancient detail that still holds but no longer needs the full rationale.** A 20-round-old decision that's still load-bearing but settled enough that nobody re-litigates it can drop its defense paragraph and keep just the bold lead + tag.

Keep the last 2–3 rounds in full; keep any still-actively-load-bearing entry in full regardless of age. The goal: a returning collaborator skims the log in seconds, not minutes, and the entries they see at full length are the ones that still shape the design.

### 15. Status / Rounds

A flat numbered list — one entry per planning round — capturing what changed *and* what to focus on next:

```
- **Round 1**: foundational decisions captured + schema sketch + open questions enumerated. _Next:_ resolve the privacy predicate for participant invites (Q3 [blocks-v1]).
- **Round 2**: naming → `conversation`; REST verbs split per resource type; … _Next:_ lock the sync cursor shape (Q7 [blocks-impl]).
- **Round 9**: introduced `preempted_by` denormalization on `chat.messages` … _Next:_ graduate to implementation plan.
```

Each entry has two clauses:

1. **What changed.** A summary of what this round resolved or superseded. Cite supersessions explicitly (`R7 supersedes R5's contiguity-cache rule`).
2. **`_Next:_` clause** — a one-line italic tail naming the recommended next-round focus, citing the Open question(s) by ID + tag. This persists the agent's between-rounds recommendation into the doc itself, so a user picking the doc up a week later via `design-iterate` doesn't lose the chat-only handoff context. When the plan is complete, the `_Next:_` clause is `graduate to implementation plan` (or `run design-review for an external check first`).

The Rounds log is the doc's git log. Scan it to recover narrative context: "why does the doc say X" → "round 5 supersedes round 3 because Y." Read the latest `_Next:_` clause to recover the agent's prior recommendation for what to work on now.

#### Compressing older Status entries

Like the Decisions log, the Rounds log grows monotonically and needs active pruning. When you append a new round, also walk the older entries and condense any whose detail no longer carries weight:

- **Rounds whose `_Next:_` clause is long-finished** (the work it recommended landed many rounds ago) can drop the `_Next:_` clause entirely.
- **Rounds whose summary detail is no longer load-bearing** (the decisions they narrate were themselves superseded by later rounds, and the supersession is recorded in the superseding round's entry) can collapse to a one-line summary: `**Round <n>**: <one-clause summary>`. The supersession pointer must remain readable somewhere — either in the older entry's residual line, or (cleaner) in the superseding entry.

Keep the last 2–3 rounds in full so the recent trajectory and the current `_Next:_` are intact. Never delete a round entry outright — the round numbering must stay contiguous and grep-able. The goal is to keep the log skimmable as N grows, not to falsify the audit trail.

### 16. See also

End-of-doc cross-references: closely related plans, files in the codebase, prior incidents, design discussions. This is for forward navigation; the Cross-references at the top are for backward context.

---

## Tone, voice, and conventions

- **Direct and concrete.** Prefer "we INSERT a row" to "the system persists an entity record." Prefer "the worker scans `WHERE …`" to "a reconciliation process examines candidates."
- **Justify every non-obvious decision.** "Why X" subsections, inline rationale paragraphs, trade-off tables. The reader should never wonder *why* we did the thing.
- **Surface trade-offs explicitly when alternatives competed.** A `✅ / ⚠️` list is the canonical form:

```
- ✅ One sync mechanism. The "walk backwards" loop sees reactions in the same pass.
- ✅ Append-only is consistent for every state change in the conversation.
- ⚠️ A super-popular message with hundreds of reactions inflates the conversation sync stream. Acceptable at Tier 0; revisit if hot.
```

- **Be honest about v1 compromises.** Label them: `Tier 0 trade-off`, `v1 ships with X — semantics improve once Y lands`, `acceptable at this scale; revisit if Z becomes hot`.
- **Future-aware, not future-driven.** Note "schema-additive" or "the column stays for a future X" when reserving space — but do not design around hypothetical requirements. If a column is reserved for a future feature, justify it with one line and accept it'll be NULL forever otherwise.
- **No marketing words.** Avoid "elegant," "clean," "best-in-class," "robust," "powerful." If a design is good, the rationale should make that obvious.
- **Bold the decision lead.** Each foundational-decision item, each Decisions-log bullet, leads with a bold phrase that names the call. Skim-readability is critical.
- **Link, don't recapitulate.** When a rule lives in a wiki page or another plan, link to it and trust the reader.
- **Active voice about the system.** "The worker scans," "the manager partitions," "ingestion is best-effort."
- **Identifiers in backticks.** `requireRecordSync`, `chat.messages`, `MEDIA_MAX_BYTES_IMAGE`, `'pending'`.
- **Wiki-style links** if the project uses them (`[[backend/social-schema#no-cross-schema-fks]]`); otherwise relative markdown links.
- **Em-dashes for asides;** parentheses for quick clarifications. Don't mix.
- **Dates in absolute form** (`2026-05-08`), not relative (`Thursday`, `last week`). Relative dates rot.

---

## Recurring constructs you should reach for

- **"Why X" subsections** — short paragraphs justifying a non-obvious shape decision. Almost every schema section earns at least one.
- **Per-flow allowlist tables** — when N entities behave differently along the same axis, tabulate. Example from chat-and-notifications.md:

```
| Type | Bumps `last_message_*` | Bumps `last_ordering_event_at` | Triggers push? |
|---|:-:|:-:|:-:|
| `content` | ✓ | ✓ | ✓ |
| `edit`    | ✓ | — | — |
```

- **Inline pseudocode SQL** — when a query plan is the actual design decision, write the SQL.
- **Code-shape interface sketches** — `abstract` classes, Zod schemas, type aliases. Not full implementations; enough that a reader can pattern-match. Example from contact-book-ordering.md:

```ts
abstract bulkLookupByPhone: (
  request: { phones: string[]; excludeUserId?: string },
) => Promise<{
  discoverableUsers: Array<{ userId: string; phoneE164: string }>;
  invitablePhones: string[];
}>;
```

- **Worked examples / concrete snippets** — for unusual data shapes or wire formats. JSON / JSONC blocks with comments are the standard form:

```jsonc
{
  "limit": 100,
  "lastConversationSyncAt": "2026-04-27T18:14:33.421Z",
  "conversations": [ { "conversationId": "...", "lastMessageId": "..." } ]
}
```

- **Numbered foundational decisions** — for design plans of meaningful complexity. Letting downstream sections reference "per decision #6" keeps cross-referencing tight.
- **Sequencing suggestion** (when the plan covers a multi-step rollout) — a short "Sequencing suggestion (not a sprint plan)" subsection at the end. Numbered steps with what each unblocks. Explicitly NOT a sprint plan.
- **Known limitations (inherited)** — a short subsection enumerating limits the design accepts but didn't introduce. Keeps a future reader from blaming this plan for things that are upstream.

---

## Anti-patterns to avoid

- **Don't skip "why."** A column type without a reason is half a decision.
- **Don't bury supersession.** If round N invalidates round M, the body must reflect the new design AND the rounds log must call out the supersession explicitly.
- **Don't leave dead text** when a decision changes the design. Purge the obsolete wording from the body and let the Decisions/Status logs carry the audit trail.
- **Don't speculate beyond the scope.** "This will probably also be useful for X" is a comment for the open-questions list, not a load-bearing assumption inside the body.
- **Don't pre-build extensibility hooks for hypotheticals.** Reserved columns/enums are fine when they're cheap and clearly justified ("schema-additive when dedup ships"). They're not fine when they imply a roadmap nobody has agreed to.
- **Don't conflate `status` (lifecycle) and `deletedAt` (operational soft-delete).** Different concerns. If a system has both, distinguish them in writing.
- **Don't promise type-safety/runtime-validation guarantees the schema doesn't enforce.** If a constraint is application-layer, say so.
- **Don't model billing / external / cosmetic codes in core domain tables** (e.g., CPT codes alongside lab analytes). Reserve those for the layer that owns them, and explicitly document the decision.
- **Don't auto-resolve the user's open question.** When a question requires a judgment call, surface alternatives and let the user choose. The agent's job is to frame the call sharply, not to make it.
- **Don't append `(resolved)` tags inside Open questions.** Move the entry to Decisions log.
- **Don't write build-spec specifics** (file paths, exact TypeScript implementations, specific function names beyond illustrative). Those belong in sprint plans, not design plans.

---

## Pre-hand-off sanity checklist

Before handing a round back to the user — whether bootstrapping Round 1 (`design-create`) or driving a later round (`design-iterate`) — walk this checklist over the file and fix what you find *in the same round*. Each item has cost the system a round when missed. The create/iterate skills layer a few step-specific checks on top of this shared set.

- **Every decision has rationale.** A bold lead (a foundational-decision item or a Decisions-log bullet) with no paragraph following it is half a decision — add the rationale or move the item to Open questions.
- **Every Open question carries exactly one severity tag** (`[blocks-v1]` / `[blocks-impl]` / `[deferred]` / `[exploratory]`). Untagged entries are a process bug.
- **Every cross-reference resolves.** Each link points at a real adjacent doc / wiki page / code path. Mark broken or unauthored references explicitly (`(TBD — not yet authored)`) rather than letting them rot silently.
- **No marketing words** — walk for `elegant`, `clean`, `robust`, `powerful`, `best-in-class` and purge.
- **No build-spec specifics** — no exact file paths beyond illustrative, no exact TypeScript implementations, no specific function names beyond illustrative. Those belong in the implementation plan.
- **No `(resolved)` tags inside Open questions** — resolved entries move to the Decisions log.
- **v1 / Tier-0 compromises are labeled in place** — `Tier 0 trade-off`, `v1 ships with X — semantics improve once Y lands`, `acceptable at this scale; revisit if Z becomes hot`. A reader should never have to guess whether a piece is the long-term answer or a deliberate stop-gap.
- **Tone matches the guide** — active voice about the system; identifiers in backticks; em-dashes for asides; absolute dates only.

---

## Driving a round with the user

### Continuing an existing doc

1. **Read the entire doc, end to end.** Identify Open questions, recent Decisions log entries, and the most recent Status round.
2. **Pick 1–3 questions** to focus this round. Prefer questions whose resolutions are reasonably independent (so they don't snowball each other).
3. **For each, frame the choice** for the user: 2–3 viable options with trade-offs, your recommendation, and one sentence on what gets unblocked by the answer.
4. **After the user answers**, update the doc:
   - Body section gets the new design (purge the old wording).
   - Decisions log gets a tagged bullet.
   - Open questions loses the resolved entry; gains any newly-surfaced sub-questions.
   - Status / Rounds gets a new numbered entry summarizing what changed and (if applicable) what was superseded.
5. **Re-read the affected sections** to catch any wording from earlier rounds that the resolution silently invalidated.

### Starting a new doc

1. **Confirm the title and the canonical-term clarification** with the user.
2. **Draft the framing block** — 1–2 sentences saying what the doc is and what status it's at. Default to `Living planning artifact … not a build spec yet`.
3. **Draft the cross-references list** — adjacent plans, wiki pages, code paths the reader will need. One line each, annotated.
4. **Surface the foundational decisions** by asking the user the 5–8 highest-leverage shape questions: scope, ownership, primitives, lifecycle shape, reach into other services. Capture the answers in numbered form.
5. **Sketch the schema** if applicable — but flag every column whose type / nullability / constraint is uncertain as an Open question rather than guessing.
6. **Initialize Open questions** with everything you didn't resolve in this round.
7. **Initialize Status / Rounds** with `**Round 1**: foundational decisions captured + open questions enumerated.`

### What to do when the user doesn't know

- **Surface the analogues.** Find similar systems / prior plans that resolved the same shape question and present those resolutions as starting points.
- **Default to v1 simplicity.** When the user is genuinely undecided, prefer the smallest answer that doesn't paint into a corner — and note the alternative as deferred.
- **Document the punt.** "Deferred — revisit when concrete demand emerges" is a legitimate resolution. Tag it in the Decisions log so a future round can reopen with proper context.

---

## Round-end completeness assessment

After every round (including Round 1), the agent emits a completeness assessment to chat. This is **not** part of the doc body — it's a chat-only recommendation that helps the user decide whether to keep iterating, run an external review, or graduate to an implementation plan.

The assessment is the agent's honest read on the plan's state. It is allowed to be opinionated; the user is allowed to disagree. Do not pad the "complete" verdict to be polite, and do not list every minor wording quirk to look thorough.

### When a design plan is "complete"

A design plan is complete enough to graduate to an implementation plan when **all** of the following hold:

- **Foundational decisions are locked.** The numbered list (or Purpose/Scope prose) has no `TBD`, no "leaning toward X," no "we'll figure this out." Each foundational decision has a rationale paragraph.
- **Scope and Non-Goals are explicit.** Both the in-scope and out-of-scope-for-v1 lists are populated. No silent "well, we'd probably also do…" assumptions in the body.
- **Schema / data model is sketched** (when applicable). Per-table column tables, indexes, constraints, polymorphic-parent rules, and the "Why X" subsections for non-obvious shape decisions are all present. Open questions about column types / nullability are resolved or explicitly deferred.
- **Lifecycle / operational sections cover the load-bearing flows.** Creation, mutation, soft-delete, sync, GC, idempotency — for the flows the system actually uses, the design is committed. Failure modes (peer down, replay, race, partial commit) have an answer or are filed as deferred with rationale.
- **API surface is sketched** (when applicable). Endpoint table is present; field-level shapes are sketched in Zod / type-alias form; access scopes (`internal_function` vs. `public_api`) are tagged.
- **Cross-service contracts are defined** (when applicable). Required peer helpers are named with their `accessScope`. Read-before-BEGIN / write-after-COMMIT discipline (or the project's equivalent) is stated.
- **Privacy / authorization** (when applicable) — the access predicate for each scope is stated; cross-tenant leakage is addressed.
- **Open questions list contains zero `[blocks-v1]` and zero `[blocks-impl]` entries** — only `[deferred]` and `[exploratory]` tags remain. Each `[deferred]` item carries a punt rationale; `[exploratory]` items are explicitly noted as "surfaced for context, not load-bearing." This is the literal grep-able test for design-plan completeness.
- **Decisions log and Status log are coherent** — every body decision has a tagged log entry; Status entries narrate the round-by-round trajectory honestly; supersessions are called out.
- **Tone and conventions are clean** — no marketing words, no build-spec specifics, identifiers in backticks, dates absolute.

If any of these is not true, the plan is **not yet complete**.

### What to emit after each round

The chat output uses this exact structure (one block, terse, no preamble):

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

Verdict semantics:

- **`not-yet-complete`**: At least one load-bearing item from the "complete" list above is still open or undecided. The plan should not graduate to impl yet. The "What's still load-bearing-open" block enumerates the gaps.
- **`substantially-complete`**: All load-bearing items are decided, but the plan still has rough edges — wording inconsistencies, an Open Questions list with stale entries, a Decisions log missing a recent decision's tag, marketing language to purge. The plan *could* graduate, but the user may want to clean up first. The "Top nits" block enumerates them.
- **`complete`**: Load-bearing decisions are made *and* the doc is internally clean. Recommend graduating to an implementation plan (or running `design-review` for an external check first, if the user wants one).

Calibration:

- **Don't oscillate.** If round N said `substantially-complete` and round N+1 only changed a comment, the verdict shouldn't drop back to `not-yet-complete`. Verdicts are about plan state, not round size.
- **Don't pad either side.** A 3-bullet "load-bearing-open" list is a real list. A 12-bullet "top nits" list is a sign the plan is not actually substantially-complete — promote the most material items into "load-bearing-open" and re-grade.
- **Cite the section.** Every bullet names the affected section ("Schema § `chat.messages`", "Open questions Q4") so the user can navigate without re-reading the doc.
- **One block per round.** Don't emit multiple completeness assessments in the same round; the user reads the latest as authoritative.

This assessment is the agent's recommendation, not a gate. The user decides when to graduate. But the assessment forces the agent to take a position each round, which surfaces drift before it compounds.

---

## Skeleton template

Drop sections that don't apply. Add domain-specific sections where the design needs them.

```markdown
# <Domain> — planning doc

<1–2 sentences: what this doc covers, what status it's at,
"not a build spec yet" if applicable>.

**Canonical term:** `<term>`. <when to use; what synonyms it replaces>.

Cross-references:
- [[<wiki-or-doc-path>]] — <what it contributes>
- [[<...>]] — <...>

---

## Foundational decisions

1. **<Decision lead in bold>.** <Rationale paragraph(s).>
2. **<...>.** <...>
3. ...

---

## Scope and Non-Goals

**In scope:**
- ...

**Out of scope (for v1):**
- ...

---

## Service boundaries / Data ownership

<If applicable: which service owns what, which crosses are allowed.>

---

## Schema sketch

### `<schema>.<table>`

<one-paragraph framing of what this table represents>

| Column | Type | Notes |
|---|---|---|
| `id` | `uuid` (v7) PK | ... |
| ... | ... | ... |

Constraints:
- ...

Indexes:
- ...

**Why <choice>.** <Rationale.>

### `<schema>.<other_table>`

...

---

## <Lifecycle> / <Sync> / <Re-ingestion> / <GC> / etc.

<One section per major operational concern. Use SQL/pseudocode where it
disambiguates. Use ✅/⚠️ trade-off lists when alternatives competed.>

---

## API surface (sketch — not normative)

| HTTP | Path | Notes |
|---|---|---|
| `POST` | `/...` | ... |

---

## Cross-service contract

<If applicable: how this design interacts with peer services. New helpers
required. Read-before-BEGIN / write-after-COMMIT discipline. Failure modes.>

---

## Privacy / Authorization / Safety

<For features with non-trivial access-control stories.>

---

## Open questions

1. **<Title>.** `[blocks-v1 | blocks-impl | deferred | exploratory]`
   <Question + state of debate + indicative direction or
   explicit "deferred until X.">
   Blocks: <named blockee — required for blocks-v1 / blocks-impl;
   omit for deferred / exploratory>.
2. ...

---

## Deferred (out of scope for this plan)

- <one-liner with reason>
- ...

---

## Known limitations (inherited)

<If applicable: limits the design accepts but didn't introduce, with pointer
to the upstream source of the limit.>

---

## Sequencing suggestion (not a sprint plan)

<If the plan covers a rollout: numbered steps with what each unblocks.
Explicitly not a sprint plan — call that out in the heading.>

---

## Decisions log (resolved)

- **<Decision lead>.** <Brief restatement.> (R<n>)
- ...

---

## Status

- **Round 1**: <summary>. _Next:_ <one-line recommended focus for next round, citing Open question IDs + tags>.
- **Round 2**: <summary>. _Next:_ <…>.
- ...

---

## See also

- <related plans / wiki / code paths>
```

---

## Quick reference: when in doubt

- **Add a "Why X"** if a reader could plausibly ask "why this shape?".
- **Move an Open question to Decisions log** the moment it resolves; never mark "(resolved)" in place.
- **Tag every Decisions-log bullet with `(R<n>)`** so the audit trail is intact.
- **Update Status every round**, even if the round was small.
- **Purge stale wording** when a round supersedes it; trust the Decisions/Status logs to carry the audit trail.
- **Compress older log entries** that no longer carry weight — superseded Decisions-log bullets keep their tag and supersession pointer but drop the rationale paragraph; ancient Status rounds whose `_Next:_` is long-finished collapse to a one-line summary. Keep the last 2–3 rounds and any still-load-bearing entry in full. See § 14 and § 15.
- **Prefer concrete over abstract**: an example JSON, an actual SQL fragment, a typed interface sketch — over prose paraphrase.
- **Surface alternatives, don't decide unilaterally** when the user has skin in the game.
