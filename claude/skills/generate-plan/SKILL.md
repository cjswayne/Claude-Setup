---
name: generate-plan
description: Generates detailed implementation plans for coding tasks. Used when the user asks to plan a feature, create a plan, design an implementation approach, or when a task is complex enough to warrant structured planning before coding. Resolves ambiguity with the user, researches unknowns via MCP and codebase exploration, extracts existing patterns, authors the plan (including when to use /executor, /test-runner, /verifier, and -- for any UI-touching plan -- a dedicated feature-qa testing section during execution, and when those may run concurrently), runs /plan-reviewer on the first draft, and audits via the plan-reviewer subagent.
---

# Generate Plan

## Overview

This skill produces implementation plans in the project's standard format (YAML frontmatter + markdown body, saved to `.claude/plans/`). The planning parent authors the markdown; a fresh `executor` writes the file (stub `Write`, then `Edit` the body). In Claude Code plan mode, the built-in `~/.claude/plans/` file is only for ExitPlanMode approval — the project plan still goes to `.claude/plans/`. Before writing the plan, it resolves ambiguity with the user, researches unknowns, and extracts existing codebase patterns that new code must match. Each plan document must spell out **when** `/executor`, `/test-runner`, and `/verifier` should be used during plan execution, and **when those delegations may run concurrently** (independent file areas, disjoint test suites, parallel validation where ordering does not matter). As soon as the **first draft** of the plan file exists on disk, run `/plan-reviewer` (delegate to the `plan-reviewer` subagent) to audit it; do not wait for polish passes before the initial review unless you are only fixing obvious typos before save.

## Workflow

### Phase 1: Disambiguate

Before any planning, identify gaps in the user's request. Collect **all** questions into a single batch using `AskUserQuestion` (or conversationally if unavailable).

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

1. **Codebase exploration** -- Use `Explore subagent`, `Grep`, `Glob`, and `Read` to find existing patterns, models, services, and conventions in the project. Always ground the plan in how the codebase already works.
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

Generate a slug from the plan name with a random hex suffix (e.g., `feature_name_a1b2c3d4.plan.md`). Save under the **project** `.claude/plans/` directory (never `~/.claude/plans/`, which is plan-mode scratch, and never any `.cursor` directory).

#### Step 1 — Tool contract (mandatory, before any save)

If you are in plan mode, draft in the built-in plan file and call ExitPlanMode for approval first; the project plan is then saved to `.claude/plans/` with `Write` (plan mode only allows writing its own plan file).

Call `Write` with exactly two **named parameters as separate fields**. Do not wrap them in a JSON string. Do not use `raw`, `input`, `file`, or `data`.

Copy-paste shape (stub only — see Step 2):

```
Write
  file_path: C:/Users/me/proj/.claude/plans/feature_name_a1b2c3d4.plan.md
  content:   ---
            name: Feature Name
            overview: What the plan does.
            todos:
              - id: read-and-understand-plan
                content: Read the entire plan from top to bottom before any code changes
                status: pending
            isProject: false
            ---

            # Feature Name

            STUB — executor will replace this body.
```

`content` is the document itself, not a JSON-encoded copy. `file_path` is an absolute path string, not an object.

Forbidden (these are the failure mode — do not emit them):

```
Write
  raw: {"file_path":"...","content":"..."}

Write
  input: "{\"file_path\":\"...\",\"content\":\"...\"}"
```

#### Step 2 — Stub first, then replace the body

Do **not** put the full plan in the first `Write`. Keep the first write small; the executor fills the body with `Edit`.

1. `Write` a short stub only: YAML frontmatter + `# Title` + the line `STUB — executor will replace this body.` Keep it under ~30 lines.
2. `Edit` that stub line (or the stub body after the frontmatter) with the full plan markdown.
3. If a full-body `Edit` fails or is too large, split the body into section-sized `Edit` chunks (Architecture, then Data Flow, and so on).

#### Step 3 — Parent authors; executor writes the file

The planning parent is usually near context limits after research. It must **not** issue the full-body `Write` or `Edit` itself.

1. Parent authors the complete plan markdown (in its draft / the executor prompt).
2. Parent dispatches `Agent(subagent_type="executor")` with: the absolute target path, the stub `Write` example from Step 1, the stub-then-`Edit` rule from Step 2, and the **full plan markdown verbatim**.
3. The executor (fresh context) performs the stub `Write`, then the body `Edit`.
4. Parent runs Phase 4 (`/plan-reviewer`) only after the executor returns the absolute path of the saved file.

Example parent dispatch (fill in the real path and markdown):

