---
name: qa-risk
description: >
  This skill should be used when scoring a StructuredRequirement for risk. Runs in parallel with
  qa-visual and qa-explore during Phase 2. Outputs RiskContext with P0-P3
  priority, coverage gaps, and test-depth guidance.
---

# qa-risk

Score a requirement using **Probability x Impact** (1-9 scale), assign a P0-P3 priority, classify risk categories, and detect acceptance-criteria coverage gaps.

## Input

`StructuredRequirement` from qa-intake.

## Scoring

### Probability (1-3)

| Value | Label    | Signals                                    |
|-------|----------|--------------------------------------------|
| 1     | Unlikely | Simple, no dependencies, no failure history |
| 2     | Possible | Moderate complexity or some dependencies    |
| 3     | Likely   | High complexity, many dependencies, past failures |

### Impact (1-3)

| Value | Label    | Signals                                         |
|-------|----------|-------------------------------------------------|
| 1     | Minor    | Cosmetic, no functional loss                     |
| 2     | Degraded | Partial feature loss, workaround exists          |
| 3     | Critical | Blocker, security issue, data loss, revenue loss |

**Score = Probability x Impact** (range 1-9)

### Score to Priority

| Score | Priority | Action   | Gate Impact       |
|-------|----------|----------|-------------------|
| 9     | P0       | BLOCK    | Automatic FAIL    |
| 6-8   | P1       | MITIGATE | CONCERNS at gate  |
| 4-5   | P2       | MONITOR  | None              |
| 1-3   | P3       | DOCUMENT | None              |

### Priority to Test Depth

| Priority | Happy | Unhappy    | Edge     | Error |
|----------|-------|------------|----------|-------|
| P0       | All   | All        | All      | All   |
| P1       | All   | Key errors | Critical | Key   |
| P2       | Main  | Basic      | -        | Basic |
| P3       | Smoke | -          | -        | -     |

## Output

Return a `RiskContext` object:

- **score** — P x I (1-9)
- **probability** — 1, 2, or 3
- **impact** — 1, 2, or 3
- **priority** — P0 | P1 | P2 | P3
- **categories** — one or more of: TECH, SEC, PERF, DATA, BUS, OPS
- **action** — BLOCK | MITIGATE | MONITOR | DOCUMENT
- **justification** — one-sentence rationale
- **factors** — revenueImpact, userImpact, securityRisk, complianceRequired, previousFailure, complexity, usage
- **acGaps** — list of `{ acceptanceCriteriaId, description }` for missing coverage

## Example

**Requirement:** "Checkout with saved card"

- Probability: **3** (payment integrations, third-party dependencies, past gateway timeouts)
- Impact: **3** (revenue blocker, handles payment data)
- Score: **9** → **P0 BLOCK** — automatic gate FAIL, full test depth across all paths.

## Knowledge References

- [`../knowledge/probability-impact.md`](../knowledge/probability-impact.md)
- [`../knowledge/risk-governance.md`](../knowledge/risk-governance.md)
- [`../knowledge/test-priorities-matrix.md`](../knowledge/test-priorities-matrix.md)

## Rules

1. Always justify every score with concrete signals from the requirement.
2. Flag any acceptance criteria that lack testable conditions as `acGaps`.
3. Security and compliance factors automatically raise impact to 3.
4. When in doubt, score conservatively (higher).
