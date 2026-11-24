---
name: install-delegation-rule
description: "Installs the 'delegate requested changes to a subagent' rule into user-level Claude settings (~/.claude/rules) if it isn't there yet. Use when Charles asks to set up, install, sync or check the delegation rule, sets up Claude Code on a new PC, or asks why changes aren't being delegated to subagents."
---

# Install the Delegation Rule

Makes sure the user-level rules folder has the rule that tells Claude to delegate requested changes to a subagent and pick the model by complexity. The canonical copies live next to this file.

## Files

| Bundled copy (this folder) | Install to |
|----------------------------|------------|
| `delegate-changes-to-subagent.md` | `~/.claude/rules/delegate-changes-to-subagent.md` |
| `subagent-model.md` | `~/.claude/rules/subagent-model.md` |

## Steps

1. Resolve `~/.claude/rules/` for the current user and create it if missing.
2. For `delegate-changes-to-subagent.md`:
   - Missing: copy the bundled file there.
   - Present and identical (ignore line-ending differences): leave it.
   - Present but different: show Charles the diff and ask before overwriting. Never overwrite silently.
3. For `subagent-model.md`:
   - Missing: copy the bundled file there.
   - Present and already mentions `delegate-changes-to-subagent.md`: leave it.
   - Present without that mention: append this sentence to the end of its main paragraph and change nothing else: For requested changes, pick the model by complexity instead (see `delegate-changes-to-subagent.md`).
4. Report what was created, updated or already in place, with the full paths.

## Notes

- This installs the rule for this PC only. Cloud sessions and other computers need it in the Claude project instructions too; mention that once.
- Don't edit any other rules or settings files.
