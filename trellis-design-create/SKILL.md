---
name: trellis-design-create
description: Create a new Design Plan [$0=output path]
disable-model-invocation: true
---

This skill is invoked when a user wants to make
a new Design Plan.

Before doing anything, read the [Design Plan Documents — Authoring Guide](design-plan.md)

If the user has not provided any other information (other than the invoking this skill): stop and tell the user you are ready for their initial briefing for round 1 of design doc iteration.

If the user has provided additional detail along with this skill invocation: you can use that as input to start round 1.

After round 1 is complete, you should continue iterating with the user in accordance to the design plan authoring guide.
