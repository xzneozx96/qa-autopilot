---
name: qa-ci
description: This skill should be used when generating a GitHub Actions CI workflow for Playwright test suites — includes burn-in testing, parallel sharding, artifact upload, caching, environment configuration, and security hardening.
---

# QA CI Pipeline Scaffold

Generate a GitHub Actions workflow from `AdapterContext` + `SelectiveTestingConfig`. Falls back to GitHub Actions if `AdapterContext.ciPlatform` is unknown.

## Knowledge Fragments

- `ci-burn-in.md` — burn-in loop strategy and flakiness thresholds
- `burn-in.md` — repeat-run mechanics for new/changed specs
- `playwright-config.md` — timeout defaults, shard configuration, reporter setup

## Inputs

| Field | Source |
|---|---|
| `ciPlatform` | `AdapterContext.ciPlatform` |
| `changedSpecs` | `SelectiveTestingConfig.changedSpecs` |
| `shardCount` | `SelectiveTestingConfig.shardCount` (default: 4) |
| `environments` | `AdapterContext.environments` (local / staging / production) |

## CI Features

- **Burn-in**: Run new/changed specs 10x before merge to catch flaky tests early
- **Sharding**: Split full regression across N parallel runners (default 4)
- **Artifacts**: Upload screenshots, videos, traces, and HAR files on failure
- **Caching**: Cache `node_modules` and Playwright browser binaries by lockfile hash
- **Timeouts**: `actionTimeout: 15s`, `navigationTimeout: 30s`, `expect: 10s`
- **Environment config**: Parameterised `BASE_URL` for local, staging, and production

## Workflow Stages

```yaml
PR opened/updated:
  → Smoke tests (@smoke, target < 5 min)
  → Changed file tests (diff-based, from SelectiveTestingConfig)
  → P0 + P1 tests

Merge to main:
  → Full regression (all tests, sharded across N runners)
  → Burn-in for new/changed specs (10 repeat runs)

Staging deploy:
  → E2E smoke against staging BASE_URL

Production deploy:
  → Critical smoke (@p0 @smoke, non-blocking)
  → Alert on failure (Slack / PagerDuty webhook)
```

## Security

Never interpolate PR titles, branch names, or commit messages directly into `run:` steps. Use `env:` to pass untrusted values as environment variables — this prevents script injection via attacker-controlled ref names.

```yaml
# Safe
- run: echo "$PR_TITLE"
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}

# Unsafe — do not do this
- run: echo "${{ github.event.pull_request.title }}"
```

## Output

A `.github/workflows/qa.yml` file structured around the stages above, with jobs for `smoke`, `changed`, `regression`, `burn-in`, `staging-smoke`, and `production-smoke`. Artifact upload steps use `actions/upload-artifact` scoped to failure only (`if: failure()`).
