# Execute-and-Heal Patterns

## Principle

Never commit tests that haven't been run. Infrastructure failures (wrong selectors,
wrong data shapes, missing auth, timing issues) are the pipeline's responsibility to
fix. Logic failures (wrong expected values) are the user's responsibility.

## Failure Classification

### Selector Miss

**Error patterns:**
- `Error: locator.click: Error: strict mode violation: getByRole('button', { name: '...' }) resolved to N elements`
- `Error: locator.toBeVisible: Error: locator resolved to 0 elements`

**Playwright error hints:** When strict mode fails, Playwright lists the matching elements
with suggested alternative selectors. Parse these hints — they often contain the exact
fix (e.g., `getByRole('tabpanel', { name: 'Workbook' }).getByLabel('Remove')`).

**MCP heal:** Use `browser_snapshot` to see the accessibility tree, find the ambiguous
elements, and construct a scoped selector.

**CLI-only heal:** Parse the error output for Playwright's selector suggestions.

### Data Shape

**Error patterns:**
- `TypeError: Cannot read properties of undefined (reading 'start')`
- `TypeError: seg.translations is not iterable`
- `Expected: "nǐ hǎo"` / `Received: undefined`

**Root cause:** Seed data uses wrong field names or wrong structure (e.g., per-item
entries instead of array-per-key for out-of-line IDB stores).

**MCP heal:** Use `browser_evaluate` to inspect an actual record from the IDB store
and compare field names with the seed data.

**CLI-only heal:** Read `src/types.ts` for the correct type definition and `src/db/index.ts`
for the store schema (keyPath, out-of-line keys).

### Auth/Setup

**Error patterns:**
- URL after navigation contains `/login`, `/auth`, `/unlock`
- Expected feature element not visible, but page contains auth form elements

**Root cause:** Tests don't bypass the auth gate.

**MCP heal:** `browser_evaluate` to inspect sessionStorage/localStorage for trial/bypass keys.

**CLI-only heal:** Grep for `sessionStorage.setItem`, `trial`, `bypass` in source.

### Timing

**Error patterns:**
- `Timeout 5000ms exceeded while waiting for...`
- `element is not visible` (intermittent)
- `element is disabled`

**Fixes:**
- Add `waitForResponse()` before the action that depends on an API call
- Increase assertion timeout: `expect(locator).toBeVisible({ timeout: 10_000 })`
- Add `waitFor` on a loading spinner to disappear

### Logic (DO NOT AUTO-HEAL)

**Error patterns:**
- `Expected: 3` / `Received: 0` (both defined, values differ)
- `Expected: "Saved to workbook"` / `Received: "Added to workbook"`

These indicate either:
1. The test's expected value is wrong (AC misunderstanding)
2. The application has a bug
3. The test setup doesn't create the expected state

In all cases, a human must decide. Flag and escalate.

## Heal Loop Control

- Max 3 iterations total (not 3 per test)
- Re-run only previously-failing tests in each iteration
- If a test changes category between iterations (e.g., selector_miss → timing), that's
  normal — the first fix revealed a deeper issue
- If the same test fails with the same error after a heal attempt, do not retry the
  same fix — escalate
