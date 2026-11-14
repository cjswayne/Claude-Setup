# Shopify: Pull the Latest Theme Before Reading It

Repo theme folders go stale. Merchants edit in the theme editor, apps inject files, and other sessions push. Reading the repo copy and calling it "live" leads to wrong diagnoses.

## When this applies

Any time you read theme code to explain, debug or QA what a store is showing, including before quoting a file and line as the cause of a storefront behavior.

## Required

1. Identify the theme the user is looking at: the live theme (role `main`) by default, or the preview or unpublished theme they name. Get the id with `shopify theme list --store <store>` if you don't already have it.
2. Before reading theme files, pull that theme read-only into a fresh temp folder outside the repo, never over the working tree:

   ```bash
   shopify theme pull --store <store> --theme <id> --path <temp-dir>
   ```

   Use `--only <file>` (repeatable) to fetch just the files you need when the full theme is slow.
3. Read from the pulled copy. Diff it against the repo folder (`diff --strip-trailing-cr -w`) and tell the user which files differ, if any.
4. Don't overwrite repo files with the pulled copy unless the user asks (see `sync-github-branch` and `shopify-never-touch-live-theme`).
5. When you report a cause, say which theme and when it was pulled, e.g. "live theme 159945720034, pulled just now".
