---
name: qa-trace
description: >
  This skill should be used when building a traceability matrix linking acceptance criteria
  to tests, validate coverage, and produce an objective gate decision.
---

# QA Trace — Traceability & Gate Decision

Phase 6 of the QA Agent pipeline. Builds bidirectional AC-to-Test traceability,
validates coverage, and returns a structured gate decision.

## Inputs

| Input                    | Source              |
|--------------------------|---------------------|
| `ReviewedTestFile[]`     | qa-review phase     |
| `StructuredRequirement`  | qa-intake phase     |
| `RiskContext`            | qa-risk phase       |
| `ExecutionResult`        | qa-execute phase    |

## Process

1. **Map** every acceptance criterion to the test(s) that exercise it.
2. **Reverse-map** every test back to its criterion to detect orphan tests.
3. **Compute** `coverageRate` = covered criteria / total criteria.
4. **Incorporate execution results:** If `ExecutionResult` is provided:
   - Each unresolved failure becomes an open risk with score based on category:
     `logic` → score 8, `unknown` → score 7, infrastructure categories → score 5.
   - Healed tests are noted in recommendations (not blocking).
   - `allPassed: false` prevents PASS — minimum gate is CONCERNS.
5. **Evaluate** gate decision using the logic below.
6. **Emit** `GateDecision` output.

## Gate Decision Logic

Evaluated in precedence order — first match wins.

**FAIL** if ANY of:
- Any risk with score = 9 is OPEN
- Any P0 acceptance criterion has no test
- Review score < 60
- Any test has `unresolvedFailures` with category `logic` or `unknown`

**CONCERNS** if ANY of:
- Risks with score 6-8 exist (even with mitigation)
- P1 acceptance criteria have gaps
- Review score 60-79
- Any test was healed (indicates fragile generation — not blocking but noteworthy)

**PASS** if ALL of:
- All P0/P1 acceptance criteria covered
- No open critical/high risks
- Review score >= 80

Precedence: **FAIL > CONCERNS > PASS**

## Waiver Rules

- Each waiver requires: **approver name**, **reason**, and **expiry date**.
- A waived item moves from FAIL to CONCERNS (never directly to PASS).
- Expired waivers automatically revert to FAIL.

## Output Schema

Returns `GateDecision`:

```typescript
type GateDecision = {
  decision: 'PASS' | 'CONCERNS' | 'FAIL';
  timestamp: string;
  traceabilityMatrix: Array<{
    criterionId: string;
    criterion: string;
    priority: string;
    tests: string[];
    covered: boolean;
    waiverReason?: string;
  }>;
  coverageRate: number;
  reviewScore: number;
  openRisks: Array<{ score: number; title: string; category: string }>;
  recommendations: string[];
  summary: string;
};
```

## Templates

- [`../templates/risk-report.md`](../templates/risk-report.md) — risk report output format
- [`../templates/trace-matrix.md`](../templates/trace-matrix.md) — traceability matrix output format

## Knowledge

- `risk-governance.md`
- `adr-quality-readiness-checklist.md`
