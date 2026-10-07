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

## Credits

Many of the role-based skills and Cursor rules here (for example `ai-engineer`, `backend-architect`, `python-pro`, `quant-analyst` and `security-auditor`) started from [wshobson/agents](https://github.com/wshobson/agents) by Seth Hobson, used under the MIT License below. Some have since been edited. My own additions include `ai-pipeline-analyst`, `debug-sequential-task-planner`, `generate-plan`, `task-decomposer`, the rules in `claude/rules` and the subagents in `claude/agents`. The `verifier`, `debugger` and `test-runner` subagents grew out of the examples in [Cursor's subagent docs](https://cursor.com/docs/subagents) and have since been rewritten.

```
MIT License

Copyright (c) 2024 Seth Hobson

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
