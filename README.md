# QA Autopilot

> **From user story to production-ready Playwright tests — fully automated.**

QA Autopilot is a Claude Code skill that orchestrates a complete, multi-phase QA pipeline. Drop in a user story or ticket description and get back battle-hardened Playwright tests, a risk report, a traceability matrix, a gate decision, and CI configuration — all without leaving your editor.

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

```
qa-agent/
├── SKILL.md                    ← Orchestrator (you are here)
│
├── qa-adapter/                 ← Phase 0:   Detect framework, runner, selectors
├── qa-instrument/              ← Phase 0.5: Add data-testid to components (conditional)
├── qa-intake/                  ← Phase 1:   Parse requirements → StructuredRequirement
│
├── qa-visual/                  ← Phase 2 ┐
├── qa-explore/                 ← Phase 2 ├── Parallel context discovery
├── qa-risk/                    ← Phase 2 ┤  (risk scored by opus)
├── qa-live-explore/            ← Phase 2 ┘  (live browser — conditional)
│
├── qa-design/                  ← Phase 3:   Design test scenarios (hard stop for approval)
├── qa-generate/                ← Phase 4:   Generate .spec.ts files + factories + fixtures
├── qa-execute/                 ← Phase 4.5: Run tests + self-heal (up to 3 iterations)
├── qa-review-heal/             ← Phase 5:   Quality review across 6 dimensions (opus)
├── qa-trace/                   ← Phase 6:   Traceability matrix + gate decision (opus)
│
├── qa-ci/                      ← Phase 7 ┐  Parallel: CI workflow + test scripts
├── qa-selective/               ← Phase 7 ┘
│
├── knowledge/                  ← Domain knowledge loaded on demand
├── schemas/                    ← JSON schemas for all inter-agent contracts
└── templates/                  ← Output templates (test file, risk report, trace matrix)
```

### Sub-agent model routing

| Sub-agent | Model | Role |
|---|---|---|
| qa-adapter | sonnet | Detect framework |
| qa-instrument | sonnet | Add data-testid attributes |
| qa-intake | sonnet | Parse requirements |
| qa-visual | sonnet | Extract UI context from Figma / screenshots |
| qa-explore | sonnet | Scan codebase for selectors and routes |
| **qa-risk** | **opus** | Score risk (Probability × Impact) |
| qa-live-explore | sonnet | Inspect running app via Playwright MCP |
| qa-design | sonnet | Design test scenarios |
| qa-generate | sonnet | Write Playwright spec files |
| qa-execute | sonnet | Run tests, classify failures, self-heal |
| **qa-review-heal** | **opus** | Review quality across 6 dimensions |
| **qa-trace** | **opus** | Build traceability matrix + gate decision |
| qa-ci | sonnet | Generate GitHub Actions workflow |
| qa-selective | sonnet | Generate package.json test scripts |

### Pipeline flow

```
Phase 0    qa-adapter          → AdapterContext
Phase 0.5  qa-instrument       → InstrumentContext       (conditional: low data-testid coverage)
Phase 1    qa-intake           → StructuredRequirement
           ⏸ Checkpoint A     → Block on gaps, warn on minor issues

Phase 2 ──────────────────────── PARALLEL ─────────────────────────
         qa-visual             → VisualContext           (conditional: Figma/screenshots provided)
         qa-explore            → CodebaseContext
         qa-risk               → RiskContext
         qa-live-explore       → LiveContext             (conditional: dev server running)
─────────────────────────────────────────────────────────────────────

Phase 3    qa-design           → TestScenarios[]
           ⏸ Checkpoint C     → Hard stop: user approves / revises / adds scenarios

Phase 4    qa-generate         → Playwright .spec.ts + factories + fixtures
Phase 4.5  qa-execute          → ExecutionResult (self-heal up to 3 iterations)
Phase 5    qa-review-heal      → ReviewReport    (opus, auto-heal anti-patterns)
Phase 6    qa-trace            → GateDecision    (opus)
           ⏸ Checkpoint D     → Hard stop: PASS / CONCERNS / FAIL + user approval

Phase 7 ──────────────────────── PARALLEL ─────────────────────────
         qa-ci                 → .github/workflows/e2e-tests.yml
         qa-selective          → package.json scripts + diff-based selection
─────────────────────────────────────────────────────────────────────

           Commit all generated files
```

### Gate decision

The `qa-trace` sub-agent (opus) evaluates every run against this rubric:

| Decision | Condition |
|---|---|
| **FAIL** | Any P0 AC uncovered · risk score 9 open · review score < 60 · logic/unknown test failures |
| **CONCERNS** | P1 gaps · risk score 6–8 · review score 60–79 · healed tests present |
| **PASS** | All P0/P1 ACs covered · no critical/high open risks · review score ≥ 80 |

Precedence: FAIL > CONCERNS > PASS.

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

## Requirements

- [Claude Code](https://claude.ai/code) CLI
- Claude API access (Sonnet + Opus)
- Node.js 18+ (for Playwright)
- Git (for commit step)

---

## Installation

Copy the skill folder into your Claude skills directory:

```bash
cp -r qa-agent ~/.claude/skills/qa-agent
```

That's it. Claude Code auto-discovers skills in `~/.claude/skills/` — no restart required.

> **Want the latest version?**
> ```bash
> # Clone fresh
> git clone https://github.com/xzneozx96/qa-autopilot.git
> cp -r qa-autopilot/qa-agent ~/.claude/skills/qa-agent
>
> # Or pull updates later
> cd qa-autopilot && git pull
> cp -r qa-agent ~/.claude/skills/qa-agent
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

The pipeline runs end-to-end, pausing only at the 3 hard checkpoints:
- **Checkpoint A** — requirement gaps (auto-skipped for warnings)
- **Checkpoint C** — scenario approval (always stops for your sign-off)
- **Checkpoint D** — gate decision (always stops before committing)

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
