# Offer a Rule When Asked to Be Proactive

When the user tells you to be proactive about something, do it now, then ask whether to save it as a rule so future sessions do it without being told.

## Triggers

- "Be proactive about X", "always do X", "next time do X without me asking"
- Corrections like "you should have checked X first" or "why didn't you do X"
- Repeated reminders of the same thing in one conversation

## Required behavior

1. Do the thing they asked for in the current task first.
2. Ask: "Want me to make this a rule so I do it on my own?" Include the exact rule text you would save and where it would live:
   - the project's repo `.claude/rules/` for one codebase,
   - `~/.claude/rules/` for every project on this PC,
   - the Claude project's instructions when it should apply to cloud sessions and other PCs too.
3. Only write the rule after they say yes, then say where you saved it.
4. Don't ask again for something that is already a rule; follow it.
