---
name: sequential-task-planner
description: Decompose a request into a sequential task plan and execute it end-to-end in a single turn. Identifies the problem, surfaces unknowns and assumptions, drafts an ordered task list, then implements each step with verification. Use when the user asks for an implementation, fix, refactor, migration, or any multi-step work that should be planned and completed within one turn rather than handed back to the user mid-flight.
---

# Sequential Task Planner

A disciplined four-phase loop the agent runs in one turn:
**1) Identify  →  2) Unknowns  →  3) Plan  →  4) Execute & Verify.**

Default to doubt. Treat the plan as a draft. Surface assumptions explicitly so the user can correct them at the end.

## When to use this skill

Apply when the request is non-trivial and meets any of:
- Touches 2+ files or systems
- Has implicit decisions (library choice, schema shape, error handling)
- Involves a fix where root cause is not yet confirmed
- Looks like "build / refactor / migrate / wire up / set up"

Skip for: trivial single-line edits, pure questions, or cases where Plan mode is more appropriate (large architectural decisions with significant trade-offs — call `SwitchMode` to `plan` instead).

## The four phases (run them in order, in one turn)

### Phase 1 — Identify the problem

Write a short block at the top of the response covering:
- **Restated goal**: one sentence in the user's terms
- **Scope**: what is in / out of scope
- **Success criteria**: how the user (and the agent) will know it works
- **Failure modes to watch**: 1–3 things most likely to go wrong

Keep this to ~5–8 lines. If the goal is genuinely ambiguous, ask one focused question via `AskQuestion` *before* continuing — do not guess at architectural intent.

### Phase 2 — Capture unknowns and assumptions

List two short blocks:

```
Assumptions (proceeding as if true):
- <assumption 1>
- <assumption 2>

Unknowns (will verify during execution):
- <unknown 1> — verification: <how it will be checked>
- <unknown 2> — verification: <how it will be checked>
```

Rules:
- Resolve unknowns with tools first (Read / Grep / Glob / SemanticSearch / Shell), not by asking the user, when the answer is in the codebase.
- Only escalate to `AskQuestion` if an unknown is a true blocker (a decision only the user can make).
- Flag any version-, API-, or environment-specific guess as an unknown — never present a guess as a fact.

### Phase 3 — Draft a sequential task plan

Use `TodoWrite` to record the plan. Each todo must be:
- **Concrete** (a verb + object, e.g., "Add `validateInput` to `web/services/x.js`")
- **Ordered** (later steps may depend on earlier ones)
- **Verifiable** (has an obvious done-condition)

Plan shape:
1. Read / locate (gather exact symbols, files, schemas)
2. Decide (lock in approach based on what was read)
3. Implement (smallest viable changes, file by file)
4. Lint / type-check (`ReadLints` on touched files)
5. Verify (run tests, scripts, or a focused manual check)
6. Summarize (what changed, what is still uncertain)

Mark the first todo `in_progress` in the same batch as the `TodoWrite` call.

### Phase 4 — Execute & verify in the same turn

Execution rules:
- **Do not stop** between todos to wait for the user. Complete the whole plan unless a true blocker appears.
- **Mark done immediately** after each step finishes — only one todo `in_progress` at a time.
- **Batch independent reads / searches** in a single tool-call message; sequence only true dependencies.
- **Re-plan in place** if a step reveals the plan was wrong. Update the todo list with `merge: true` and continue — do not silently abandon steps.
- **Always run `ReadLints`** on files you edited before declaring done.
- **Log caught errors** — never swallow with `catch (_) {}` (per project rule).

Blocker policy: if a step uncovers a decision only the user can make, stop, set the affected todo back to `pending`, and use `AskQuestion` with concrete options. Do not invent the answer.

## Final summary template

End the turn with:

```
Summary
- What was done: <files / behaviors changed>
- How to verify: <commands, URLs, or manual steps>
- Still uncertain: <items the user should double-check>
- Suggested next steps: <follow-ups, if any>
```

Keep tone cautious. Phrase verification as "this may work, please confirm against your environment" rather than "this works".

## Anti-patterns

- Jumping into edits before Phases 1–2 are written down
- Using `TodoWrite` only as decoration, then ignoring the order
- Asking the user a question that could have been answered with a Read or Grep
- Declaring done without running `ReadLints` or any verification step
- Hiding assumptions inside code comments instead of stating them in the response
- Stopping mid-plan to "check in" when no actual blocker exists

## Minimal worked example (shape only)

> User: "Add a retry wrapper around `fetchProducts` in the webhook handler."

```
Identified problem
- Goal: wrap fetchProducts with bounded retry on transient failures
- Scope: only fetchProducts call sites in productUpdate handler; not other handlers
- Success: transient 5xx/network errors retry up to N times with backoff; permanent errors propagate
- Watch: double-invocation side effects, log noise, infinite retry on 4xx

Assumptions
- fetchProducts is idempotent for the read path
- Existing logger is acceptable for retry telemetry

Unknowns
- Exact error shape thrown by fetchProducts — verify by reading the function
- Whether a retry util already exists — verify by Grep "retry"
```

Then `TodoWrite`:
1. Read `handleProductUpdate.js` and `fetchProducts` definition
2. Grep for existing retry helper
3. Decide: reuse helper or add a small local one
4. Implement wrapper + wire in call sites
5. `ReadLints` on touched files
6. Summarize and flag uncertainties

Execute all six in the same turn, updating todo status as you go.

## Notes and caveats

- This skill encodes a workflow, not a guarantee. The agent (and the user) should still treat output as a draft to review.
- For large architectural work, prefer `SwitchMode` → `plan` first; this skill is for "I know roughly what to build, now build it carefully".
- If the project has a `/test-runner` subagent rule (it does in Datify), delegate the verification step there instead of running tests inline.
