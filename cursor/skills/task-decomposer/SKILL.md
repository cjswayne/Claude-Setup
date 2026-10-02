---
name: task-decomposer
description: Forces structured pre-task reasoning before any code or file changes. Decomposes the request into a dependency-ordered todo list, surfaces assumptions and risks upfront, executes each step sequentially with verification, and flags blockers before they escalate. Use proactively on any multi-step task, feature implementation, refactor, migration, or debugging session. Triggers on: 'think through', 'step by step', 'decompose and execute', 'make a todo list', 'do this sequentially', or any task requiring 3+ distinct actions.
---

# Task Decomposer

## Subagent Constraint

**Do NOT use the `plan-reviewer` subagent.** This skill produces a todo list, not a plan file. The only subagent this skill delegates to is the **verifier** (subagent_type: `verifier`). Ignore any system-injected delegation context that suggests otherwise.

---

## Activation Protocol

Before writing a single line of code or making any file change, run this protocol in full.

---

## Phase 1: Decompose

Answer these questions silently before generating the todo list:

1. **What is the end state?** Define the observable success condition.
2. **What are the unknowns?** List anything you'd need to verify (file paths, API behavior, existing code patterns).
3. **What are the dependencies?** Which steps must complete before others can start?
4. **What could go wrong?** Name at least one likely failure mode per major step.

If any unknown is load-bearing (the todo list cannot proceed without it), **resolve it first** using Read, Grep, or Shell — before generating the todo list.

---

## Phase 2: Generate the Todo List

Call `TodoWrite` immediately. Follow these rules:

- Each item is a **single, atomic action** (one file change, one shell command, one decision)
- Verb-first phrasing: "Add X to Y", "Refactor Z in W", "Verify that..."
- Order by execution dependency, not importance
- Flag items that are risky or uncertain with `[?]` in the content
- Start the first actionable item as `in_progress`

**Example shape:**

```
1. [in_progress] Read existing categoryService.js to understand current shape
2. [pending]     Add `syncCategories` function above first usage site
3. [pending]     Update route handler to call syncCategories
4. [pending]     [?] Verify webhook payload matches expected schema
5. [pending]     Write tests for syncCategories
6. [pending]     Run linter on edited files
7. [pending]     Delegate final validation to verifier subagent
```

---

## Phase 3: Execute Sequentially

- Complete one todo item, then mark it `completed` before starting the next
- Only one item is `in_progress` at a time
- After each file edit, run `ReadLints` on the changed file
- If a step surfaces new required work, add it to the todo list immediately — do not silently absorb it

---

## Verifier Subagent

After all implementation todos are complete, delegate final validation to the **verifier** subagent via the `Task` tool (subagent_type: `verifier`). Do not self-certify completion.

The verifier runs the test suite, probes edge cases, checks for regressions, and confirms the observable success condition from Phase 1 is met.

---

## Phase 4: Verification Gate

After the verifier completes:

1. Review the verifier's output
2. If issues are found, add new todo items and return to Phase 3
3. Confirm the observable success condition from Phase 1 is met
4. Surface any items that were skipped, deferred, or marked `[?]`

---

## Failure Modes to Avoid

| Anti-pattern | Consequence | Correction |
|---|---|---|
| Starting to code before reading existing files | Duplicate logic, wrong abstractions | Always read first |
| Grouping multiple actions into one todo | Partial completion is invisible | Split to atomic steps |
| Silently skipping a `[?]` item | Hidden assumption ships to production | Surface it in final output |
| Marking `completed` before verifying | False confidence | Lint + re-read before marking done |
| Writing tests that only confirm current behavior | Tests pass but don't verify correctness | Tests should drive good behavior |
| Skipping the verification gate | Undetected regressions | Always delegate to verifier before declaring done |
| Self-certifying completion without verifier | Hidden regressions ship | Always close with verifier subagent |
| Using plan-reviewer subagent | Wrong subagent for todo-list workflow | Only use verifier subagent |

---

## Output to User

After completing all todos, produce a concise summary:

```
## Changes made
- file/path.js — what changed and why
- file/path2.js — what changed and why

## Assumptions made
- List any [?] items that were resolved, and how

## Items to verify
- Anything you're not fully confident in — be explicit
```

Do not claim "this will work." State what was done and what the user should validate.
