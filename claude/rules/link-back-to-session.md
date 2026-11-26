# Leave a Link Back to the Claude Session

Charles needs to jump back into the session that did a piece of work without searching for it. Anything durable a session leaves behind should carry a link back to that session.

## Get the link

1. Desktop app: call `mcp__ccd_session_mgmt__get_session` with `session_id: "self"` and use its `link` field (looks like `claude://claude.ai/epitaxy/local_<id>`). Get it once per session and reuse it.
2. Cloud session: use the session's claude.ai/code URL.
3. CLI or no link available: use the resume command `claude --resume <id>`, with the id from the `CLAUDE_CODE_SESSION_ID` environment variable.
4. A subagent uses its parent session's link; the parent passes it in the brief.

## Where to leave it

Add it whenever the session creates or updates one of these:

- Notion tasks: in the status note or comment written about the work
- Git commits: a trailer line `Claude-Session: <link>` above the Co-Authored-By line
- Pull requests: a `Claude session: <link>` line at the end of the description, above the Generated-with line
- Plan files in `.claude/plans/`: a `session: <link>` field in the frontmatter
- Handoff docs, reports, READMEs or notes written for Charles or the JadePuma team
- Scheduled tasks and routines: in the task description
- Shopify preview themes: in the summary given to Charles alongside the preview link (theme names are too short)

If a place already has a link from an earlier session, add the new one rather than replacing it.

## Where not to leave it

Never put it in anything a client, customer or the public sees: client emails, storefront content, theme code, app code comments, Klaviyo or other marketing content, or PRs in repos clients can read. In those cases, give Charles the link in chat instead.
