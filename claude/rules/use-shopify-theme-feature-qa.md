---
description: "When to dispatch shopify-theme-feature-qa for Shopify theme UI changes"
paths:
  - "**/*.{liquid,css,js,scss}"
  - "**/shopify.theme.toml"
  - "**/templates/**/*.json"
---

# Use Shopify Theme Feature-QA

When validating **Shopify theme** storefront UI end-to-end (real browser
interaction and visual checks after section/snippet/template/asset changes),
use the `/shopify-theme-feature-qa` subagent (also referred to as
theme-feature-qa).

It drives the running theme preview via the chrome-devtools MCP and reports
functional + visual bugs. Its oracle is the parent agent's change brief /
acceptance criteria, never the Liquid implementation. It never edits theme
files.

## Do not use it for

- Backend-only / Admin API / non-storefront work
- Headless unit tests (use `/test-runner`)
- Static file review without a running preview (use `/verifier` or code review)

## Prerequisites the parent must ensure

1. `shopify theme dev` / `st dev` is running and a `PREVIEW_URL` is known
   (typically `http://127.0.0.1:9292`).
2. The Task prompt includes: change brief (or acceptance doc path), changed
   files list (route scoping only), target product/page handles, and
   `PREVIEW_URL`.
3. Docs and artifacts stay under
   `~/.cursor/docs/shopify-theme-feature-qa/` — never inside the theme folder.

See `~/.cursor/docs/shopify-theme-feature-qa/loadability-runbook.md`.
