---
name: impl-review
description: Review an Implementation Plan (dispatches a fresh-context reviewer subagent)
argument-hint: <root> [<review-output-path>]
disable-model-invocation: true
---

# Implementation Plan Review — Orchestrator Skill

You are the **orchestrator** for an external review of an implementation plan — `overview.md`, `decisions.md`, `status.md`, every sprint file, `progress.md`, and (if present) `post-mortem.md` in a feature root. Activated when the user wants an agent-based read on the whole plan.

## Hard invariant — read this first

**You do not perform the review yourself. The review is dispatched to a fresh-context subagent via the `Agent` tool.** This conversation may contain the authoring context — iterate rounds, locked decisions argued live, the user's reasoning. A review produced inside it is not a fresh read: the reviewer's value is auditing the plan *without* the author's reasoning loaded, and self-review silently collapses that distance (the same audit property [nested-dispatch.md](../specs/nested-dispatch.md) protects in the execution and resolve flows). Even in a brand-new conversation, dispatch — it keeps the main chat small and the audit uniform.

The reviewer's brief lives at [`../subagents/impl-plan-reviewer.md`](../subagents/impl-plan-reviewer.md). You pass the subagent the absolute path to that brief plus the per-invocation parameters; the subagent reads the brief itself. Do **not** copy the brief into the spawn prompt, and do not read it end-to-end yourself — you only need to know it exists.

## Inputs

Extract from the user's natural-language invocation:

1. **`<root>`** — the feature root. Resolve as follows:
   - If the user named a feature root (e.g., "review the impl plan at `docs/refunds/`"), use that path as `<root>`.
   - If the user named `design.md`, `overview.md`, `progress.md`, `decisions.md`, `status.md`, or a sprint file, use its parent directory as `<root>`.
   - If neither holds, ask the user and stop.
2. **`<review-output-path>`** (optional) — where the review document lands. If missing, default to `<root>/impl-review-R<N>.md` (where `<N>` is the plan's current round number from `status.md`) — don't force a round-trip with the user just to name the output file. Tell the user where you wrote it.
3. **Additional instructions** (optional) — free text supplied with the invocation. Forward them to the reviewer verbatim.
4. **Chain request** (optional) — if the user asked to integrate the findings in the same invocation ("review the impl plan *and integrate the findings*"), note it; see § "Chaining" below.

## Pre-flight checks

Run these before dispatching. Stop and report on any failure.

1. **`<root>/overview.md` exists.** If `<root>` has `design.md` but no `overview.md`, the implementation plan hasn't been scaffolded — tell the user this is an `impl-create` job and stop.
2. **The plan has at least one round.** `status.md` must exist with at least a Round 1 entry; a rootful of empty stubs isn't reviewable — report it and stop.
3. **The output path is safe to write.** If the user explicitly named a path that already exists, warn and stop for confirmation before overwriting. If the *default* path already exists (the round was reviewed before), don't overwrite it — use `<root>/impl-review-R<N>-2.md` (then `-3`, …) and say so.

## Dispatch the reviewer

Use the `Agent` tool with `subagent_type=general-purpose`. The spawn prompt is short and self-contained:

```
You are reviewing a trellis implementation plan (the whole plan, holistically).

Read your brief at:
  <absolute-path-to-trellis-dir>/subagents/impl-plan-reviewer.md

That brief tells you exactly what to do. Read it end to end before doing anything else.
It also tells you to load Trellis instruction files from `~/.trellis/instructions.md` and `<repo-root>/.trellis/instructions.md` when present; do that before reviewing.

Parameters:
- Feature root: <absolute path to <root>>
- Review output path: <absolute review-output-path>

Additional user instructions (forwarded verbatim; override the brief on conflict):
<paste any free-text instructions the user attached, or "(none)">

Write the review document to the review output path per your brief. Your final message back to me must contain: one line confirming the review was written (with the path and a one-line overall read), followed by the Open questions disposition summary your brief specifies. Do not paste the full review into the message.
```

Resolve `<absolute-path-to-trellis-dir>` from wherever the top-level trellis `SKILL.md` was loaded — do not assume a fixed install location. Do not run the reviewer in the background — its result gates the rest of this operation.

**Fallback — no subagent dispatch at all.** If your harness exposes no subagent dispatch (verified per the discovery rule in [nested-dispatch.md](../specs/nested-dispatch.md) — attempt the spawn first; never assume from one tool index), perform the review yourself by following the brief end-to-end, and say explicitly in your hand-off that the fresh-context property is degraded because this conversation may contain authoring context.

## Validate and relay

When the reviewer returns:

1. **Confirm the review file exists** at the output path and is a real review (starts with a `# Review:` heading, has an Executive summary). If the file is missing or empty, report the failure — do not fabricate a review or retry silently.
2. **Relay the reviewer's hand-back to the user**: where the review landed, the one-line overall read, and the Open questions disposition summary verbatim. The user should get the disposition summary without opening the file.
3. Recommend the next move: normally *"integrate the review via impl-integrate-feedback"* (or note that you're continuing into it, if chained).

## Chaining — review → integrate in one invocation

`impl-review → impl-integrate-feedback` is a **sanctioned chain** (see SKILL.md § "Sanctioned operation chains"). If — and only if — the user asked for it in this invocation ("review and integrate", "review the plan and apply the findings"), continue after the relay step:

1. Load [`impl-integrate-feedback.md`](./impl-integrate-feedback.md) and run it end-to-end with `<feedback-path>` = the review you just produced and `<root>` unchanged.
2. All of that skill's gates still apply — in particular, a Reviewer-wrong rate above ~20% stops the chain there, exactly as it would stop a standalone integration.
3. The fresh-context audit holds across the chain because the review came from the dispatched subagent, not from this context.

The chain never extends further: "review and integrate" does not continue into a fresh iterate round, a re-slice, or impl-execute. Without an explicit chain request, stop after the relay step.

## What this operation does not do

- It does not edit the plan. Only the chained integrate step (when explicitly requested) edits anything.
- It does not read the plan bodies into orchestrator context beyond the pre-flight checks (file existence, `status.md` round count). The reviewer reads the plan; you don't need to.
- It does not review the plan inline when dispatch is available — see the hard invariant.
- It does not auto-chain. Integration runs only on an explicit ask.

---

## Additional Instructions

The **Trellis instruction precedence chain** (`~/.trellis/instructions.md` → `<repo-root>/.trellis/instructions.md` → instructions supplied with this invocation) is applied by `SKILL.md` before this file is dispatched, so it is already in effect; honor it. Canonical statement — the single source shared by every instruction file and subagent brief: [instruction-precedence.md](../specs/instruction-precedence.md).
