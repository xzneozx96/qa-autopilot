---
name: qa-selective
description: This skill should be used when determining which tests to run at each stage of the development lifecycle, generating package.json scripts, or configuring CI pipelines based on test tags and diff context.
knowledge:
  - selective-testing.md
  - test-priorities-matrix.md
---

# QA Selective — Test Promotion Rules

Phase 7 of the QA Agent pipeline. Defines which tests run at each stage, generates package.json scripts, and produces CI configuration based on test tags and AdapterContext.

## Inputs

- Generated test tags from prior pipeline phases
- AdapterContext (framework, runner, project structure)

## Promotion Stages

| Stage | Tests | Time Budget | Blocks | Failure Action |
|-------|-------|-------------|--------|----------------|
| Pre-commit | @smoke | < 2 min | Commit | Block |
| PR | changed + @p0 + @p1 | < 10 min | Merge | Block |
| Merge to main | @regression (full) | < 30 min | Deploy | Block |
| Staging | @smoke E2E | < 15 min | Release | Rollback |
| Production | @p0 @smoke | < 5 min | — | Alert team |

## Generated Scripts

```json
{
  "test:smoke": "playwright test --grep @smoke",
  "test:p0": "playwright test --grep @p0",
  "test:p0-p1": "playwright test --grep '@p0|@p1'",
  "test:regression": "playwright test",
  "test:changed": "bash scripts/test-changed-files.sh"
}
```

## Diff-Based Selection Logic

| Changed artifact | Tests to run |
|-----------------|--------------|
| Test file | Run it directly |
| Component | Find related specs by name |
| API route | Integration + E2E specs |
| Config / dependency | Full regression |
| Docs only | Skip tests |

## Knowledge Fragments

- `selective-testing.md` — strategies for scoping test runs to changed surface area
- `test-priorities-matrix.md` — tag definitions and priority tiers (@smoke, @p0, @p1, @regression)
