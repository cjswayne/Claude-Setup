---
name: generate-plan
description: Generates detailed implementation plans for coding tasks. Used when the user asks to plan a feature, create a plan, design an implementation approach, or when a task is complex enough to warrant structured planning before coding. Resolves ambiguity with the user, researches unknowns via MCP and codebase exploration, extracts existing patterns, authors the plan (including when to use /executor, /test-runner, and /verifier during execution and when those may run concurrently), runs /plan-reviewer on the first draft, and audits via the plan-reviewer subagent.
---

# Generate Plan

## Overview

This skill produces implementation plans in the project's standard format (YAML frontmatter + markdown body, saved to `.cursor/plans/`). Before writing the plan, it resolves ambiguity with the user, researches unknowns, and extracts existing codebase patterns that new code must match. Each plan document must spell out **when** `/executor`, `/test-runner`, and `/verifier` should be used during plan execution, and **when those delegations may run concurrently** (independent file areas, disjoint test suites, parallel validation where ordering does not matter). As soon as the **first draft** of the plan file exists on disk, run `/plan-reviewer` (delegate to the `plan-reviewer` subagent) to audit it; do not wait for polish passes before the initial review unless you are only fixing obvious typos before save.

## Workflow

### Phase 1: Disambiguate

Before any planning, identify gaps in the user's request. Collect **all** questions into a single batch using `AskQuestion` (or conversationally if unavailable).

**What to look for:**

- Multiple valid interpretations of the request
- Missing business logic or domain rules
- Unspecified edge cases (empty inputs, partial failures, concurrency, rate limits)
- Unknown acceptance criteria ("what does done look like?")
- Destructive or irreversible operations that need confirmation
- Priority or sequencing conflicts between steps
- Environment, configuration, or API version assumptions

**How to ask:**

- Group questions as **Blocking** (plan cannot proceed) or **Advisory** (plan proceeds with stated assumption).
- For each question, explain why you are asking and offer your best-guess default.
- Ask everything in one batch -- do not drip-feed questions across multiple turns.

**Example:**

> **Questions before I start planning:**
>
> 1. **(Blocking)** You mentioned "sync products." Should this be a one-time migration or a recurring scheduled job? *Default assumption: one-time.*
> 2. **(Blocking)** Should archived/draft products be included? *Default assumption: exclude archived, include draft.*
> 3. **(Advisory)** I'll assume exponential backoff with 3 retries for Shopify API rate limits unless you prefer a different strategy.

Do not proceed to Phase 2 until all blocking questions are answered.

### Phase 2: Research Unknowns

For anything the plan depends on that you are not confident about, research it **before** writing the plan. Follow this priority order:

1. **Codebase exploration** -- Use `SemanticSearch`, `Grep`, `Glob`, and `Read` to find existing patterns, models, services, and conventions in the project. Always ground the plan in how the codebase already works.
2. **Shopify MCP tools** -- For any Shopify-related work:
   - `introspect_admin_schema` to verify GraphQL field names, types, and API version
   - `search_dev_docs` or `fetch_docs_by_path` to confirm current API behavior, webhooks, or Polaris components
3. **Web search** -- Fallback for non-Shopify unknowns (library APIs, third-party services, general patterns).

**Rules:**

- Never assume Shopify API structure from memory. Always verify via MCP.
- Never assume a file path, model field, or function signature exists. Read the file first.
- If research reveals a constraint or limitation, note it in the plan's "Caveats" section.
- Use `sequentialthinking` from `user-sequential-thinking` MCP for complex reasoning during research.

### Phase 2b: Extract Existing Patterns (mandatory)

Before writing the plan, identify the **analogous existing code** in the codebase that new code must be consistent with. For each area the task touches (billing, routing, data fetching, API calls, etc.):

1. **Find the closest existing implementation.** Read the full function or flow, not just signatures.
2. **Extract the concrete patterns** new code must match. Write them down explicitly as a dedicated section in the plan called `## Existing Patterns (extracted)`. Examples:
   - "Return URLs for billing approvals route through a `/return` handler that syncs state, not directly to the app page" (from `billing.js` lines 120-145)
   - "API routes are mounted in `routes/index.js`, not `web/index.js`" (from the existing router at `routes/index.js` line 27-46)
   - "Shopify Admin API is queried as source of truth for billing status; MongoDB is a cache only" (from `billing.js` charge verification flow)
   - "GraphQL mutations must declare all forwarded variables including `$test: Boolean`" (from existing `APP_PURCHASE_ONE_TIME_CREATE` mutation)
3. **Include file paths and line numbers** for each extracted pattern so executors can verify them.
4. **Paste extracted patterns verbatim into executor Task prompts** during plan execution. Do not tell an executor "follow the pattern in billing.js" — paste the actual code snippet and say "match this exactly." Executors that must discover patterns themselves will get them wrong.

**Why this matters:** Plans describe *what* to build. Extracted patterns describe *how the codebase already does it*. Without the latter, executors produce code that is architecturally correct but functionally broken — wrong return URLs, wrong file mounts, wrong source of truth, missing GraphQL variables.

