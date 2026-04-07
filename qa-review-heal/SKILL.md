---
name: qa-review-heal
description: >
  This skill should be used when generated test files need quality review and auto-healing.
  Reviews tests across 6 dimensions and fixes detected anti-patterns in-place.
---

# QA Review + Auto-Heal

Phase 5 of the QA Agent pipeline. Reviews generated tests for quality and automatically heals detected anti-patterns by editing files in-place.

## Inputs / Outputs

- **Inputs:** `GeneratedTestFile[]` + `StructuredRequirement` (for AC mapping in coverage dimension)
- **Outputs:** `ReviewedTestFile[]` (same file paths, healed content) + `ReviewReport`
- **Edit mode:** Auto-heal uses the Edit tool to patch generated files in-place.

## 6 Review Dimensions

| # | Dimension | Checks | Auto-heal |
|---|-----------|--------|-----------|
| 1 | Resilience | Selectors stable? Timeouts? Network patterns? | CSS -> data-testid. Add timeouts. |
| 2 | Determinism | No race conditions? No hard waits? | waitForTimeout -> waitForResponse. Reorder intercept before navigate. |
| 3 | Isolation | Stateless? Self-cleaning? No shared mutable state? | Add fixture teardown. Remove globals. |
| 4 | Coverage | Every AC mapped? Depth matches priority? | Flag unmapped AC. Suggest missing scenarios. |
| 5 | Maintainability | No duplication? Clear intent? | Extract repeated patterns into fixtures (3+ rule). |
| 6 | Performance | No unnecessary waits? Parallel-safe? | Remove redundant waits. UI setup -> API setup. |

## Anti-Pattern Healing

| Detected anti-pattern | Healed to |
|-----------------------|-----------|
| `page.waitForTimeout(N)` | `page.waitForResponse()` or `expect().toBeVisible()` |
| `.className` selector | `getByTestId()` or `getByRole()` |
| Navigate then intercept | Intercept then navigate |
| UI-based data setup | API-based factory setup |
| Hardcoded test data | Faker-based factory |
| Global state mutation | Fixture-scoped state |
| Missing cleanup | Auto-cleanup in fixture teardown |

## Quality Gate

A test file passes the quality gate when:

- Overall score >= **80** (out of 100)
- **Zero** critical-severity issues remaining

## Output Schema

```typescript
type ReviewReport = {
  overallScore: number;            // 0-100
  dimensions: Array<{
    name: string;
    score: number;
    issues: Array<{
      severity: 'critical' | 'warning' | 'info';
      description: string;
      file: string;
      line: number;
      autoHealed: boolean;
      healAction?: string;
    }>;
  }>;
  healedCount: number;
  remainingIssues: number;
  passesQualityGate: boolean;     // true if score >= 80 and no critical issues
};
```

## Knowledge Fragments

- [`test-quality.md`](../knowledge/test-quality.md)
- [`test-healing-patterns.md`](../knowledge/test-healing-patterns.md)
- [`timing-debugging.md`](../knowledge/timing-debugging.md)
- [`selector-resilience.md`](../knowledge/selector-resilience.md)
