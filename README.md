# QA Autopilot

> **From user story to production-ready Playwright tests — fully automated.**

QA Autopilot is a Claude Code skill that orchestrates a complete, multi-phase QA pipeline. Drop in a user story or ticket description and get back battle-hardened Playwright tests, a risk report, a traceability matrix, a gate decision, and CI configuration — all without leaving your editor.

---

## Requirements

- [Claude Code](https://claude.ai/code) CLI
- Claude API access (Sonnet + Opus)
- Node.js 18+ (for Playwright)
- Git (for the commit step)

## Installation

```bash
git clone https://github.com/xzneozx96/qa-autopilot.git
cp -r qa-autopilot ~/.claude/skills/qa-autopilot
```

Claude Code auto-discovers skills in `~/.claude/skills/` — no restart required.

> **To update later:**
> ```bash
> cd qa-autopilot && git pull
> cp -r . ~/.claude/skills/qa-autopilot
> ```

---

## Usage

Invoke from any project in Claude Code:

```
As a user, I can reset my password via email link so that I can regain access to my account.

AC:
- Given I am on the login page, when I click "Forgot password", then I see the reset form
- Given I submit a valid email, when the server responds, then I receive a confirmation message
- Given I click the reset link in the email, when the token is valid, then I can set a new password
- Given I submit a new password, when it meets requirements, then my password is updated and I am redirected to login

Figma: https://figma.com/design/abc123/reset-flow    (optional)
```

The pipeline runs end-to-end, pausing only at 3 hard checkpoints:
- **Checkpoint A** — requirement gaps (auto-skipped for warnings)
- **Checkpoint C** — scenario approval (always stops for your sign-off)
- **Checkpoint D** — gate decision (always stops before committing)

---

## What it does

```
User story + ACs  ──▶  QA Autopilot  ──▶  Playwright tests
                                          Risk report
                                          Traceability matrix
                                          Gate decision (PASS / CONCERNS / FAIL)
                                          CI workflow
                                          package.json test scripts
```

The skill runs a 7-phase pipeline coordinated by an orchestrator that spawns specialized sub-agents for each job. Reviewer phases use the more powerful `opus` model; implementation phases use `sonnet` for speed and cost efficiency.

---

## Architecture

QA Autopilot is a **multi-agent pipeline** — not a single monolithic prompt. The orchestrator (`SKILL.md`) spawns specialized sub-agents for each phase, passing typed JSON contracts between them. Reviewer phases use `opus` for deep evaluation; implementation phases use `sonnet` for speed. Parallel phases run as concurrent sub-agents in a single dispatch.

```
qa-autopilot/
├── SKILL.md                    ← Orchestrator: coordinates the full pipeline
│
├── qa-adapter/                 ← Phase 0:   Tech stack detection
├── qa-instrument/              ← Phase 0.5: Codebase instrumentation (conditional)
├── qa-intake/                  ← Phase 1:   Requirements parsing
│
├── qa-visual/                  ← Phase 2a ─┐
├── qa-explore/                 ← Phase 2b  ├── Parallel context discovery
├── qa-risk/                    ← Phase 2c  ┤  (opus for risk)
├── qa-live-explore/            ← Phase 2d ─┘  (conditional: dev server)
│
├── qa-design/                  ← Phase 3:   Test scenario design
├── qa-generate/                ← Phase 4:   Playwright file generation
├── qa-execute/                 ← Phase 4.5: Test execution + self-heal loop
├── qa-review-heal/             ← Phase 5:   Quality review + auto-heal (opus)
├── qa-trace/                   ← Phase 6:   Traceability + gate decision (opus)
│
├── qa-ci/                      ← Phase 7a ─┐  Parallel: CI + scripts
├── qa-selective/               ← Phase 7b ─┘
│
├── knowledge/                  ← 21 domain knowledge files, loaded on demand
├── schemas/                    ← 11 JSON schemas — typed contracts between agents
└── templates/                  ← Canonical output templates (test, factory, fixture, reports)
```

---

## Pipeline deep-dive

### Phase 0 — Tech Stack Detection (`qa-adapter`, sonnet)

The pipeline starts blind. `qa-adapter` inspects the project without installing anything, using only Glob, Grep, and Read:

- **Package manager** — detects pnpm / yarn / bun / npm from lockfiles
- **Frontend framework** — React, Vue, Angular, Svelte; sub-detects Next.js, Nuxt, SvelteKit, Remix
- **Backend framework** — Express, NestJS, FastAPI, Django, Rails (from deps and import patterns)
- **Component library** — shadcn/ui, MUI, Ant Design, Bootstrap
- **API client** — Axios, tRPC, Apollo, raw fetch
- **Auth pattern** — JWT, cookie, session, OAuth, next-auth
- **Selector coverage** — counts `data-testid` and ARIA attributes across source files, classifies as `none / low / medium / high`
- **Existing test setup** — finds `playwright.config.ts`, existing spec files, custom fixtures
- **Dev server** — infers start command and base URL
- **CI platform** — detects GitHub Actions, GitLab CI, CircleCI from config files

Output: `AdapterContext` — the shared foundation every downstream agent reads.

---

### Phase 0.5 — Codebase Instrumentation (`qa-instrument`, sonnet) — CONDITIONAL

Only fires when `AdapterContext.selectorStrategy.dataTestIdCoverage` is `"none"` or `"low"`. Skipped entirely for `"medium"` or `"high"`.

Scopes to components referenced by the user story, then adds `data-testid` attributes following a strict naming convention:

```
{feature}-{element-type}
```

Examples: `admin-login-email-input`, `checkout-submit-button`, `order-form-error`.

Rules enforced: kebab-case only · no color/size/position descriptors · no component library internals · element-type suffix required. Changes are committed as a separate `chore:` commit before test generation begins, so test history stays clean.

If instrumentation fails (read-only files, unknown framework), the pipeline continues with ARIA fallbacks — selectors are flagged as tech-debt in test comments.

---

### Phase 1 — Requirements Parsing (`qa-intake`, sonnet)

Parses any free-form input — user story, Jira ticket, PR description, plain English — into a machine-readable `StructuredRequirement`:

- **User story** → extracts `actor`, `action`, `value`
- **Acceptance criteria** → normalises each AC into `precondition → action → expectedResult` triple, tagged `functional` or `nfr`
- **NFR criteria** → classifies security, performance, reliability, maintainability requirements with thresholds
- **Gap detection** → flags `blocking` gaps (vague success criteria, missing error paths, no actor) that must be resolved before continuing; flags `warning` gaps (happy-path-only, no edge cases) that are noted but don't halt the pipeline
- **Complexity signal** → `low / medium / high` based on AC count, NFR breadth, and integration surface

**⏸ Checkpoint A** — pipeline halts on blocking gaps. Warnings are shown inline and the pipeline continues automatically.

---

### Phase 2 — Context Discovery (PARALLEL, 3–4 agents)

Four sub-agents run concurrently in a single dispatch. Results are merged into `MergedContext` before Phase 3.

#### `qa-visual` (sonnet) — UI Context Extraction
Extracts interactive elements, component-to-testid mappings, page flows, UI states, and responsive breakpoints from:
- **Figma** (via Figma MCP server or REST API) — exact component names, annotations, flows. High confidence.
- **Screenshots** (via vision) — inferred from visible text and layout. Medium confidence, marked as `"inferred"`.
- **Both** — Figma for structure, screenshot for visual verification. Highest confidence.

Returns `null` if no visual input is provided — pipeline handles this gracefully.

#### `qa-explore` (sonnet) — Codebase Scanner
Grounds generated tests in real code by scanning the project:
- **Selectors** — finds `data-testid`, ARIA, and CSS selectors in feature-scoped components; scores each as `EXCELLENT / GOOD / ACCEPTABLE / POOR`
- **API routes** — extracts endpoint definitions using framework-specific patterns (FastAPI decorators, Express router, NestJS controllers)
- **Factories** — finds `*.factory.ts` files and `create*` functions for test data seeding
- **Fixtures** — finds existing Playwright fixture files and `mergeTests` compositions
- **Auth patterns** — locates login flows, token storage, `storageState`, API bypass shortcuts
- **Test conventions** — naming patterns, folder layout, existing tag usage (`@smoke`, `@critical`)

Sets `selectorWarning` when zero `data-testid` attributes are found.

#### `qa-risk` (opus) — Risk Scoring
Evaluates the requirement using a **Probability × Impact** matrix (1–9 scale):

| | Impact 1 (Minor) | Impact 2 (Degraded) | Impact 3 (Critical) |
|---|---|---|---|
| **Probability 1** (Unlikely) | P3 — Document | P3 — Document | P2 — Monitor |
| **Probability 2** (Possible) | P3 — Document | P2 — Monitor | P1 — Mitigate |
| **Probability 3** (Likely) | P2 — Monitor | P1 — Mitigate | **P0 — BLOCK** |

Security and compliance factors automatically raise Impact to 3. Outputs `RiskContext` with score, priority, categories (TECH / SEC / PERF / DATA / BUS / OPS), justification, and `acGaps` for acceptance criteria that lack testable conditions.

#### `qa-live-explore` (sonnet) — Runtime Inspector — CONDITIONAL
Only runs when `AdapterContext.playwrightMcpAvailable` is `true` and a dev server is running. Uses Playwright MCP to inspect the live application:
- Navigates to the feature page; detects and bypasses auth gates via `sessionStorage`/`localStorage` trial flags
- Captures the accessibility tree (`browser_snapshot`) to find actual selector uniqueness — catches ambiguous elements that static analysis misses
- Discovers IndexedDB schemas via `browser_evaluate` — reveals the actual data shapes your tests need to seed
- Records network requests to validate API route patterns
- Captures screenshots of key states

Live-verified selectors take precedence over statically-inferred ones in `MergedContext`.

---

### Phase 3 — Test Scenario Design (`qa-design`, sonnet)

Produces a concrete `TestScenarios[]` from `MergedContext`. Design rules:

- Every AC must have ≥ 1 scenario (full traceability)
- Scenario depth is risk-driven: P0 gets full coverage across all paths; P3 gets smoke only
- Real selectors from `qa-explore` are used — no invented selectors
- Real API routes are referenced for network-first intercept patterns
- Each scenario specifies: preconditions, steps with selectors, network intercepts, expected assertions, priority tags, test ID (`{EPIC}.{STORY}-E2E-{SEQ}`)

**⏸ Checkpoint C (mandatory)** — pipeline always stops here. You review the scenario table, then `[Y]` approve, `[A]` add a scenario, `[R]` revise with feedback, or `[?N]` drill into scenario N's full detail.

---

### Phase 4 — Test Generation (`qa-generate`, sonnet)

Transforms approved scenarios into production-quality Playwright `.spec.ts` files, following 12 non-negotiable rules:

1. **Network-first** — `waitForResponse()` registered _before_ `goto()` or `click()`, never after
2. **Selector hierarchy** — `data-testid` → ARIA role → accessible text → CSS class (CSS class = hard block)
3. **Deterministic waits** — `waitForResponse()`, `expect().toBeVisible()`, spinner disappearance; zero `waitForTimeout()`
4. **Factory-based setup** — data seeded via API factory functions; UI is assertions-only
5. **Fixture composition** — `mergeTests()` pattern, no inheritance chains
6. **Auto-cleanup** — all created resources tracked and deleted in fixture teardown
7. **Tagging** — every test gets `@p0`–`@p3` + `@smoke`/`@regression`
8. **Test IDs** — `{EPIC}.{STORY}-E2E-{SEQ}` in test title
9. **No hardcoded data** — faker for all dynamic values
10. **Co-located helpers** — inline for 1 use, utility for 2–3, fixture for 3+
11. **Playwright prereq check** — stops if `@playwright/test` not installed (pipeline error, not a generation error)
12. **No seeding fallback** — if no API seeding endpoints found, generates UI-based setup and flags as tech-debt

---

### Phase 4.5 — Execute & Self-Heal (`qa-execute`, sonnet)

Runs the generated tests immediately after generation — before review. Classifies each failure by root cause and applies targeted healing (max 3 iterations):

| Failure category | Detection pattern | Heal strategy |
|---|---|---|
| `selector_miss` | "locator resolved to 0 elements", "strict mode violation" | With MCP: `browser_snapshot` to find the correct unique selector. Without MCP: parse Playwright's error hint for the suggested scoped selector. |
| `data_shape` | `Cannot read property`, `undefined is not an object` | With MCP: `browser_evaluate` to inspect IDB/app state and discover actual field names. Without MCP: re-read TypeScript types from source. |
| `auth_setup` | Page URL contains "login", "auth"; expected element not found | With MCP: inspect `sessionStorage`/`localStorage` for bypass flags. Without MCP: grep source for auth bypass patterns. |
| `timing` | "Timeout waiting for", "element not visible" | Add explicit `waitForResponse()` before the assertion or increase assertion timeout. |
| `logic` | Assertion value mismatch where both expected and actual are defined | **Never auto-healed.** Flagged as a potential AC misunderstanding or application bug. |
| `unknown` | Matches no pattern | Logged verbatim. Escalated to user. |

Healed tests are noted as `"fragile"` in the review phase and contribute to a CONCERNS gate decision.

---

### Phase 5 — Quality Review + Auto-Heal (`qa-review-heal`, opus)

Reviews every generated test file across **6 dimensions**, then edits files in-place to fix detected anti-patterns:

| Dimension | What it checks | What it heals |
|---|---|---|
| **Resilience** | Selector stability, timeout patterns, network intercept order | CSS → `data-testid`; adds missing timeouts; moves intercepts before navigation |
| **Determinism** | Race conditions, hard waits, intercept placement | `waitForTimeout` → `waitForResponse`; reorders intercept-then-navigate |
| **Isolation** | Stateless tests, self-cleaning, no shared mutable state | Adds fixture teardown; removes global state mutations |
| **Coverage** | Every AC mapped, depth matches risk priority | Flags unmapped ACs; suggests missing scenarios |
| **Maintainability** | No duplication, clear intent, extract at 3+ rule | Extracts repeated patterns into fixtures |
| **Performance** | No unnecessary waits, parallel-safe, API over UI setup | Removes redundant waits; converts UI setup to API setup |

A file passes the quality gate when: overall score ≥ 80 **and** zero critical-severity issues remain. If the gate fails after one retry, the pipeline writes `_qa-output/review-report.md` and halts — no partial commit.

---

### Phase 6 — Traceability & Gate Decision (`qa-trace`, opus)

Builds a bidirectional traceability matrix — every AC mapped to the tests that cover it, every test mapped back to its AC (orphan tests flagged). Then computes the gate decision:

| Decision | Conditions (first match wins) |
|---|---|
| **FAIL** | Any P0 AC uncovered · any open risk score = 9 · review score < 60 · any `logic` or `unknown` unresolved test failure |
| **CONCERNS** | Any risk score 6–8 · P1 AC gaps · review score 60–79 · any healed tests (fragility signal) |
| **PASS** | All P0/P1 ACs covered · no open critical/high risks · review score ≥ 80 · all tests pass |

Waivers are supported: each requires an approver name, reason, and expiry date. A waived FAIL becomes CONCERNS — never directly PASS.

**⏸ Checkpoint D (mandatory)** — pipeline always halts here with the gate banner. `[APPROVE]` continues to Phase 7 and commits. `[REVISE]` re-runs from Phase 3 with your feedback. `[ABORT]` stops with files uncommitted.

---

### Phase 7 — CI & Script Generation (PARALLEL)

Two agents run concurrently after gate approval:

**`qa-ci` (sonnet)** generates a complete GitHub Actions workflow with:
- **Burn-in testing** — new/changed specs run 10× before merge to catch flakiness early
- **Sharding** — full regression split across N parallel runners (default 4)
- **Artifacts** — screenshots, videos, traces, HAR files uploaded on failure
- **Caching** — `node_modules` + Playwright browser binaries cached by lockfile hash
- **Environment matrix** — parameterised `BASE_URL` for local, staging, production
- **Security** — untrusted ref names passed via `env:` variables, never interpolated into `run:` steps

**`qa-selective` (sonnet)** generates `package.json` scripts and diff-based selection rules:

| Stage | Tests | Time budget |
|---|---|---|
| Pre-commit | `@smoke` | < 2 min |
| PR | changed files + `@p0` + `@p1` | < 10 min |
| Merge to main | full `@regression` | < 30 min |
| Staging deploy | `@smoke` E2E | < 15 min |
| Production | `@p0 @smoke` | < 5 min |

---

### Data contracts

Every phase produces a typed JSON output that the next phase consumes. Schemas live in `schemas/` and are referenced by both the orchestrator and each sub-skill:

| Schema | Produced by | Consumed by |
|---|---|---|
| `AdapterContext` | qa-adapter | all phases |
| `InstrumentContext` | qa-instrument | orchestrator merge |
| `StructuredRequirement` | qa-intake | qa-visual, qa-explore, qa-risk, qa-design, qa-review-heal, qa-trace |
| `VisualContext` | qa-visual | qa-design |
| `CodebaseContext` | qa-explore | qa-design, qa-generate |
| `RiskContext` | qa-risk | qa-design, qa-trace |
| `LiveContext` | qa-live-explore | qa-design, qa-generate, qa-execute |
| `TestScenarios[]` | qa-design | qa-generate |
| `ExecutionResult` | qa-execute | qa-trace |
| `ReviewReport` | qa-review-heal | qa-trace |
| `GateDecision` | qa-trace | orchestrator |

### Sub-agent model routing

| Sub-agent | Model | Rationale |
|---|---|---|
| qa-adapter | sonnet | Mechanical detection — grep, glob, classify |
| qa-instrument | sonnet | Source file edits — mechanical, rule-bound |
| qa-intake | sonnet | Structured extraction — well-defined transformation |
| qa-visual | sonnet | UI analysis — pattern matching from Figma/screenshots |
| qa-explore | sonnet | Codebase scanning — grep/glob intensive |
| **qa-risk** | **opus** | Evaluative judgement — Probability × Impact requires nuanced reasoning |
| qa-live-explore | sonnet | Browser navigation — tool-call intensive, not reasoning-heavy |
| qa-design | sonnet | Scenario design — structured, rule-driven |
| qa-generate | sonnet | Code generation — high volume, well-specified rules |
| qa-execute | sonnet | Test execution + pattern-matched healing |
| **qa-review-heal** | **opus** | Quality judgement — 6-dimension review requires deep analysis |
| **qa-trace** | **opus** | Coverage evaluation + gate decision — high-stakes, must not miss edge cases |
| qa-ci | sonnet | YAML generation — template-driven |
| qa-selective | sonnet | Script generation — rule-driven |

---

## Output artifacts

| Artifact | Path |
|---|---|
| Playwright tests | `tests/e2e/{feature}/` |
| Factories & fixtures | `tests/e2e/{feature}/` or `tests/e2e/support/` |
| Risk report | `_qa-output/risk-report.md` |
| Traceability matrix | `_qa-output/trace-matrix.md` |
| Gate decision | `_qa-output/gate-decision.md` |
| Review report | `_qa-output/review-report.md` |
| CI workflow | `.github/workflows/e2e-tests.yml` |
| Test scripts | `package.json` |

---

## Supported frameworks

The `qa-adapter` phase auto-detects your stack. Confirmed working with:

- **Runners:** Playwright (primary), Vitest, Jest, Cypress
- **Frameworks:** Next.js, Nuxt, SvelteKit, Remix, plain Vite, CRA
- **Languages:** TypeScript, JavaScript
- **CI:** GitHub Actions (generated), adaptable to others

---

## License

MIT