### Phase 3: Write the Plan

Save the plan to `.cursor/plans/` using the project's standard format. Generate a slug from the plan name with a random hex suffix (e.g., `feature_name_a1b2c3d4.plan.md`).

**Plan format:** Match existing plans for YAML frontmatter and core sections (`Architecture`, `Data Flow`, files, decisions, caveats). **Always** include `## Existing Patterns (extracted)` and `## Execution: Subagent dispatch` in new plans.

```markdown
---
name: Human-readable Plan Name
overview: 1-3 sentence summary of what the plan accomplishes and the high-level approach.
todos:
  - id: read-and-understand-plan
    content: Read the entire plan from top to bottom and confirm full understanding before any code changes are made
    status: pending
  - id: short-kebab-id
    content: One-line description of what this step accomplishes and which file(s) it touches
    status: pending
  - id: next-step
    content: Next step description
    status: pending
isProject: false
---

# Plan Title

> **Execution requirement:** Read and understand this entire plan before making any code changes. Any subagents listed in `## Execution: Subagent dispatch` below **must be executed** via `Task(...)` calls — do not substitute prose descriptions or skip subagent delegation steps.

## Architecture Overview

(Mermaid diagram or prose describing the high-level architecture)

## Data Flow

(Numbered steps describing how data moves through the system)

## Existing Patterns (extracted)

Concrete patterns from the existing codebase that all new code must match. Each entry includes file path, line numbers, and the specific convention. Executor prompts must paste these verbatim.

- **Pattern name**: Description, file path, line range. Code snippet if short enough.

## Files to Create

- **path/to/new/file.js** -- Description of purpose and key exports/methods

## Files to Modify

- **[path/to/existing/file.js](path/to/existing/file.js)** -- Description of what changes and why

## Key Design Decisions

- **Decision name**: Rationale and trade-offs considered.

## Caveats / Things to Verify

- Constraints, limitations, or assumptions that need runtime validation.

## Execution: Subagent dispatch

(When executing this plan, call each subagent explicitly via `Task` with the matching `subagent_type`.)

- **Per step or phase**: For each todo or logical phase, include an explicit `Task(...)` invocation — do not just name the subagent in prose. Each invocation must specify `subagent_type`, `description`, and a `prompt` with enough context for the subagent to act without re-reading this document. **Prefix every `Task(...)` code block with the verbatim marker comment** `<!-- plan-execution: verbatim-prompt -->` on the line immediately preceding the opening code fence. This marker is the contract with the `plan-execution` rule: the executing agent MUST pass the `prompt=` field byte-for-byte. Example:

  <!-- plan-execution: verbatim-prompt -->
  ```
  Task(
    subagent_type="executor",
    description="Wire new route handler in tags.js",
    prompt="Edit web/routes/tags.js lines 200-250 to add the POST /tags/bulk handler described in the plan. Follow the existing handler pattern on lines 150-199. Return when the file is saved and lint-clean."
  )
  ```

  If a step has documented fill-in placeholders (e.g. `<LOG_FILE_PATH>` captured from a prior phase), spell them out in a short `**Placeholders:**` bullet immediately under the Task block so the executing agent knows which substitutions are allowed. Anything not listed there must NOT be substituted.

