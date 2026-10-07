---
description: "How to save a plan file — executor writes a stub with Write, then Edits in the body; the project plan lives in .claude/plans, not ~/.claude/plans"
---

# Creating a plan file: Write tool call

Claude Code's plan mode keeps its own scratch plan under `~/.claude/plans/`. That file is only for the ExitPlanMode approval step — it is **not** the project plan. Once the plan is approved (or when not in plan mode), save the project plan to `<project-repo>/.claude/plans/<slug>_<8hex>.plan.md` with **Write** (then **Edit**).

Use exactly two named parameters on Write:

- `file_path` (string): `<project-repo>/.claude/plans/<slug>_<8hex>.plan.md` (absolute)
- `content` (string): the stub or document markdown as plain text

Pass them as separate fields. Do not wrap them in a JSON string.

## Who writes the file

The planning parent (the agent that researched the plan) must **not** issue the full-body Write or Edit after a long research session. Author the markdown, then dispatch `Agent(subagent_type="executor")` with the absolute path and the full plan text. The executor performs the file writes in a fresh context.

## Two-step write (mandatory)

Do not put the full plan in the first Write.

1. Write a short stub only (frontmatter + title + `STUB — executor will replace this body.`, under ~30 lines).
2. Edit that stub line with the full plan body.
3. If a full-body Edit fails or is too large, split it into section-sized Edit chunks.

## Correct (stub Write)

```
Write
  file_path: C:/Users/me/proj/.claude/plans/feature_name_a1b2c3d4.plan.md
  content:   ---
             name: Feature Name
             overview: What the plan does.
             todos:
               - id: first-step
                 content: Do the first step in path/to/file.js
                 status: pending
             isProject: false
             ---

             # Feature Name

             STUB — executor will replace this body.
```

`content` is the document itself, not a JSON-encoded copy of the document.

## Incorrect

```
Write
  raw: {"file_path":"...","content":"..."}
```

```
Write
  input: "{\"file_path\":\"...\",\"content\":\"...\"}"
```

- Do not leave the plan only in plan mode's `~/.claude/plans/` file
- Do not use a `raw`, `input`, `file`, or `data` parameter
- Do not stringify the whole argument (`"{\"file_path\":...}"`)
- Do not send `file_path` without `content`
- Do not put the full plan in the first Write
- Do not have the research parent Write the full plan itself
- Do not double-encode markdown (no extra JSON escaping of the plan body)
- Do not write to `~/.claude/plans/` (plan-mode scratch) or any `.cursor` directory (see `plan-location`)

## Self-check before sending

1. The tool name is `Write` or `Edit`
2. Parameter names are only `file_path` and `content` (Write) or `file_path` / `old_string` / `new_string` (Edit)
3. `file_path` is an absolute path string under the project's `.claude/plans/`, not an object
4. The first Write is a stub, not the full plan
5. An executor is performing the write, not the exhausted parent
6. The call is not a JSON blob and does not use `raw` or `input`
