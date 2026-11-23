# Client Apps Go in apps/private

New Shopify apps built for a single client (custom apps, extension-only apps, custom-distribution apps) are created in:

`C:/Users/cjswa/Documents/jadepuma/apps/private/<app-name>`

- Pass `--path C:\Users\cjswa\Documents\jadepuma\apps\private` to `shopify app init`.
- Never create a client app directly in `apps/`; that folder is for JadePuma's own public apps.
- Save the app's plan in `<app>/.claude/plans/` (see `plan-location`).
