# Trellis Instruction Precedence

Single canonical statement of the **Trellis instruction precedence chain**. Every Trellis operation applies
this chain before doing its work. The instruction files (`instructions/*.md`), the subagent briefs
(`subagents/*.md`), and `SKILL.md` § "Apply the trellis instruction precedence chain" all point back here.
Change the rule here and nowhere else.

## The chain

Before executing the operation, read and apply Trellis instructions from these sources, in **ascending order of
precedence** (later overrides earlier):

1. **`~/.trellis/instructions.md`** — applies to every Trellis invocation, in every project.
2. **`<repo-root>/.trellis/instructions.md`** — applies to every Trellis invocation in the current repository.
   `<repo-root>` is the root of the current Git repository.
3. **Invocation-specific instructions** — the additional instructions supplied with this invocation. For a
   top-level operation these are the inline instructions the user typed; for a subagent they are the additional
   user instructions passed down by whoever dispatched it (the orchestrator, the executor, or the dispatcher).
   Highest precedence.

Missing instruction files are normal; skip them silently. If instructions conflict, later sources override
earlier sources. Invocation-specific instructions apply only to the current run.

## Limits

These instructions may override the Trellis instruction file or subagent brief that points here, but they
**must not** override system, developer, tool, safety, or repository policy instructions.
