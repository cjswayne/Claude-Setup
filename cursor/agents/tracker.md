# Tracking Agent

## Purpose

Ensure every user-facing interaction in `web/frontend/` is instrumented with the `trackEvent` function from `web/frontend/utils/tracking.js`.

## When to Run

Run this agent whenever frontend code is created or modified under `web/frontend/pages/` or `web/frontend/components/`. This includes new pages, new components, or edits that add/change interactive elements.

## trackEvent Signature

```js
import { trackEvent } from "../utils/tracking";
// Adjust relative path depth based on file location

trackEvent(action, category, label);
```

| Param    | Type   | Required | Description |
|----------|--------|----------|-------------|
| action   | string | yes      | Snake_case identifier: `{scope}__{verb}_{noun}` |
| category | string | no       | PascalCase feature name (e.g. `"CategoryManager"`) |
| label    | string | no       | Contextual value — path, ID, or dynamic descriptor |

## Naming Conventions

Follow the existing patterns exactly:

- **action**: `{page_or_component}__{verb}_{noun}` using double underscore as separator.
  - Scope prefixes by page: `home__`, `category_manager__`, `tag_page__`, `bulk_migration__`, `product_type_migrator__`, `assign_bulk_categories__`, `usage_banner__`, `pricing__`, `completeness_score__`, `settings__`, etc.
  - Verb examples: `open`, `click`, `save`, `start`, `cancel`, `toggle`, `change`, `refresh`, `clear`, `confirm`, `reject`, `accept`, `exclude`, `submit`, `download`.
  - Noun examples: `category_manager`, `tag_migrator`, `run`, `filter`, `search`, `modal`, `product`, `configuration`, `subscription`, `cap`.
- **category**: PascalCase feature group — `"BulkAssignCategories"`, `"UsageBanner"`, `"Pricing"`, `"CompletenessScore"`, `"CategoryManager"`, `"TagMigrator"`, `"ProductTypeMigrator"`, etc.
- **label**: A route path (`"/assign-bulk-categories"`), an entity ID, or a descriptive string (`"new"`, `"update"`). Optional — omit when not useful.

## Interaction Types to Instrument

Add `trackEvent` calls for every instance of these elements. This includes both standard React/Polaris components and Shopify `s-*` web components.

### Standard React / Polaris Components

| Interaction        | JSX Pattern                        | Where to Add trackEvent |
|--------------------|------------------------------------|-------------------------|
| Button click       | `<Button onClick={...}>`           | Inside the onClick handler, before or after the primary logic |
| Link navigation    | `<Link to={...}>`, `<a href={...}>` | Wrap with an onClick handler that calls trackEvent, or add to existing onClick |
| Form submit        | `<Form onSubmit={...}>`            | Inside the onSubmit handler |
| Toggle / checkbox  | `onChange={...}` on toggles/checks | Inside the onChange handler |
| Select change      | `onChange={...}` on selects        | Inside the onChange handler |
| Icon button        | `<Button icon={...} onClick={...}>` | Inside the onClick handler |
| Modal open/close   | Modal trigger buttons              | When modal opens and when it closes |
| Save bar actions   | Save / Discard bar buttons         | Inside each bar action handler |
| Tab switch         | `<Tabs onSelect={...}>`            | Inside the onSelect handler |
| Pagination         | Page change callbacks               | Inside the page change handler |
| External links     | `<a href="..." target="_blank">`   | Add onClick that calls trackEvent before navigation |
| Bulk actions       | Bulk action buttons in resource lists | Inside each bulk action handler |

### Shopify `s-*` Web Components

These are native Shopify web components used throughout the app. They accept standard DOM event handlers like `onClick` and `onChange`.

| Interaction           | JSX Pattern                                  | Where to Add trackEvent |
|-----------------------|----------------------------------------------|-------------------------|
| s-button click        | `<s-button onClick={...}>`                   | Inside the onClick handler. If no onClick exists, add one. For `commandFor` buttons that toggle modals, add onClick alongside the command attributes. |
| s-link navigation     | `<s-link href={...}>`                        | Add an onClick handler that calls trackEvent. Do NOT call `e.preventDefault()` — let navigation proceed. |
| s-clickable click     | `<s-clickable href={...}>`                   | Add an onClick handler that calls trackEvent. These are clickable wrappers (e.g. YouTube cards). Do NOT prevent default. |
| s-clickable-chip      | `<s-clickable-chip onClick={...}>`           | Inside the onClick handler. These are tag/filter chip interactions (remove excluded item, select filter, etc.). |
| s-select change       | `<s-select onChange={...}>`                  | Inside the onChange handler. Track the selection change with the new value as the label. |
| s-checkbox change     | `<s-checkbox onChange={...}>`                | Inside the onChange handler. Track the toggle with checked state as the label. |
| s-navigation click    | `<s-navigation onClick={...}>`               | Inside the onClick handler or add one for nav items. |

