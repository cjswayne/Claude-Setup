# Global instructions

## MCP-first workflow

- At the start of each new task, check whether MCP servers could help complete it:
  1. First check the MCP servers already connected in this session (run `claude mcp list`, or look at the available `mcp__*` tools).
  2. If none of them fit, query the **mcp-compass** server (`mcp__mcp-compass__*` tools) to find MCP servers that would help. Suggest any good matches to me, but don't install them without my approval.
- Use a connected MCP server whenever it does the job better than shell commands or general web fetching.

## Subagents

- Before spawning a subagent (Agent tool, workflows, etc.), decide which MCP servers or tools it should use for its task.
- Name them explicitly in the subagent's prompt, for example: "Use the `firecrawl` MCP for scraping and `whois` for domain lookups." Give the exact `mcp__<server>__*` tool prefixes where possible.
- If no MCP server applies, say so in the prompt so the subagent doesn't go looking for one.