```
Agent(
  subagent_type="executor",
  description="Write plan file to disk",
  prompt="Write the plan file.

Call Write with two separate named fields only (file_path, content). Never raw or input. Never stringify the whole argument.

1. Write this stub to <ABSOLUTE_PLAN_PATH> (content is the stub text below, not a JSON blob). Use the real name/overview/title from the pasted plan. Keep stub todos to read-and-understand-plan only even if the full plan has more todos.
2. Edit the line 'STUB — executor will replace this body.' with the plan body only — everything after the closing frontmatter --- in FULL PLAN MARKDOWN below (do not paste a second YAML frontmatter). Then Edit the stub frontmatter with the full plan frontmatter if they differ. If a replace fails or is too large, split into section-sized Edit chunks.

STUB CONTENTS:
---
name: <NAME>
overview: <OVERVIEW>
todos:
  - id: read-and-understand-plan
    content: Read the entire plan from top to bottom before any code changes
    status: pending
isProject: false
---

# <TITLE>

STUB — executor will replace this body.

FULL PLAN MARKDOWN (verbatim):
<PASTE COMPLETE PLAN HERE>

Return the absolute path when the file is on disk."
)
```

**Plan format:** Match existing plans for **structure only** — YAML frontmatter shape and core section headings (`Architecture`, `Data Flow`, files, decisions, caveats). **Do not** copy deferral voice, roadmap labels, or scoping slang from older plans (`locked for v1`, `deferred to v2`, `out of scope for v1`, `Future work — … v2`, `acceptable for v1`, `none in v1`). Those are historical habits, not the format contract. **Always** include `## Existing Patterns (extracted)` and `## Execution: Subagent dispatch` in new plans.

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

> **Execution requirement:** Read and understand this entire plan before making any code changes. Any subagents listed in `## Execution: Subagent dispatch` below **must be executed** via `Agent(...)` calls — do not substitute prose descriptions or skip subagent delegation steps.
>
> **Manual prerequisite (only when this plan includes a `/feature-qa` step):** Before execution begins, the human operator must (1) start the dev server (`npm run dev`) and (2) log the agent's browser into the Shopify dev store so the embedded app loads without hitting the login gate (see `web/docs/feature-qa/loadability-runbook.md`). This session must be ready by the time the `feature-qa` subagent runs; otherwise it will redirect to `accounts.shopify.com` and stop. The agent cannot perform this login itself.

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
- State each as a constraint of **this plan** (concrete behavior). Do not label caveats `for v1` / `until v2`. True deferrals: **Out of scope for this plan:** … and optional **Follow-up (only if requested):** … — still no product version numbers unless the user named them.

## Testing: feature-qa

(Include this section whenever the plan adds or changes a UI feature — new/changed pages or components. Omit it entirely for backend-only plans.)

- **Spec sheet**: Path to the acceptance-criteria spec sheet this plan will produce/update at `docs/feature-qa/<slug>.acceptance.md`.
- **Acceptance criteria (testable bullets)**: Concrete, observable pass/fail statements the `feature-qa` subagent will drive the browser against (e.g. "Submitting the form with a blank name field shows an inline error and does not call the API"). These bullets are the oracle for `feature-qa` -- never the implementation.
- **Prerequisite reminder**: Restate that the dev server must be running and the agent browser must already be signed in per `web/docs/feature-qa/loadability-runbook.md` before the `feature-qa` dispatch step runs (mirrors the top-of-plan callout).

## Execution: Subagent dispatch

(When executing this plan, call each subagent explicitly via `Agent` with the matching `subagent_type`.)

- **Per step or phase**: For each todo or logical phase, include an explicit `Agent(...)` invocation — do not just name the subagent in prose. Each invocation must specify `subagent_type`, `description`, and a `prompt` with enough context for the subagent to act without re-reading this document. **Prefix every `Agent(...)` code block with the verbatim marker comment** `<!-- plan-execution: verbatim-prompt -->` on the line immediately preceding the opening code fence. This marker is the contract with the `plan-execution` rule: the executing agent MUST pass the `prompt=` field byte-for-byte. Example:

  <!-- plan-execution: verbatim-prompt -->
  ```
  Agent(
    subagent_type="executor",
    description="Wire new route handler in tags.js",
    prompt="Edit web/routes/tags.js lines 200-250 to add the POST /tags/bulk handler described in the plan. Follow the existing handler pattern on lines 150-199. Return when the file is saved and lint-clean."
  )
  ```

  If a step has documented fill-in placeholders (e.g. `<LOG_FILE_PATH>` captured from a prior phase), spell them out in a short `**Placeholders:**` bullet immediately under the Task block so the executing agent knows which substitutions are allowed. Anything not listed there must NOT be substituted.

