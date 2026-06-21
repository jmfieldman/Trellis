# Trellis Nested-Dispatch Discovery & Fallback

Single canonical statement of how a Trellis subagent discovers whether it can spawn a **nested** subagent, and what to do when it can't. Every Trellis flow in which one subagent spawns a second — the step-executor spawning its reviewer, the question-resolver spawning its reviewer — applies this rule. The subagent briefs (`subagents/*.md`) and the orchestrator instruction files (`instructions/impl-execute.md`, `instructions/resolve-open-questions.md`) point back here. Change the rule here and nowhere else.

## Why nested dispatch exists

The audit property the whole pattern protects: **a fresh-context reader/critic inspects the work product without the producer's reasoning loaded.** A step's diff is reviewed by an agent that never saw the implementation reasoning; a resolver's proposal is critiqued by an agent that never saw the investigation. Self-review collapses that distance and silently degrades the audit, so the reviewer/critic must run in a separate context.

## The discovery rule

Before concluding you cannot dispatch a nested subagent:

1. **Attempt the spawn first.** Do not infer the dispatch tool is unavailable from its absence in one tool list. Some harnesses surface their tools across multiple indexes — primary tools vs. deferred / on-demand tools — and the dispatch primitive may live in a secondary index.
2. **Use any discovery mechanism the harness exposes.** If a tool-search / on-demand-tool primitive exists, use it to locate the dispatch tool before giving up.

A "no nested dispatch" conclusion is only valid once you have actually attempted the spawn and inspected whatever discovery mechanism exists — never assumed from a single index.

## The two failure shapes

Once you have genuinely attempted the spawn, a failure is one of two shapes:

- **A concrete blocker** — the dispatch primitive errors when invoked, the brief path is unreachable, or any other failure arises mid-spawn. This is a real failure, not a missing capability.
- **No nested dispatch at all** — the harness simply does not expose subagent dispatch (verified per the discovery rule above).

In the **no-nested-dispatch** shape the remedy is always the **orchestrator-dispatched fallback**: hand back to the orchestrator with the agreed `not_run` signal and let it dispatch the second subagent. How a flow maps the *concrete-blocker* shape is flow-specific and stated in each brief — e.g., the step-executor surfaces a concrete blocker as `status: stopped` (it has already committed work, so a mid-spawn failure is worth surfacing), whereas the question-resolver returns `reviewer_verdict: not_run` for both shapes (it edited nothing and is cheap to re-dispatch).

## Why the fallback preserves the audit

Under the fallback the **orchestrator** dispatches the reviewer/critic from a context that has not seen the producer's reasoning. The fresh-context property holds either way — nested or orchestrator-dispatched — so the audit is preserved. The fallback changes only *who dispatches*; every other invariant (clean working tree, real commits, durable records, no self-review) still applies.
