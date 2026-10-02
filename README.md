# claude-setup

The rules, skills and subagents I use day to day with [Claude Code](https://claude.com/claude-code) and [Cursor](https://cursor.com).

This repo is published automatically from my private config sync, so it always matches what I'm actually running. Please don't open PRs against it, because changes here get overwritten. Issues and ideas are welcome.

## Layout

| Path | What it is | Where it lives on my machine |
|---|---|---|
| `claude/CLAUDE.md` | Global instructions loaded into every Claude Code session | `~/.claude/CLAUDE.md` |
| `claude/rules/` | Always-on rules (coding style, workflow, tool use) | `~/.claude/rules/` |
| `claude/skills/` | Skills Claude Code loads on demand | `~/.claude/skills/` |
| `claude/agents/` | Custom subagents (executor, verifier, plan-reviewer, ...) | `~/.claude/agents/` |
| `cursor/rules/` | The same ideas as Cursor rules (`.mdc`) | `~/.cursor/rules/` |
| `cursor/skills/` | Cursor skills | `~/.cursor/skills/` |
| `cursor/agents/` | Cursor subagents | `~/.cursor/agents/` |

## Using them

Copy whatever you like into the matching folder in your own home directory. Some rules reference tools or MCP servers I have installed (for example the Shopify Dev MCP), so adjust those to your setup.
