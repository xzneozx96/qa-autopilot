# Traceability Matrix Template

Reference template for qa-trace skill output.

## Format

```markdown
# Traceability Matrix — {Feature Name}

**Generated:** {timestamp}
**Gate Decision:** {PASS / CONCERNS / FAIL}
**Coverage Rate:** {percentage}%
**Review Score:** {0-100}

## AC ↔ Test Mapping

| AC ID | Acceptance Criterion | Priority | Tests | Status |
|-------|---------------------|----------|-------|--------|
| AC-001 | {criterion text} | P0 | `{feature}.spec.ts:test-001` | ✅ Covered |
| AC-002 | {criterion text} | P0 | `{feature}.spec.ts:test-002, test-003` | ✅ Covered |
| AC-003 | {criterion text} | P1 | — | ❌ GAP |

## Open Risks

| Score | Risk | Category | Impact on Gate |
|-------|------|----------|---------------|
| {score} | {description} | {category} | {FAIL/CONCERNS/none} |

## Gate Decision Detail

**Decision: {PASS / CONCERNS / FAIL}**

**Reason:**
{Explanation of why this decision was reached}

**Conditions met:**
- {List of conditions that were evaluated}

## Waivers (if any)

| AC/Risk | Approver | Reason | Expiry |
|---------|----------|--------|--------|
| {item} | {name} | {why waived} | {date} |

## Recommendations

{Numbered list of actions to improve coverage or reduce risk}
```
