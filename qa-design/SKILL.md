---
name: qa-design
description: "This skill should be used when designing concrete test scenarios from merged requirement, Figma, codebase, and risk context. Phase 3 of the QA Agent pipeline."
---

# qa-design

Design concrete test scenarios from merged context. Each scenario maps to one or more acceptance criteria and specifies exactly what to test.

## Inputs

`StructuredRequirement` + `VisualContext` (nullable) + `CodebaseContext` + `RiskContext`

## Design Rules

1. Every AC must have at least one scenario (traceability).
2. Scenario depth is driven by risk priority (P0 = comprehensive, P3 = smoke).
3. Use real selectors from `qa-explore` -- never invent selectors.
4. Reference real API routes for network-first patterns.
5. Specify data setup via existing factories.
6. Each scenario has: precondition, steps, expected result, priority tag.

## Priority Depth Table

| Priority | Happy | Unhappy | Edge | Error | NFR |
|----------|-------|---------|------|-------|-----|
| P0 | All | All | All | All | Yes |
| P1 | All | Key | Critical | Key | If applicable |
| P2 | Main | Basic | -- | Basic | -- |
| P3 | Smoke | -- | -- | -- | -- |

## Scenario Types

- **Happy path** -- primary user flow succeeds.
- **Unhappy path** -- invalid input, system handles gracefully.
- **Edge case** -- boundary conditions, empty states, max limits.
- **Error recovery** -- network failure, timeout, server error leading to graceful degradation.
- **NFR** -- auth bypass attempt, concurrent access, slow network.

## Test ID Format

`{EPIC}.{STORY}-E2E-{SEQ}` (e.g., `1.3-E2E-001`)

## Output Schema

Each scenario must conform to `TestScenario`:

- `id` -- test ID per format above.
- `title` -- concise description.
- `acceptanceCriteriaIds` -- traced back to requirement ACs.
- `type` -- one of `happy`, `unhappy`, `edge`, `error`, `nfr`.
- `priority` -- `P0` through `P3`.
- `tags` -- e.g., `["@p0", "@smoke", "@checkout"]`.
- `preconditions` -- each with `description`, `setupMethod` (`factory` | `api` | `fixture` | `navigation`), optional `factoryName` or `apiEndpoint`.
- `steps` -- each with `action`, optional `selector` and `selectorQuality`, optional `networkIntercept` (`method`, `url`, `expectedStatus`).
- `expectedResults` -- each with `assertion`, optional `selector` and `value`.

## Figma Context Handling

When `VisualContext` is null, rely solely on `CodebaseContext` selectors. When both are available, prefer Figma-suggested `testId` values mapped to actual codebase selectors found by `qa-explore`.

## Knowledge Fragments

- `test-levels-framework.md`
- `test-priorities-matrix.md`
- `nfr-criteria.md`
