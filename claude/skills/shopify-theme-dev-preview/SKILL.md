---
name: shopify-theme-dev-preview
description: Safe setup for previewing and shipping Shopify theme changes without touching the live theme. Use BEFORE any `shopify theme dev`, `theme push`, `theme share`, `theme delete`, a preview_start/launch.json entry that runs the Shopify CLI, or any request like "preview this", "spin up a dev theme", "push to a draft", "push it live" in a Shopify theme repo.
---

# Shopify theme dev preview, without touching live

Two failure modes this prevents (both happened on Bixby, Nov 2026):

1. **Default environment points at live.** `shopify.theme.toml` `[environments.default]` often has `theme = "<live id>"`. Any `theme dev` or `theme push` without `--theme` silently targets live. `theme dev` uploads the whole folder the moment it starts.
2. **Stale folder overwrites newer live work.** A worktree cut from `main` is often behind live (theme editor edits, other sessions' pushes). Syncing that folder to live, even for 17 seconds, reverts every file that was newer on live. On Bixby it reverted the freshly launched BFCM page to the 2025 version, and nobody noticed for three days because the check only looked for deleted files.

Follow every step. Do not skip the checks because "it's only a preview".

## 1. Know which theme is live

```bash
shopify theme list --store <store>.myshopify.com --json
```

Note the id whose `role` is `live`. That id is off limits for everything in steps 2 to 5.

## 2. Read every toml

Read each `shopify.theme.toml` (repo root and theme folder). For every `[environments.*]` block, write down which theme id it targets. If any targets the live id, treat environments as unusable: never pass `-e`, and always pass `--theme` explicitly.

## 3. Get a preview theme

Prefer a copy of live, so the preview shows the real content and settings:

```bash
shopify theme duplicate --store <store>.myshopify.com --theme <live id> --name "<Client> preview <task>" --json
```

Record the new id. Name it so anyone can tell what it is for. Reuse it for the whole task.

## 4. Make the folder current before syncing anything

The folder you sync must not be behind live for files you are not editing.

```bash
shopify theme pull --store <store>.myshopify.com --theme <live id> --path "<scratch>/live-now"
```

Compare `<scratch>/live-now` with the working folder (ignore line endings). For each differing file you are NOT changing in this task, copy the live version into the working folder first. For files you ARE changing, three-way merge (base = git commit the branch started from, ours = working folder, theirs = live) with `git merge-file -p ours base theirs`.

## 5. Run the dev server against the preview theme only

```bash
shopify theme dev --store <store>.myshopify.com --theme <preview id> --path "<theme folder>" --port 9292
```

- Always pass `--store`, `--theme <preview id>` and `--path`. Never rely on the current directory's toml.
- A `.claude/launch.json` entry that runs `shopify theme dev` must include `--theme <preview id>` in `runtimeArgs`. Never add one without it.
- Add `--ignore <file>` for files the store rejects (they otherwise show an error page).
- Read the first lines of output before doing anything else. The preview URL must contain `preview_theme_id=<preview id>`. If it names the live id, stop the server at once and go to step 8.

To push files to the preview without a server:

```bash
shopify theme push --store <store>.myshopify.com --theme <preview id> --path "<theme folder>" --only <file> --only <file> --nodelete --json
```

Never use `--allow-live` or `--publish` here.

## 6. Going live (only when the user says so in the current message)

1. Pull the current live copies of exactly the files you will push into a scratch folder.
2. Compare each with the git base. If live changed, three-way merge so theme editor edits are kept.
3. Stage only those files in a scratch theme folder (with the standard empty `assets config layout locales sections snippets templates` folders).
4. Push in dependency order, each with `--only` and `--nodelete`: section and snippet files that add schema or blocks first, then templates that use them. A template pushed before its section's new block type is rejected.
   ```bash
   shopify theme push --path "<stage>" --store <store>.myshopify.com --theme <live id> --allow-live --nodelete --only <file> --json
   ```
5. Verify on the live storefront with preview mode cleared (`?preview_theme_id=` then check `Shopify.theme.id` equals the live id).
6. Commit the live versions of the pushed files back to the repo, so `main` matches live.

## 7. Before reporting a page as live

Pull the live file and compare its content with the version you meant to ship. "Pushed" is not "live" until that comparison matches. When the user later says a page is "finished", check live before deleting its preview themes.

## 8. If live was touched by accident

1. Stop the server or command immediately.
2. Pull the full live theme into a scratch folder.
3. Compare every file's **content** (not just file names) with both git `main` and the last known live state (other sessions' pushes, theme editor edits). List every file that changed.
4. Tell the user exactly which files changed and what each now contains. Do not guess that only the files you edited were affected.
5. Restore any file that was reverted, using the steps in 6.

## 9. Clean up

When the task ships: stop dev servers, remove launch.json entries for them, and delete preview themes after the user confirms (`shopify theme delete --store <store>.myshopify.com --theme <preview id> --force`). Never delete the live theme or themes you did not create.
