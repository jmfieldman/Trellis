# Implementation Plan — Skeleton Templates

Companion to [implementation-plan.md](./implementation-plan.md) (document anatomy and authoring conventions). Skeleton templates for every implementation-plan file. Loaded by impl-create at scaffold time, and by an impl-iterate round only when it creates a new sprint file mid-plan. Drop sections that don't apply; adapt the content to the target project per the authoring guide's "Architecture is inherited, not prescribed" rule.

---


## Skeleton templates

### `overview.md`

```markdown
# <Feature> — Implementation Plan

> Source design: [<design plan>](<path/to/design-plan.md>). Read end-to-end before picking up Sprint 01.

This plan breaks <feature> into <N> PR-sized sprints. Each sprint ships a coherent, independently-testable deliverable. Earlier sprints establish foundations; later sprints layer …

## Philosophy

### The plan is the source of truth
…

### Tier-0-aware, but never tier-0-locked
…

### Cross-service contract over internal cleverness
…

### Test the cross-service seams hardest
…

### Small, shippable sprints
…

## Architectural invariants the sprints uphold

1. <invariant — restated from the design plan; tests must verify when touched>
2. …

## Module / directory layout impact

```
src/services/<svc>/
├── …
```

## Cross-sprint conventions

- **Branching:** …
- **Migrations:** …
- **Seed data:** …
- **OpenAPI / spec sync:** …
- **Tests:** …
- **Logging:** …
- **Errors:** …
- **Idempotency:** …
- **Feature flags:** …

## Sprint organization

### Rationale for the sprint order

- **Sprints 01–02 — foundation.** …
- **Sprint 03 — …** …
- …

### Rationale for sprint-sized slices

A sprint is scoped to roughly one to two weeks of focused work by a single engineer. Each:
- Targets one architectural seam or one user-visible capability.
- Has a clear "done" line with acceptance criteria.
- Leaves the repo buildable, testable, and demonstrable.
- Can be reviewed as a single PR.

The slices are not equal in size. They are equal in shippability.

## Sprint roster

| #  | Sprint                                       | Outcome                                                  |
| -- | -------------------------------------------- | -------------------------------------------------------- |
| 01 | [<title>](./01-<topic>.md)                   | …                                                        |
| 02 | [<title>](./02-<topic>.md)                   | …                                                        |
| …  | …                                            | …                                                        |

## Dependency graph

```
01 ──► 02 ──► 03 ──► …
```

- **01 → 02**: <why>
- **02 → 03**: <why>
- …

## Feature-wide locked decisions

| Decision   | Value |
|------------|-------|
| …          | …     |

## Out-of-scope across all sprints

- …

## Open questions

1. **<title>.** <question + state of debate + indicative direction or "deferred until X.">
2. …

<!--
  No Decisions log or Status section here.
  Plan-level decisions live in ./decisions.md.
  Plan-level round-by-round audit trail lives in ./status.md.
-->

## What this plan does *not* try to do

- …

## How to read each sprint

Every sprint follows the same skeleton: Goal → Prerequisites → Deliverables → Out of scope → Locked decisions → Architecture notes → Public surface (when applicable) → Progress → Implementation Steps with verification → Step Dependency Chart → Acceptance checklist. Implementers should be able to pick up a sprint and execute it without re-reading the design doc, though the design doc is the source of truth for any conflict.
```

### `NN-<topic>.md`

```markdown
# Sprint NN — <Short title>

> Part of the broader [<Feature> Implementation Plan](./overview.md).

## Goal

<one sentence — outcome, not activity>

## Prerequisites

- <prior sprint or peer-service surface>
- …

## Deliverables at end of sprint

- <observable artifact>
- …

## Out of scope

- <forward-link to the sprint that owns it>
- …

## Key design decisions, locked before implementation

| Decision                        | Value |
|---------------------------------|-------|
| …                               | …     |

## Architecture notes

- **Why <choice>.** …
- **Why <other choice>.** …

## Public surface

### `<METHOD> <path>`

#### Request

```jsonc
{ … }
```

#### Response <code>

```jsonc
{ … }
```

#### Errors

| Status | Code | Cause |
|---|---|---|
| … | … | … |

---

## Progress

- [ ] Step 1 — …
- [ ] Step 2 — …
- …

---

# Implementation Steps

## Step 1 — <title>

**Goal**: <one sentence>

**Build/test boundary**: <omit unless this step may temporarily leave the repo uncompilable / untestable; if needed, explain why unavoidable and which later step restores the gates>

**Actions**:

1. <concrete action with file paths and code samples>
2. …
3. **Subtle bug**: <gotcha spelled out>

**Deliverables**:
- …

**Verification**:
- …

## Step 2 — <title>

…

---

## Step Dependency Chart

```
1 ──► 2 ──► …
```

- <one-liner per parallelizable / hard-sequenced edge>

## Acceptance checklist for the sprint as a whole

- [ ] …
- [ ] …
```

### `progress.md`

```markdown
# <Feature> — Progress

> Part of the [<Feature> Implementation Plan](./overview.md).

## Sprint 01 — <title>

- [ ] Step 1 — …
- [ ] Step 2 — …
- …

## Sprint 02 — <title>

- [ ] Step 1 — …
- …
```

### `decisions.md`

```markdown
# <Feature> — Decisions

> Part of the [<Feature> Implementation Plan](./overview.md).
>
> Cross-cutting decisions only. Sprint-scoped decisions live in the relevant sprint file's Decisions log section.

- **<Decision lead in bold>.** <Brief restatement / rationale.> (R<n>)
- **<Decision lead in bold>.** <…> (R<n>)
- …
```

### `status.md`

```markdown
# <Feature> — Status

> Part of the [<Feature> Implementation Plan](./overview.md).
>
> Plan-level round-by-round audit trail. Sprint-scoped Status / Feedback-incorporated entries live in the relevant sprint file when a round materially changes a single sprint.

- **Round 1**: scaffolding + sprint roster + open questions enumerated. _Next:_ <one-line recommended focus, citing Open question IDs + tags + scope where relevant>.
- **Round 2**: … _Next:_ <…>.
- …
```

### `post-mortem.md`

Created lazily — the first sprint to ship creates this file in the same PR that checks off its final step. Each subsequent shipped sprint appends a new section.

```markdown
# <Feature> — Post Mortems

> Part of the [<Feature> Implementation Plan](./overview.md).

## Sprint 01 — <title>

- <distilled observation, lesson, or follow-up>
- <distilled observation, lesson, or follow-up>
- …

## Sprint 02 — <title>

- _No post-mortem-worthy observations._
```
