---
name: qa-explore
description: >
  This skill should be used when scanning a target codebase to discover selectors, API routes,
  factories, fixtures, auth patterns, and test conventions before generating tests.
  Phase 2 sub-agent — runs in parallel with other Phase 2 skills.
---

# qa-explore — Codebase Scanner

Scan the target codebase to ground generated tests in real code. Receives a project root, StructuredRequirement (feature focus), and AdapterContext (framework patterns).

## Discovery Targets

1. **Selectors** — Find `data-testid` attributes in components related to the feature. Score each by quality.
2. **API routes** — Find endpoint definitions matching the AdapterContext backend framework. Extract URL patterns, methods, request/response shapes.
3. **Existing factories** — Find `create*` factory functions and seed helpers (`*.factory.ts` files).
4. **Existing fixtures** — Find Playwright fixture files (`*.fixtures.ts`), understand `mergeTests` composition.
5. **Auth patterns** — Determine auth type (cookie, JWT, OAuth, session). Locate login flow and any API shortcut for test setup.
6. **Test conventions** — Naming patterns, folder structure, tag usage (`@smoke`, `@critical`), spec count.

## Selector Quality Scoring

| Example                            | Quality      |
|------------------------------------|-------------|
| `data-testid="submit-btn"`        | EXCELLENT   |
| `role="button" name="Submit"`     | GOOD        |
| `text="Submit"`                   | ACCEPTABLE  |
| `.btn-primary`                    | POOR        |

Prefer higher-quality selectors. Flag components that lack `data-testid` as improvement opportunities.

## Procedure

1. **Scope** — Use the StructuredRequirement to identify relevant source directories and components.
2. **Selectors** — Grep for `data-testid=` in scoped component files. Also scan for `role=`, `aria-label=`, and CSS-class-only selectors.
3. **API routes** — Grep for route decorators based on AdapterContext.backendFramework (e.g., `@app.get|post|put|delete` for FastAPI, `router.get|post` for Express).
4. **Factories** — Glob for `*.factory.ts`, `*.factory.js`, and grep for `function create` or `export const create`.
5. **Fixtures** — Glob for `*.fixtures.ts` and grep for `mergeTests`.
6. **Auth** — Search for login endpoints, token storage, `storageState`, cookie-setting middleware.
7. **Conventions** — Glob for `*.spec.ts` to assess count, naming, and folder layout. Grep for `@tag` annotations.

## Output

Return a `CodebaseContext` object containing `selectors`, `apiRoutes`, `existingFactories`, `existingFixtures`, `authPattern`, `testConventions`, and `selectorWarning`. See the schema in `../../schemas/` for the full TypeScript type definition.

`selectorWarning` is set when `data-testid` coverage is `"none"` across all scanned components:

```json
{
  "selectorWarning": "Zero data-testid attributes found in scoped components. Generated selectors will fall back to ARIA roles and CSS classes, which are more fragile. Consider adding data-testid attributes to interactive elements before running tests in production."
}
```

Leave `selectorWarning` null when any `data-testid` attributes are found.

## Knowledge References

- [`../knowledge/selector-resilience.md`](../knowledge/selector-resilience.md) — selector strategy and fallback hierarchy
- [`../knowledge/fixture-architecture.md`](../knowledge/fixture-architecture.md) — Playwright fixture composition patterns
- [`../knowledge/data-factories.md`](../knowledge/data-factories.md) — factory and seed helper conventions


## Merge Protocol with Live Context

When `qa-live-explore` runs in parallel (Phase 2), the orchestrator merges both outputs.
The merge rules are:

| Field | Merge Rule |
|-------|-----------|
| `selectors` | If `LiveContext.verifiedSelectors` has an entry for a static selector, replace the static selector with the verified one. Keep static selectors that have no live counterpart. |
| `apiRoutes` | Static-only. Live explore does not discover API routes. |
| `existingFactories` | Static-only. |
| `existingFixtures` | Static-only. |
| `authPattern` | If `LiveContext.authBypass.method` is not null, add the bypass snippet to `authPattern`. |
| `testConventions` | Static-only. |

The merged `CodebaseContext` is passed to `qa-design` and `qa-generate`.
`LiveContext.idbSchemas` and `LiveContext.domSnapshots` are passed as separate inputs
to `qa-generate` for seed data generation and selector strategy.
