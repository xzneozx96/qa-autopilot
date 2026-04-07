---
name: qa-execute
description: >
  This skill should be used after qa-generate to run generated Playwright tests, classify failures,
  and self-heal infrastructure issues (selectors, data shapes, auth, timing).
  Phase 4.5 of the QA Agent pipeline. Does NOT heal logic failures.
---

# qa-execute — Test Runner & Self-Healer

Run generated tests before they reach review or commit. Classify failures by root
cause and auto-heal infrastructure issues. Escalate logic failures to the user.

## Position in Pipeline

```
qa-generate (Phase 4) → qa-execute (Phase 4.5) → qa-review-heal (Phase 5)
                              ↑        |
                              └────────┘  (heal loop, max 3 iterations)
```

## Inputs

| Input | Source |
|-------|--------|
| Generated test files | qa-generate (Phase 4) |
| `AdapterContext` | qa-adapter (Phase 0) |
| `LiveContext` | qa-live-explore (Phase 2), if available |
| `StructuredRequirement` | qa-intake (Phase 1) |

## Step 1: Run All Generated Tests

Execute the test suite and collect structured results:

```bash
npx playwright test {feature-dir} --reporter=json 2>&1
```

Parse the JSON reporter output to extract:
- Total test count, pass count, fail count
- For each failure: test title, error message, error stack, file path, line number

If Playwright MCP is available (`AdapterContext.playwrightMcpAvailable`), also keep
the browser session alive for interactive debugging of failures.

## Step 2: Classify Failures

Categorize each failure to determine the heal strategy:

| Category | Pattern Match | Heal Strategy |
|----------|--------------|---------------|
| **selector_miss** | "strict mode violation", "locator resolved to 0 elements", "locator resolved to N elements" | With MCP: `browser_snapshot` the page, find the correct unique selector. Without MCP: parse Playwright's error hint which often suggests the exact scoped selector. |
| **data_shape** | "Cannot read propert", "undefined is not an object", "Expected .* to be .* but received undefined" | With MCP: `browser_evaluate` to inspect IDB/app state and discover the actual field names. Without MCP: re-read TypeScript types from source and compare with seed data. |
| **auth_setup** | Page URL contains "login", "auth", "unlock"; expected element not found and page shows auth UI | With MCP: `browser_evaluate` to inspect sessionStorage/localStorage. Without MCP: grep source for auth bypass patterns. |
| **timing** | "Timeout .* waiting for", "element not visible", "element not enabled" | Add explicit `waitForResponse()` before the assertion, or increase timeout on the assertion. |
| **logic** | Assertion value mismatch where expected and actual are both defined (e.g., "expected 3, received 0") | **Do NOT auto-heal.** This may indicate an AC misunderstanding or a bug in the application. Flag for user review. |
| **unknown** | Does not match any pattern above | Log the full error. Do not attempt heal. Flag for user review. |

## Step 3: Heal and Re-run

For each healable failure (categories: `selector_miss`, `data_shape`, `auth_setup`, `timing`):

1. Read the failing test file.
2. Apply the targeted fix based on category:
   - **selector_miss:** Replace the failing selector with the verified unique selector.
     If MCP is available, use `browser_snapshot` to find it. If not, parse the Playwright
     error hint (Playwright often lists alternative selectors in its error output).
   - **data_shape:** Update the seed data helper to use correct field names and structure.
     If MCP is available, use `browser_evaluate` to inspect actual IDB records.
     If not, read `src/types.ts` and `src/db/index.ts` to find correct shapes.
   - **auth_setup:** Add `page.addInitScript()` to the test's `beforeEach` using the
     bypass snippet from `LiveContext.authBypass`. If no bypass known, add a setup step
     that navigates through the auth flow.
   - **timing:** Add `await page.waitForResponse()` or increase assertion timeout.
3. Re-run ONLY the previously-failing tests:
   ```bash
   npx playwright test {feature-dir} --grep "{failing-test-title}" --reporter=json 2>&1
   ```
4. If still failing, reclassify and try again.
5. **Hard cap: 3 iterations total.** After 3 rounds:
   - Commit the passing tests.
   - Write unhealed failures to `_qa-output/failing-tests.md`.
   - Each unhealed failure becomes an open risk in the gate decision.

## Step 4: Report

Display inline result (do not pause if all pass):

```
🧪 Execution result:
  Passed: {pass_count}/{total_count}
  Healed: {heal_count} in {iteration_count} round(s)
  Blocked: {blocked_count} → escalating to gate

  [if healed > 0]
  Heals applied:
    - {test_title}: {category} → {fix_description}

  [if blocked > 0]
  Unhealed failures:
    - {test_title}: {category} — {error_summary}
```

## Scope Boundaries

- Does **NOT** change test design — only fixes infrastructure (selectors, data, auth, timing).
- Does **NOT** auto-heal logic failures — these indicate AC misunderstandings or app bugs.
- Does **NOT** retry infinitely — hard cap at 3 heal iterations.
- Does **NOT** modify application source code — only test files and test helpers.

## Output

Return an `ExecutionResult` object conforming to `execute-output.schema.json`.

## Error Handling

| Failure | Action |
|---------|--------|
| Playwright not installed | Pipeline error. Should have been caught in Phase 0. Report and stop. |
| All tests pass on first run | Skip heal loop. Report success. |
| Dev server not running | Attempt to start using `AdapterContext.devServerCommand`. If fails, report and stop. |
| MCP not available during heal | Fall back to error-message-based healing (no live DOM inspection). |

## Knowledge References

- [`../knowledge/execute-heal.md`](../knowledge/execute-heal.md) — failure classification patterns and heal strategies
- [`../knowledge/selector-resilience.md`](../knowledge/selector-resilience.md) — selector quality hierarchy
- [`../knowledge/timing-debugging.md`](../knowledge/timing-debugging.md) — timing and wait patterns
