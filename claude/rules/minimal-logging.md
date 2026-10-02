---
description: "Do not add info/trace logs unless explicitly requested; only warn, error, or debug"
---

# Minimal Logging

Do **not** add logging unless it is a warning, an error, a debug log, or the user explicitly asked for logs in the current message.

## Allowed without an explicit request

- `warn` / `warning` — unexpected but recoverable conditions
- `error` — failures in catch blocks or hard failures (always log caught errors)
- `debug` — temporary or gated diagnostic detail behind a debug level/flag

## Forbidden without an explicit request

- `info` / `log` / `trace` / routine progress messages
- "dropping X", "cache hit", "starting Y", "finished Z" style breadcrumbs
- New console or logger calls that only narrate happy-path control flow

```typescript
// BAD — noisy info / progress logging
console.log("[brand-voice] dropping incomplete descriptor", { title });
logger.info("source cache hit", { clientId });

// GOOD — error or warn only when something is wrong
catch (error) {
  logger.error("Failed to load brand voice", { clientId, error });
  throw error;
}

logger.warn("Brand voice descriptor missing whenAndWhere", { title });
```

## Exceptions

- The user explicitly asks to add, keep, or increase logging
- An existing project convention requires a specific log (follow the codebase; do not invent new info logs nearby)
- Do not remove existing logs unless asked; this rule constrains **adding** new ones