- **Correct subagent per step**: `/executor` for mechanical edits, `/test-runner` for running/fixing tests, `/verifier` for post-change validation, `/debugger` only when investigating a failure.
- **Concurrency**: Where steps are independent (e.g. unrelated files, parallel test jobs, validations that do not depend on each other's output), say to **call applicable `Task(...)` invocations concurrently in a single message**; where order matters (e.g. implement then test then verify), state the sequence clearly.
- **Final gate**: The last step must be an explicit `Task(subagent_type="verifier", ...)` call unless the project specifies otherwise.
```

**Plan authoring rules:**

- Every step in `todos` must name specific file paths.
- Every step must have clear inputs, outputs, and success criteria.
- Steps that mutate code must be followed by a test-runner or verifier step.
- The final step must always be a verifier step.
- **Execution mapping**: In `## Execution: Subagent dispatch`, each referenced subagent must include an explicit `Task(subagent_type=..., description=..., prompt=...)` block — not just a prose mention of "/executor". The prompt must be self-contained enough for the subagent to act without re-reading the full plan.
- **Extracted patterns in prompts**: Every executor `Task(...)` prompt must include the relevant extracted patterns from `## Existing Patterns (extracted)` verbatim — the actual code snippets and conventions, not just references like "see billing.js." Executors that must discover patterns on their own will get them wrong.
- **Verbatim-prompt marker**: Every `Task(...)` code block in the dispatch section must be preceded by the HTML comment `<!-- plan-execution: verbatim-prompt -->` on its own line, with no blank line between the marker and the code fence. This is the contract that tells the executing agent (per the `plan-execution` rule) to copy the `prompt=` field byte-for-byte. If a step uses fill-in placeholders (e.g. `<LOG_FILE_PATH>` captured from an earlier phase), list them in a `**Placeholders:**` bullet directly under the Task block; anything not listed there must not be substituted.
- **Parallel execution safety**: When marking steps as concurrent, verify that no two concurrent executors write to the same file or to files that import each other. If unavoidable, add a reconciliation step immediately after the batch (grep for duplicate exports, duplicate `const` declarations, duplicate function definitions). Document this constraint explicitly in the dispatch section.
- **Integration checkpoints**: For plans with 5+ implementation steps, insert at least one concrete integration checkpoint between major milestones (e.g., "CHECKPOINT: hit GET /api/endpoint and confirm 200 with expected shape" or "CHECKPOINT: load the page and confirm the component renders with real data"). Checkpoints are blocking — do not proceed past them until they pass. Place them between backend completion and frontend work, or between service-layer completion and route wiring.
- **Concurrent execution**: When multiple steps are safe to run in parallel, state that explicitly (e.g. "Call two `Task(subagent_type='executor', ...)` concurrently in a single message for `A.js` and `B.js`"). When steps must be serial, state dependencies.
- Assign the correct subagent to each step (executor for mechanical changes, test-runner for tests, verifier for validation, debugger for failures).
- Shopify-related steps must specify MCP tool usage.
- Use `sequentialthinking` for non-trivial logic design within the plan body.
- Never embed open questions or unknowns in the plan -- those should have been resolved in Phase 1.

### Phase 4: Audit via plan-reviewer (first draft)

When the **first draft** of the plan is created and saved under `.cursor/plans/`, **immediately** run `/plan-reviewer` by delegating to the `plan-reviewer` subagent. Treat this as the standard moment for the initial audit; if the verdict is **REJECT** or **REVISE**, revise the plan and **run `/plan-reviewer` again** on the updated draft until the audit outcome is acceptable per the rules below.

**Delegation instructions:**

Use the `Task` tool with `subagent_type="plan-reviewer"`:

```
Task(
  subagent_type="plan-reviewer",
  description="Audit implementation plan",
  prompt="Audit the following plan file: .cursor/plans/<filename>.plan.md

Read the plan file and the plan-reviewer agent instructions at .cursor/agents/plan-reviewer.md.

Review the plan across all six dimensions:
1. Instruction clarity -- are steps specific enough for subagents to execute?
2. Prompt engineering -- do instructions follow good prompt practices?
3. Subagent delegation -- is the right subagent assigned to each step?
4. MCP and tool usage -- are correct MCP servers specified for Shopify work?
5. Pattern extraction -- does the plan include an Existing Patterns section with concrete code-level conventions extracted from the codebase?
6. Execution safety -- are parallel steps safe (no overlapping file writes), are integration checkpoints present, and do executor prompts include extracted patterns verbatim?

Also run the Context Gap Analysis from the plan-reviewer instructions.

Return the full structured audit report with:
- Questions for User (blocking and advisory)
- Plan Summary
- Issues Found (Critical / Warning / Suggestion)
- Corrected Plan (if issues found)
- Verdict (APPROVE / REVISE / REJECT)"
)
```

**After the audit:**

- If the verdict is **APPROVE**, present the plan to the user for final review.
- If the verdict is **REVISE**, fix the flagged warnings/suggestions, then present the revised plan with a summary of changes.
- If the verdict is **REJECT**, fix all critical issues, re-run the audit, and repeat until the plan passes.
- Always surface any new **Blocking** questions from the audit to the user before proceeding.

## Checklist

Before considering the plan complete, verify:

- [ ] All ambiguous points from the user's request were resolved (Phase 1)
- [ ] All unknowns were researched, not assumed (Phase 2)
- [ ] Existing codebase patterns were extracted with file paths, line numbers, and code snippets (Phase 2b)
- [ ] Plan follows the project's YAML frontmatter + markdown format
- [ ] Plan includes `## Existing Patterns (extracted)` section with concrete conventions
- [ ] Every todo step names specific files and has clear success criteria
- [ ] The plan includes **Execution: Subagent dispatch** (or equivalent) with explicit `Task(subagent_type=..., description=..., prompt=...)` blocks for each subagent step — not just prose references like "use /executor"
- [ ] Every executor `Task(...)` prompt includes relevant extracted patterns verbatim (code snippets, not just file references)
- [ ] Every `Task(...)` code block in the dispatch section is preceded by `<!-- plan-execution: verbatim-prompt -->` and any allowed placeholders are documented in a `**Placeholders:**` bullet below the block
- [ ] No two concurrent executor steps write to the same file or to files that import each other
- [ ] Integration checkpoints exist between major milestones (plans with 5+ steps)
- [ ] Subagent assignments are correct per the delegation table
- [ ] Shopify steps specify MCP tool usage
- [ ] A verifier step exists at the end
- [ ] Test-runner steps follow implementation steps
- [ ] `/plan-reviewer` was run on the **first draft** after save, and re-run after material revisions
- [ ] Plan-reviewer audit returned APPROVE or REVISE (not REJECT)
- [ ] All blocking audit questions were surfaced to the user
