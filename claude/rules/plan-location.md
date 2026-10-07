---
description: "Where to save generated plans"
---

# Plan File Location

When a plan is made, save it inside the relevant project repository's own
`.claude/plans/` directory. Never write plans to the user-level
`~/.claude/plans/` (Claude Code's plan-mode scratch space) or to any `.cursor`
directory.

- ✅ GOOD: `<project-repo>/.claude/plans/my_feature_1234abcd.plan.md`
- ❌ BAD: `C:\Users\<user>\.claude\plans\my_feature_1234abcd.plan.md`
- ❌ BAD: `<project-repo>/.cursor/plans/my_feature_1234abcd.plan.md`

If the workspace has multiple repos, put the plan in the repo the plan is
actually about. Create the `.claude/plans/` directory in that repo if it does
not exist yet.