- **Correct subagent per step**: `/executor` for mechanical edits, `/test-runner` for running/fixing tests, `/verifier` for post-change validation, `/feature-qa` for browser-driven end-to-end + visual QA of any UI feature (new/changed pages or components under `web/frontend/`; runs in parallel with `/verifier`, oracle is the acceptance criteria, never edits code), `/debugger` only when investigating a failure.
- **Concurrency**: Where steps are independent (e.g. unrelated files, parallel test jobs, validations that do not depend on each other's output), say to **call applicable `Agent(...)` invocations concurrently in a single message**; where order matters (e.g. implement then test then verify), state the sequence clearly.
- **Per-phase verification gate (mandatory for multi-phase plans)**: Every implementing phase must be immediately followed by an independent verification gate — a `Agent(subagent_type="verifier", ...)` block that **re-runs the concrete build/test commands itself** (real commands looked up from the project's `package.json`, not invented) and inspects that phase's outputs, before the next phase is dispatched. The gate does not trust the executor's self-report. State the protocol explicitly in the dispatch preamble: proceed to the next phase only on `PASS`; on `FAIL`, dispatch a `Agent(subagent_type="debugger", ...)` with the gate's exact failing output to fix, then re-run the same gate until it passes. Because the verifier is read-only (it reports, it does not edit), the plan must include the debugger failure path — a verifier gate with no fix path is incomplete. Include a reusable debugger `Agent(...)` block with a documented `<GATE_FAILURE_DETAILS>` placeholder rather than one per phase.
- **Final gate**: The last step must be an explicit `Agent(subagent_type="verifier", ...)` call unless the project specifies otherwise. This final pass is in addition to the per-phase gates and should include the cross-phase wiring trace.
```

**Plan authoring rules:**

- Every step in `todos` must name specific file paths.
- Every step must have clear inputs, outputs, and success criteria.
- Steps that mutate code must be followed by a test-runner or verifier step.
- The final step must always be a verifier step.
- **No invented product version roadmaps:** Do not frame decisions as `v1`, `for v1`, `locked for v1`, `MVP only`, `ship in v1`, `deferred to v2`, or `later version` unless the user **explicitly** asked for versioned/phased product delivery. Write the behavior as the design for this work. Execution **Phase N** sections (dispatch steps inside this plan) are fine and are not product versions. Real API/library version pins (e.g. Shopify `2026-04`, Gmail API `v1`) are fine.
- **Execution mapping**: In `## Execution: Subagent dispatch`, each referenced subagent must include an explicit `Agent(subagent_type=..., description=..., prompt=...)` block — not just a prose mention of "/executor". The prompt must be self-contained enough for the subagent to act without re-reading the full plan.
- **feature-qa manual prerequisite**: If the plan includes a `/feature-qa` step, the top-of-plan `> **Execution requirement:**` callout MUST also state the manual prerequisite that the human start the dev server and log the agent browser into the Shopify dev store before execution (per `web/docs/feature-qa/loadability-runbook.md`), so the session is ready by the time `feature-qa` runs. Omitting this means feature-qa will hit the login gate and stop.
- **feature-qa test section (mandatory for UI-touching plans)**: If the plan adds or changes any UI (new/changed pages or components), include a `## Testing: feature-qa` section (see template) with testable acceptance-criteria bullets and the spec sheet path under `docs/feature-qa/*.acceptance.md`. Add a matching `Agent(subagent_type="feature-qa", ...)` block in `## Execution: Subagent dispatch` whose prompt points at that spec sheet -- do not rely on the generic subagent-delegation bullet alone to cover UI testing. Backend-only plans omit this section entirely.
- **Extracted patterns in prompts**: Every executor `Agent(...)` prompt must include the relevant extracted patterns from `## Existing Patterns (extracted)` verbatim — the actual code snippets and conventions, not just references like "see billing.js." Executors that must discover patterns on their own will get them wrong.
- **Verbatim-prompt marker**: Every `Agent(...)` code block in the dispatch section must be preceded by the HTML comment `<!-- plan-execution: verbatim-prompt -->` on its own line, with no blank line between the marker and the code fence. This is the contract that tells the executing agent (per the `plan-execution` rule) to copy the `prompt=` field byte-for-byte. If a step uses fill-in placeholders (e.g. `<LOG_FILE_PATH>` captured from an earlier phase), list them in a `**Placeholders:**` bullet directly under the Task block; anything not listed there must not be substituted.
- **Parallel execution safety**: When marking steps as concurrent, verify that no two concurrent executors write to the same file or to files that import each other. If unavoidable, add a reconciliation step immediately after the batch (grep for duplicate exports, duplicate `const` declarations, duplicate function definitions). Document this constraint explicitly in the dispatch section.
- **Integration checkpoints**: For plans with 5+ implementation steps, insert at least one concrete integration checkpoint between major milestones (e.g., "CHECKPOINT: hit GET /api/endpoint and confirm 200 with expected shape" or "CHECKPOINT: load the page and confirm the component renders with real data"). Checkpoints are blocking — do not proceed past them until they pass. Place them between backend completion and frontend work, or between service-layer completion and route wiring.
- **Per-phase verification gates**: For any multi-phase plan, follow each implementing phase with a `Agent(subagent_type="verifier", ...)` gate that re-runs that phase's concrete build/test commands independently and blocks the next phase until it returns `PASS`. Resolve the actual commands from the project's `package.json` scripts (do not assume a `typecheck` or `lint` script exists — verify). Pair the gates with a reusable debugger `Agent(...)` block (documented `<GATE_FAILURE_DETAILS>` placeholder) so a `FAIL` has a defined fix-then-recheck loop. This is distinct from integration checkpoints (which validate runtime behavior at seams) and from the single final verifier (which validates the whole feature + wiring).
- **Concurrent execution**: When multiple steps are safe to run in parallel, state that explicitly (e.g. "Call two `Agent(subagent_type='executor', ...)` concurrently in a single message for `A.js` and `B.js`"). When steps must be serial, state dependencies.
- Assign the correct subagent to each step (executor for mechanical changes, test-runner for tests, verifier for validation, debugger for failures).
- Shopify-related steps must specify MCP tool usage.
- Use `sequentialthinking` for non-trivial logic design within the plan body.
- Never embed open questions or unknowns in the plan -- those should have been resolved in Phase 1.

### Phase 4: Audit via plan-reviewer (first draft)

When the **first draft** of the plan is created and saved under `.claude/plans/` (the executor from Phase 3 Step 3 has returned the absolute path), **immediately** run `/plan-reviewer` by delegating to the `plan-reviewer` subagent. Treat this as the standard moment for the initial audit; if the verdict is **REJECT** or **REVISE**, revise the plan and **run `/plan-reviewer` again** on the updated draft until the audit outcome is acceptable per the rules below.

**Delegation instructions:**

Use the `Agent` tool with `subagent_type="plan-reviewer"`:

```
Agent(
  subagent_type="plan-reviewer",
  description="Audit implementation plan",
  prompt="Audit the following plan file: .claude/plans/<filename>.plan.md

Read the plan file and the plan-reviewer agent instructions at .claude/agents/plan-reviewer.md.

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
- [ ] Project plan saved under `.claude/plans/` (not only in plan mode's `~/.claude/plans/`); save used `Write` with separate `file_path` and `content` fields — no `raw`/`input` JSON wrap
- [ ] Planning parent did not write the full plan file; an `executor` wrote a stub, then `Edit`'d the body
- [ ] Plan follows the project's YAML frontmatter + markdown format (structure only — not old-plan deferral slang)
- [ ] No invented product `v1`/`v2`/`MVP`-as-sequel language; caveats use this-plan / out-of-scope framing (unless the user named versions)
- [ ] Plan includes `## Existing Patterns (extracted)` section with concrete conventions
- [ ] Plans that add or change UI include a `## Testing: feature-qa` section with testable acceptance criteria, a spec sheet path under `docs/feature-qa/*.acceptance.md`, and a matching `Agent(subagent_type="feature-qa", ...)` dispatch block (backend-only plans omit this)
- [ ] Every todo step names specific files and has clear success criteria
- [ ] The plan includes **Execution: Subagent dispatch** (or equivalent) with explicit `Agent(subagent_type=..., description=..., prompt=...)` blocks for each subagent step — not just prose references like "use /executor"
- [ ] Every executor `Agent(...)` prompt includes relevant extracted patterns verbatim (code snippets, not just file references)
- [ ] Every `Agent(...)` code block in the dispatch section is preceded by `<!-- plan-execution: verbatim-prompt -->` and any allowed placeholders are documented in a `**Placeholders:**` bullet below the block
- [ ] No two concurrent executor steps write to the same file or to files that import each other
- [ ] Integration checkpoints exist between major milestones (plans with 5+ steps)
- [ ] Every implementing phase in a multi-phase plan is followed by an independent `verifier` gate that re-runs concrete build/test commands, plus a reusable debugger `Agent(...)` block (with a `<GATE_FAILURE_DETAILS>` placeholder) defining the FAIL → fix → re-check loop
- [ ] Subagent assignments are correct per the delegation table
- [ ] Shopify steps specify MCP tool usage
- [ ] A verifier step exists at the end
- [ ] Test-runner steps follow implementation steps
- [ ] `/plan-reviewer` was run on the **first draft** after save, and re-run after material revisions
- [ ] Plan-reviewer audit returned APPROVE or REVISE (not REJECT)
- [ ] All blocking audit questions were surfaced to the user
