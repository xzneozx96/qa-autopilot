---
name: qa-live-explore
description: >
  This skill should be used when Playwright MCP is available to perform runtime inspection of the
  application under test. Discovers actual DOM structure, IDB schemas, auth
  bypass patterns, and verifies selector uniqueness. Phase 2 sub-agent —
  runs in parallel with qa-explore, qa-visual, and qa-risk.
---

# qa-live-explore — Runtime Application Inspector

Supplement static codebase analysis with live browser inspection. Uses Playwright MCP
tools to navigate the running application and extract runtime context that static
grep/glob cannot discover.

## Prerequisites

- `AdapterContext.playwrightMcpAvailable` MUST be `true`. If `false`, the orchestrator
  skips this agent entirely — do not attempt to run without MCP.
- Application MUST be running at `AdapterContext.devServerUrl`. If the orchestrator
  started a dev server, it will be available. If not, skip with warning.

## Inputs

| Input | Source |
|-------|--------|
| `AdapterContext` | qa-adapter (Phase 0) |
| `StructuredRequirement` | qa-intake (Phase 1) |

## Procedure

### 1. Navigate to Feature

Use `mcp__playwright__browser_navigate` to open `AdapterContext.devServerUrl`.
Then navigate to the feature page indicated by `StructuredRequirement` (e.g., the
lesson page for a vocabulary workbook feature).

If the app shows a login/auth gate:
- Check `sessionStorage`, `localStorage`, and cookies via `mcp__playwright__browser_evaluate`
  for trial/bypass flags.
- If a bypass is found, record the `addInitScript` snippet in `authBypass`.
- Apply the bypass and re-navigate.
- If no bypass found, record `authBypass.method: null` and attempt to proceed.

### 2. DOM Structure Inspection

Use `mcp__playwright__browser_snapshot` to capture the accessibility tree of the
feature area. For each interactive element relevant to the feature:

- Record the accessible name and how it is computed (explicit `aria-label`,
  `title` attribute, or inherited from children).
- Record nesting relationships — which `role="button"` containers wrap which
  inner buttons.
- Record which attributes are present: `data-testid`, `aria-label`, `title`, `role`.
- **Ambiguity detection:** For each accessible name, count how many elements share it.
  If count > 1, find a scoped selector that uniquely identifies each element
  (e.g., `page.getByRole('tabpanel', { name: /workbook/i }).getByLabel('Remove')`).

### 3. IDB Schema Discovery

Run via `mcp__playwright__browser_evaluate`:

```javascript
(async () => {
  const dbs = await indexedDB.databases();
  const result = {};
  for (const { name } of dbs) {
    const db = await new Promise((resolve, reject) => {
      const req = indexedDB.open(name);
      req.onsuccess = () => resolve(req.result);
      req.onerror = () => reject(req.error);
    });
    const stores = {};
    for (const storeName of db.objectStoreNames) {
      const tx = db.transaction(storeName, 'readonly');
      const store = tx.objectStore(storeName);
      const sample = await new Promise((resolve) => {
        const req = store.openCursor();
        req.onsuccess = () => resolve(req.result?.value ?? null);
      });
      stores[storeName] = {
        keyPath: store.keyPath,
        autoIncrement: store.autoIncrement,
        keyType: store.keyPath ? 'inline' : 'out-of-line',
        fields: sample ? Object.keys(sample) : [],
        indexNames: [...store.indexNames],
      };
    }
    result[name] = stores;
    db.close();
  }
  return result;
})();
```

Record the result per-store.

### 4. Auth Bypass Discovery

Via `mcp__playwright__browser_evaluate`, inspect:

```javascript
({
  sessionStorage: Object.fromEntries(
    Object.keys(sessionStorage).map(k => [k, sessionStorage.getItem(k)])
  ),
  localStorage: Object.fromEntries(
    Object.keys(localStorage).map(k => [k, localStorage.getItem(k)])
  ),
  cookies: document.cookie,
})
```

Look for keys containing `trial`, `bypass`, `auth`, `token`, `session`.
If found, construct an `addInitScript` snippet that sets the value before page load.

### 5. Selector Verification

For each selector found by static `qa-explore` (passed via `StructuredRequirement`
context or discovered in step 2):

- Use `mcp__playwright__browser_evaluate` to count matches:
  ```javascript
  document.querySelectorAll('[aria-label="Remove from Workbook"]').length
  ```
- Or use `mcp__playwright__browser_snapshot` and count occurrences of the accessible name.
- If match count !== 1, find a scoped alternative and record it as `verifiedSelector`.

## Output

Return a `LiveContext` object conforming to `live-explore-output.schema.json`.

## Error Handling

| Failure | Action |
|---------|--------|
| App not running | Skip all steps. Return `{ degraded: true }` with empty fields. |
| Auth gate blocks navigation | Record auth gate in `authBypass`, attempt bypass. If bypass fails, record what was found and proceed with partial results. |
| MCP tool call fails | Log error, skip that discovery step, continue with remaining steps. |
| IDB not used by app | Return empty `idbSchemas`. This is not an error. |

## Knowledge References

- [`../knowledge/live-exploration.md`](../knowledge/live-exploration.md) — DOM inspection patterns and IDB introspection techniques
- [`../knowledge/selector-resilience.md`](../knowledge/selector-resilience.md) — selector quality hierarchy
