---
name: qa-adapter
description: >-
  This skill should be used when starting a new QA engagement on a project, when no AdapterContext
  exists yet, or when the project's tech stack may have changed. Detects
  frontend/backend frameworks, component library, package manager, selector
  conventions, existing test setup, CI platform, and base URL.
---

# QA Adapter — Tech Stack Detection

Phase 0 skill. Runs first to produce an `AdapterContext` that all downstream skills consume.

## When to Use

- First run on any project (no `AdapterContext` cached)
- After major dependency upgrades
- When another skill reports a context mismatch

## Discovery Process

Work through each step using Glob, Grep, and Read. Do not install anything.

### 1. Package Manager & Dependencies

- Glob for `pnpm-lock.yaml`, `yarn.lock`, `bun.lockb`, `package-lock.json` to determine package manager.
- Read `package.json` — extract `dependencies`, `devDependencies`, and `scripts`.

### 2. Frontend Framework

- Check dependencies for `react`, `vue`, `@angular/core`, `svelte`.
- Check for `next.config.*` (Next.js) or `nuxt.config.*` (Nuxt).

### 3. Backend Framework

- Grep for framework imports: `fastapi`, `express`, `@nestjs/core`, `django`, `rails`.
- Check for `requirements.txt`, `pyproject.toml`, `Gemfile` if no JS backend found.

### 4. Component Library

- Search dependencies for `@shadcn`, `@mui/material`, `antd`, `bootstrap`.
- Glob for `components/ui/` (shadcn convention).

### 5. API Client

- Grep source files for `axios`, `tRPC`, `@apollo/client`, `graphql-request`, or raw `fetch(` calls.

### 6. Auth Pattern

- Grep for `jwt`, `cookie`, `session`, `oauth`, `next-auth`, `passport` in source and config.

### 7. Selector Conventions

- Grep for `data-testid` across all source files. Count occurrences.
- Grep for `role=` and `aria-` attributes. Count occurrences.
- Classify: **high** (>60% of components), **medium** (20-60%), **low** (<20%), **none**.
- Recommend: prefer `data-testid` if high coverage, else prefer ARIA roles, else suggest adding `data-testid`.

### 8. Existing Test Setup

- Check `devDependencies` for `@playwright/test`.
- Glob for `playwright.config.*`.
- Glob for `**/*.spec.ts`, `**/*.test.ts`, `**/e2e/**` — count files.
- Read a few existing specs to note conventions (naming, fixtures, POM usage).

### 9. CI Platform

- Glob for `.github/workflows/*.yml`, `.gitlab-ci.yml`, `Jenkinsfile`, `.circleci/config.yml`.

### 10. Base URL

- Check `package.json` scripts for `--port` or `dev` command hints.
- Grep `.env*` files for `PORT`, `BASE_URL`, `VITE_` prefixes.
- Check framework config (`vite.config.*`, `next.config.*`).
- Fallback: `http://localhost:3000`.

### 11. Playwright Enforcement (Hard Gate)

This pipeline ONLY produces Playwright E2E tests. It does NOT write unit tests, integration tests, or vitest/pytest specs — ever. Detecting vitest, pytest, jest, or other frameworks is informational only.

- If `existingTestSetup.playwrightInstalled` is `false`:
  1. Scaffold Playwright immediately using the detected package manager:
     - pnpm: `pnpm create playwright`
     - npm: `npm init playwright@latest`
     - yarn: `yarn create playwright`
     - bun: `bunx create-playwright`
  2. Verify the scaffold succeeded by checking `node_modules/@playwright/test` exists.
  3. If scaffolding fails → **FAIL the pipeline** with message:
     ```
     ❌ Playwright scaffolding failed. Cannot proceed.
     Install manually: {packageManager} add -D @playwright/test
     ```
  4. Do NOT fall back to vitest, pytest, jest, or any other framework.

### 12. Playwright MCP Detection (Soft Check)

- Attempt to detect if `mcp__playwright__browser_navigate` tool is available.
- Set `playwrightMcpAvailable: true` if detected, `false` otherwise.
- If not available, emit warning (do not block):
  ```
  ⚠ Playwright MCP not detected. Running in degraded mode:
    - Live exploration skipped → using static analysis only
    - Execute phase will run tests via CLI, self-heal from error messages only
    For full capability: claude mcp add playwright npx @playwright/mcp@latest
  ```

### 13. Dev Server Detection

Detect how to start the application for live exploration:

1. Read `package.json` `scripts` for entries named `dev`, `start`, or `serve`.
2. Probe common ports (`localhost:3000`, `localhost:5173`, `localhost:4200`) for an already-running server.
3. Record the dev server command and detected/configured port.
4. If no server is running and a dev command is found, set `devServerCommand` for Phase 2 to use.
5. If no dev command found, set `devServerCommand: null` — live explore will be skipped.

## Output Schema

Produce a JSON object conforming to `AdapterContext`:

```typescript
type AdapterContext = {
  framework: string;
  backendFramework: string;
  componentLibrary: string;
  packageManager: 'npm' | 'yarn' | 'pnpm' | 'bun';
  apiClient: string;
  authPattern: string;
  selectorStrategy: {
    dataTestIdCoverage: 'high' | 'medium' | 'low' | 'none';
    ariaRoleCoverage: 'high' | 'medium' | 'low';
    recommendation: string;
  };
  existingTestSetup: {
    playwrightInstalled: boolean;
    configExists: boolean;
    existingSpecCount: number;
    conventions: string[];
  };
  ciPlatform: string;
  baseUrl: string;
  playwrightMcpAvailable: boolean;
  devServerCommand: string | null;   // e.g. "pnpm dev" or null if not found
  devServerUrl: string | null;       // e.g. "http://localhost:5173" or null
};
```

## Knowledge References

- [Overview](../knowledge/overview.md) — framework architecture and skill pipeline
- [Playwright Config](../knowledge/playwright-config.md) — config patterns and best practices
