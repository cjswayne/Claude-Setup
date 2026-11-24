---
description: "Delegate requested changes to a subagent, choosing the model by task complexity"
---

# Delegate Requested Changes to a Subagent

When Charles is talking to you and asks for a change (code, config, docs, data fixes, investigations that lead to a fix), do not make it in the main session. Delegate it to a subagent with the Agent tool, and pick the model by how complex the task is.

## Model by complexity

| Task | Model |
|------|-------|
| Trivial: one-line edit, rename, copy text, flip a default | `haiku` |
| Typical: a feature slice, a bug fix across a few files, tests, a PR | `sonnet` |
| Hard: unclear root cause, cross-cutting design, risky migrations, multi-phase plans | `opus` |

This replaces the blanket "always sonnet" default in `subagent-model.md` for these delegations. If Charles names a model, use that.

## Required

1. Decide the scope first, then write a self-contained brief: absolute paths, what to change, what not to touch, the approvals Charles has given (merge, deploy, spend) and the gates to run.
2. Name the MCP tools the subagent should use, or say none apply (see the global Subagents instructions).
3. Keep the main session for talking with Charles: relay results, decisions and blockers. Review the subagent's diff or report before saying the work is done.
4. Read-only lookups that answer a question (one grep, one status check) can stay in the main session.

## Example

```text
Charles: make the Notion task default on

BAD  — edit shared/src/schemas/site-performance.ts in the main session
GOOD — Agent(subagent_type="executor", model="haiku", prompt="In <abs path> change DEFAULT_... enabled to true, update the tests that assert off, run <gates>, report the diff.")
```
