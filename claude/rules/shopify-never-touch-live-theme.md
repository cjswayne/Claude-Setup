# Shopify CLI: Never Touch the Live Theme by Accident

`shopify theme dev` and `shopify theme push` upload local files the moment they start. If no `--theme` is passed, the CLI uses the `theme` in `shopify.theme.toml` (`[environments.default]`), which in client repos is often the **live** theme. That is how a Bixby Advent preview went live on 2026-11-06. Because that worktree was behind live, the same 17-second sync also reverted the just-launched BFCM page to its 2025 version, and nobody noticed for three days.

## Required

0. Load the `shopify-theme-dev-preview` skill before any `shopify theme dev`, `push`, `share` or `delete`, before adding a `.claude/launch.json` entry that runs the Shopify CLI, and before any request to preview or push a theme. Follow its steps.
1. Before any `shopify theme dev`, `push`, `share`, or `delete`, read every `shopify.theme.toml` in the repo and theme folder, and check which theme each environment points at.
2. Always pass `--theme <id>` explicitly, naming a development or unpublished theme. Or use `--unpublished` / `--development` for a new one. Never rely on a default environment.
3. Never target a theme whose role is `main` (live) unless the user explicitly says to publish live in the current message.
4. After starting `theme dev`, read the first lines of output. If it says it's using the live theme id, stop the server immediately and tell the user.
5. Read-only commands (`theme pull`, `theme list`) are fine against live.
6. Never sync a folder that is behind live. A worktree cut from `main` can be days behind theme editor edits and other sessions' pushes. Before syncing a whole folder anywhere, bring the files you are not editing up to date from live.
7. Push to live only with `--only <file>` and `--nodelete`, after pulling the live copies of those files and three-way merging any editor changes. Never push a whole folder to live.
8. If live may have been touched by accident, pull the full live theme and compare every file's content (not just which files exist) with `main` and with the last known live state. Report every changed file to the user and restore anything reverted.
9. "Pushed" is not "live" until a fresh pull of the live file matches what you shipped. Check that before telling the user a page is live, and before deleting its preview themes.
10. A PreToolUse hook (`~/.claude/hooks/shopify-theme-guard.py`) denies theme dev/push/delete/rename without `--theme` and asks the user before any `--allow-live`, `--publish` or `theme publish`. Never work around it (for example by wrapping the command in a script); fix the command instead.
