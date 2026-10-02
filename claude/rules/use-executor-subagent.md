---
description: "Delegate any multi-line code edit to the executor subagent instead of editing directly"
---

# Delegate Multi-Line Edits to the Executor

If a code change touches **more than one line**, dispatch
`Agent(subagent_type: "executor")` to make the edit. Do not call `Edit` /
`Write` yourself for that change.

## Required

1. Decide the change fully first (files, exact behavior, edge cases).
2. Give the executor **absolute file paths** and the complete intended code or a
   precise description of every edit — it cannot see this conversation.
3. Review the executor's diff before reporting the work as done.

## Direct edits still allowed

- Single-line changes (a value, a label, one condition, one import)
- Creating a brand-new file whose full contents you are authoring
- Applying a fix the executor got wrong, when re-delegating costs more than it saves
- The user explicitly says to edit directly / not to use subagents

## Examples

```text
# Bad — parent agent edits 40 lines across two files itself
Edit backend/src/lib/chat-tools.ts (multi-line block)
Edit backend/test/chat-tools.test.ts (multi-line block)

# Good — parent delegates
Agent(subagent_type: "executor", prompt: "In C:/.../backend/src/lib/chat-tools.ts,
replace the buildArgs function with: <full code>. Then in
C:/.../backend/test/chat-tools.test.ts add a case asserting <behavior>.")
```

## Self-check before an edit tool call

If `new_string` spans more than one line, stop and delegate instead.
