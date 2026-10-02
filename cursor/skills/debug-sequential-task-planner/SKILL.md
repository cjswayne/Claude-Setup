---
name: debug-sequential-task-planner
description: Forces structured pre-debug reasoning before any code fix. Decomposes the reported issue into a dependency-ordered todo list, requires error snippets and stack traces in the chat, isolates root cause before patching, executes fixes sequentially with verification, and flags remaining risks. Use proactively on any debugging session, error investigation, failing test, runtime exception, build failure, or unexpected behavior. Triggers on 'debug this', 'fix this error', 'why is this failing', 'step through the bug', 'trace the issue', 'root cause', or any task requiring diagnosis before repair.
---

# Debug Sequential Task Planner

## Activation Protocol

Before changing any code to fix an issue, run this protocol in full. Debugging without structure leads to whack-a-mole fixes that mask root causes.

---

## Phase 0: Collect Evidence

**This phase is mandatory and blocks all subsequent phases.**

1. **Require error snippets in the chat.** The user MUST paste the actual error output, stack trace, log excerpt, or screenshot description into the conversation. Do not guess at errors from description alone. If the error evidence is missing, stop and ask:
   > "Please paste the full error message, stack trace, or relevant log output into the chat so I can diagnose accurately."

2. **Require reproduction context.** Confirm or ask for:
   - The command or action that triggers the error
   - The environment (Node version, OS, dev vs prod, etc.) if relevant
   - Whether the error is consistent or intermittent

3. **Pin the evidence.** Restate the exact error in a quoted block so it stays visible as the conversation grows. Example:
   > ```
   > TypeError: Cannot read properties of undefined (reading 'id')
   >     at getUser (src/services/userService.js:42:18)
   >     at async handler (src/routes/users.js:15:20)
   > ```

Do not proceed past Phase 0 until at least one concrete error snippet is present in the chat.

---

## Phase 1: Decompose the Bug

Answer these questions silently before generating the todo list:

1. **What is the symptom?** Restate the observable failure in one sentence.
2. **What is the expected behavior?** Define what "fixed" looks like.
3. **Where does the error originate?** Identify the file and line from the stack trace or error output.
4. **What are the candidate root causes?** List at least two hypotheses — avoid anchoring on the first guess.
5. **What do I need to read first?** Identify every file referenced in the stack trace or error message.
6. **What could go wrong with the fix?** Name at least one regression risk per hypothesis.

If any unknown is load-bearing (the diagnosis cannot proceed without it), **resolve it first** using Read, Grep, or Shell — before generating the todo list.

---

## Phase 2: Generate the Debug Todo List

Call `TodoWrite` immediately. Follow these rules:

- Each item is a **single, atomic action** (one file read, one hypothesis test, one code change)
- Verb-first phrasing: "Read X to check Y", "Add guard for Z", "Verify fix by running W"
- Order: **read → hypothesize → isolate → fix → verify** (never fix before reading)
- Flag uncertain items with `[?]` in the content
- Start the first actionable item as `in_progress`

**Example shape:**

```
1. [in_progress] Read src/services/userService.js:35-50 to inspect getUser
2. [pending]     Read src/routes/users.js:10-20 to trace how getUser is called
3. [pending]     [?] Check if the upstream query returns null when user is missing
4. [pending]     Add null guard in getUser before accessing .id
5. [pending]     Run linter on edited files
6. [pending]     Delegate regression test to test-runner subagent
7. [pending]     Delegate final validation to verifier subagent
```

Note which steps belong to subagents so the reviewer and executor know where delegation happens.

---

## Available Subagents

Subagents live in `C:\Users\cjswa\.cursor\agents\`. Delegate to them via the `Task` tool rather than handling every concern in the parent agent.

| Subagent | When to Use in Debugging |
|---|---|
| **debugger** | Root cause is unclear after initial reads. Reproduces, isolates, and proposes minimal fixes. **Prefer this over inline guessing.** |
| **executor** | Fix is fully defined and unambiguous — apply the patch mechanically. |
| **test-runner** | Write or audit regression tests after the fix is applied. Never write tests inline. |
| **verifier** | Final validation after all todos are done. Runs the suite, probes edge cases. |
| **plan-reviewer** | Audit the debug plan if it has 5+ steps or touches multiple files. |

### Delegation Rules

- **Delegate to `debugger` immediately** when a hypothesis fails or the root cause is not obvious after reading the relevant code.
- **Always delegate regression test writing to `test-runner`.** Do not write tests inline in the parent agent.
- **Always delegate final validation to `verifier`.** Do not self-certify the fix.
- **Delegate `executor` only when the fix instructions are unambiguous.** If ambiguity remains, resolve it in the parent first.
- **Delegate `plan-reviewer` before Phase 3** on any plan with 5+ steps or cross-file changes.

---

## Phase 3: Execute Sequentially

- Complete one todo item, then mark it `completed` before starting the next
- Only one item is `in_progress` at a time
- **Read before writing.** Every file you plan to edit must be read first in this session
- After each file edit, run `ReadLints` on the changed file
- If a step surfaces a new error or secondary issue, add it to the todo list immediately — do not silently absorb it
- If a fix attempt introduces a new error, **stop and re-enter Phase 1** for the new error (paste the new snippet)
- When a step maps to a subagent, delegate via `Task` tool — do not inline the work

### Fix Principles

- **Minimal diff.** Change only what is necessary to resolve the root cause. Resist the urge to refactor adjacent code.
- **Guard, don't suppress.** Prefer null checks and input validation over try/catch that swallows errors silently.
- **Preserve existing behavior.** The fix should change the broken path, not the happy path.
- **Log caught errors.** Never use `catch (_) {}` — always log the error.

---

## Phase 4: Verification Gate

After the final todo item completes:

1. Re-read every file you changed
2. Check lints on all changed files
3. Delegate to **`verifier`** subagent to run the test suite and probe edge cases
4. Confirm the observable success condition from Phase 1 is met
5. Confirm the original error snippet **no longer reproduces**
6. Surface any items that were skipped, deferred, or marked `[?]`

---

## Failure Modes to Avoid

| Anti-pattern | Consequence | Correction |
|---|---|---|
| Fixing before reading the stack trace | Wrong file, wrong function, wasted effort | Always collect evidence in Phase 0 |
| Guessing the error instead of requiring the snippet | Misdiagnosis, phantom fixes | Demand the actual error text in the chat |
| Anchoring on the first hypothesis | Confirmation bias, incomplete fix | List at least two candidate causes |
| Fixing the symptom instead of the root cause | Bug reappears under different conditions | Trace the call chain to the origin |
| Editing multiple files at once | Can't isolate which change fixed it | One file change per todo item |
| Silently absorbing a secondary error | New bug ships with the fix | Add it to the todo list, re-enter Phase 1 if needed |
| Writing tests in the parent agent | Tests optimized to pass, not verify | Delegate all test work to `test-runner` |
| Self-certifying the fix without `verifier` | Undetected regressions | Always close with `verifier` subagent |
| Using `catch (_) {}` anywhere | Errors silently swallowed | Always log caught errors |

---

## Output to User

After completing all todos, produce a concise summary:

```
## Bug fixed
- **Error:** <one-line restatement of the original error>
- **Root cause:** <what was actually wrong and why>
- **Fix:** <what was changed>

## Files changed
- file/path.js — what changed and why

## Assumptions made
- List any [?] items that were resolved, and how

## Regression risks
- Anything that could break as a side effect — be explicit

## Items to verify
- Anything you're not fully confident in
```

Do not claim "this will work." State what was done and what the user should validate.
