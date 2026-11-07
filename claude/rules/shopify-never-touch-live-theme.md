# Shopify CLI: Never Touch the Live Theme by Accident

`shopify theme dev` and `shopify theme push` upload local files the moment they start. If no `--theme` is passed, the CLI uses the `theme` in `shopify.theme.toml` (`[environments.default]`), which in client repos is often the **live** theme. That is how a Bixby Advent preview went live on 2026-11-06.

## Required

1. Before any `shopify theme dev`, `push`, `share`, or `delete`, read every `shopify.theme.toml` in the repo and theme folder, and check which theme each environment points at.
2. Always pass `--theme <id>` explicitly, naming a development or unpublished theme. Or use `--unpublished` / `--development` for a new one. Never rely on a default environment.
3. Never target a theme whose role is `main` (live) unless the user explicitly says to publish live in the current message.
4. After starting `theme dev`, read the first lines of output. If it says it's using the live theme id, stop the server immediately and tell the user.
5. Read-only commands (`theme pull`, `theme list`) are fine against live.
