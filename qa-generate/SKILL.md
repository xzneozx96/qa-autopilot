---
name: qa-generate
description: "This skill should be used when transforming designed test scenarios into production-quality Playwright .spec.ts files. Phase 4 of the QA Agent pipeline."
---

# qa-generate

Transform `TestScenarios[]` + `CodebaseContext` + `AdapterContext` into ready-to-run Playwright test files.

## Generation Rules

Every generated file MUST follow these rules without exception:

1. **Network-first:** Register `page.waitForResponse()` BEFORE `page.goto()` or `page.click()`. Never navigate then intercept.
2. **Selector hierarchy:** `data-testid` > ARIA role > accessible text > CSS class. Never use CSS class selectors.
3. **Deterministic waits:** Use `waitForResponse()`, `expect().toBeVisible()`, spinner disappearance. NEVER `waitForTimeout()`.
4. **Factory-based setup:** Seed data via API using factory functions. UI is for assertions only.
5. **Fixture composition:** Use `mergeTests()`. No inheritance chains.
6. **Auto-cleanup:** Track created resources, delete in fixture teardown.
7. **Tagging:** Every test gets `@p0`--`@p3` + `@smoke`/`@regression` in the title.
8. **Test ID:** `{EPIC}.{STORY}-E2E-{SEQ}` in the test title.
9. **No hardcoded data:** Use faker for all dynamic values. No static emails or IDs.
10. **Co-located helpers:** Inline for 1 use, utility for 2--3, fixture for 3+.
11. **Playwright prerequisite:** Playwright MUST already be installed by `qa-adapter` (Phase 0). If `@playwright/test` is not present at this stage, this is a pipeline error — stop and report. If faker is missing, add `@faker-js/faker`.
12. **No seeding API fallback:** If no seeding endpoints are found, generate UI-based setup and flag as tech-debt.

## Generated File Structure

```
tests/
├── e2e/
│   ├── {feature}/
│   │   ├── {feature}.spec.ts
│   │   ├── {feature}.fixtures.ts
│   │   └── {feature}.factories.ts
│   └── support/
│       ├── fixtures/
│       ├── factories/
│       └── helpers/
├── playwright.config.ts
└── package.json
```

## Knowledge Fragments

Read these for detailed patterns before generating:

- [`network-first.md`](../knowledge/network-first.md)
- [`data-factories.md`](../knowledge/data-factories.md)
- [`fixture-architecture.md`](../knowledge/fixture-architecture.md)
- [`intercept-network-call.md`](../knowledge/intercept-network-call.md)

## Template References

Read these templates for canonical code patterns:

- [`../templates/test-file.md`](../templates/test-file.md) -- test structure pattern
- [`../templates/fixture-file.md`](../templates/fixture-file.md) -- fixture composition pattern
- [`../templates/factory-file.md`](../templates/factory-file.md) -- data factory pattern

## Workflow

1. Verify Playwright is installed (rule 11 — pipeline error if missing). Verify faker; install if missing.
2. If `LiveContext` is available (non-degraded), read `idbSchemas` for seed data field names and `verifiedSelectors` for unique selectors. These override any static assumptions.
3. Read template references for structure patterns.
4. Generate factory files from `AdapterContext` seeding endpoints; fall back to UI setup if none exist (rule 12).
5. Generate fixture files using `mergeTests()` composition (rule 5) with auto-cleanup teardown (rule 6).
6. Generate spec files applying all 12 rules. Tag every test (rules 7--8), use faker for data (rule 9), register network intercepts before actions (rule 1).
7. Place files according to the generated file structure above.