**Notes on `s-*` tracking:**
- `s-button` elements with `commandFor` and `command` attributes (modal triggers) should still get an `onClick` for tracking — `commandFor` and `onClick` coexist.
- `s-clickable` and `s-link` elements that navigate externally (with `target="_blank"`) should track via `onClick` without preventing default browser navigation.
- For `s-clickable-chip` removal actions, use action pattern `{scope}__remove_{noun}` (e.g. `product_type_bulk_migration__remove_excluded`).
- `s-select` tracking should include the selected value in the label param when practical.

## Step-by-Step Process

1. **Scan** — Read every `.jsx` and `.js` file under `web/frontend/pages/` and `web/frontend/components/`.
2. **Check import** — For each file, verify whether `trackEvent` is imported from `../utils/tracking` (adjust relative path for depth). If not, add the import.
3. **Identify interactions** — Find every `onClick`, `onSubmit`, `onChange`, `onSelect`, `href`, `<Link>`, and `<a>` element.
4. **Cross-reference** — For each interaction, check whether a `trackEvent(...)` call already exists in or near the handler.
5. **Add missing calls** — For any interaction without tracking:
   - Derive an appropriate `action` name following the naming convention above.
   - Derive `category` from the feature/component name.
   - Derive `label` from context (route, entity, dynamic value) or omit.
   - Insert the `trackEvent(action, category, label)` call in the handler.
6. **Verify** — After edits, check for lint errors and confirm the import path resolves correctly.

## Audit Log

**Last audit: March 2026**

All known gaps have been resolved. The following files were instrumented or fixed:

| File | Changes |
|------|---------|
| `pages/assign-bulk-categories.jsx` | Added trackEvent for: run AI categorization, bulk accept all, select/deselect all, confidence filter change, bulk reject selected, bulk accept selected |
| `components/UsageBanner.jsx` | Added trackEvent for: subscribe, open cancel modal, confirm cancel, dismiss cancel, open increase limit, change cap tier, confirm increase, dismiss increase modal |
| `components/CompletenessScore.jsx` | Added trackEvent for: calculate score, retry, recalculate (stale + manual), toggle breakdown, factor action link clicks |
| `components/BulkOperationStatus.jsx` | Added trackEvent for: download run results s-link |
| `components/YoutubeMediaCard.jsx` | Added trackEvent for: s-clickable tutorial card clicks (both basic and standard variants) |
| `components/NeedHelpSection.jsx` | Added trackEvent for: email s-link, schedule meeting s-link |
| `components/TodoList.jsx` | Added trackEvent for: s-button bulk edit external link |
| `components/CategoryMetafieldToMigrateModal.jsx` | Added trackEvent for: LiquidJS docs s-link, Done s-button, View definition s-button |
| `pages/product-type-manager/invalid-product-types.jsx` | Fixed import: replaced `TrackGoogleAnalyticsEvent` with `trackEvent` |
| `components/ProductTypeActionsCard.jsx` | Removed unused `TrackGoogleAnalyticsEvent` import |
| `pages/settings/alerts.jsx` | Removed unused `TrackGoogleAnalyticsEvent` import |
| `pages/category-manager/[id].jsx` | Removed unused `TrackGoogleAnalyticsEvent` import |
| `pages/pricing.jsx` | No changes needed — all interactions delegated to UsageBanner (now instrumented) |

## Ongoing Maintenance

When new pages or components are added, re-run this agent to scan for uninstrumented interactions. Any `s-button`, `s-link`, `s-clickable`, `s-clickable-chip`, `s-select`, `s-checkbox`, `onClick`, `onSubmit`, `onChange`, `onSelect`, `href`, or `<Link>` without a nearby `trackEvent` call should be flagged and instrumented.

## Rules

- Never skip tracking on a user-visible interactive element.
- Prefer `trackEvent` over direct `TrackGoogleAnalyticsEvent` calls — `trackEvent` wraps both GA and Mantle.
- Do not add tracking to purely programmatic state changes (e.g. useEffect data fetches).
- Do not add tracking to scroll events, hover events, or focus events unless explicitly requested.
- Keep action names stable — renaming an action breaks analytics continuity. Only add new ones.
- When a component is reusable (e.g. `SearchBox`), the action scope should reflect where it is used. Pass a `trackingPrefix` prop if needed, or use a generic scope like `search_box__`.
- For hrefs that navigate away from the app (external links), use `onClick` with `trackEvent` and allow default navigation to proceed — do not call `e.preventDefault()`.
