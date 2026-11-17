# Shopify: Create shopify.theme.toml When a Theme Has None

Every Shopify theme folder we work in should have a `shopify.theme.toml`, so the store is named in one place and CLI commands don't prompt for or guess a store.

## When this applies

At the start of any task in a Shopify theme repo or folder (a folder containing `config`, `layout`, `sections`, `snippets` and `templates`), and always before the first `shopify theme dev`, `push`, `pull`, `share` or `delete`.

## Required

1. Look for `shopify.theme.toml` in the theme folder (the folder that holds `layout/theme.liquid`), not just the repo root. Client repos usually keep the theme in a subfolder such as `Live/`, `live/` or a versioned folder name.
2. If there is none, create one in that theme folder before doing anything else with the Shopify CLI:

   ```toml
   [environments.default]
   store = "<store>.myshopify.com"
   ```

3. Find the store domain from the repo, not memory: other tomls in sibling client folders, `.shopify/`, `launch.json`, the git remote or folder name, or `shopify theme list`. If you can't determine it with confidence, ask the user for it. Never guess a store.
4. Do not put a `theme = "<id>"` line in `[environments.default]`. A default theme id that is the live theme is how a preview went live (see `shopify-never-touch-live-theme`). Pass `--theme <id>` explicitly on every command instead. Only add a `theme` to an environment if it names a development or unpublished theme, and name that environment for its purpose (for example `[environments.preview]`).
5. Never commit a toml that targets the live theme as the default. If an existing toml's default points at the live theme, tell the user rather than editing it unasked.
6. Tell the user you created the file and where. Follow the `ask-before-extra-work` rule: don't change or delete an existing toml unless asked.

## Example

```text
❌ BAD — run shopify theme dev with no toml, or create one with theme = <live id>
✅ GOOD — create <theme folder>/shopify.theme.toml with only the store, then run:
   shopify theme dev --store <store> --theme <preview id> --path <theme folder>
```
