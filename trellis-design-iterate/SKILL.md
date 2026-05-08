---
name: trellis-design-iterate
description: Iterate on a Design Plan [$0=path-to-plan-file]
disable-model-invocation: true
---

This skill is invoked when a user wants to iterate on
a Design Plan.

Before doing anything, read the [Design Plan Documents — Authoring Guide](../trellis-design-create/design-plan.md)

Then read the existing design plan file at $0.

If the user has not provided any other information (other than the invoking this skill): stop and tell the user you are ready to here their input for the next round of design doc iteration.

If the user has provided additional detail along with this skill invocation: you can use that as input to start the next round of iteration.

After incorporating the user's comments, you should continue iterating with the user in accordance to the design plan authoring guide.
