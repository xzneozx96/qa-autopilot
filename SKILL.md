---
name: qa-autopilot
description: >
  Use when the user wants to generate, write, or create end-to-end Playwright tests — whether
  from a user story, acceptance criteria, Jira/Linear ticket, PR description, or just a plain
  feature description. Orchestrates the full QA pipeline from requirements intake through test
  generation, execution, review, traceability, and CI setup. Trigger even when the user just
  says "write tests for X", "add e2e coverage", "generate playwright tests", or "I need test
  coverage for this feature" — they don't need to provide formatted ACs for this to apply.
---

# qa-autopilot — Pipeline Orchestrator

Coordinate the full QA pipeline. Spawn sub-agents for parallel phases. Present human-readable summaries at checkpoints. Handle errors and partial failures.

> **SKILL_BASE** — When this skill loads, Claude Code shows "Base directory for this skill: <path>". That path is `SKILL_BASE`. All sub-skill paths in this file are relative to it (e.g. `{SKILL_BASE}/qa-visual/SKILL.md`).

## Input Format

The user invokes with a user story, acceptance criteria, and optional visual references:

```
/qa-autopilot "As a [actor], I can [action] so that [value].

AC:
- Given [precondition], when [action], then [expected result]
- ...

Figma: https://figma.com/design/...          (optional)
Screenshots: ./path/to/screen1.png, ...      (optional)
Screenshots dir: ./designs/feature/           (optional)"
```

## Sub-Agent Model Assignment

| Sub-agent | Role | Model | Rationale |
|---|---|---|---|
| qa-adapter | Detection/discovery | sonnet | Implementation: scans and classifies |
| qa-instrument | Adding data-testid | sonnet | Implementation: edits source files |
| qa-intake | Parsing requirements | sonnet | Implementation: structured extraction |
| qa-visual | UI context extraction | sonnet | Implementation: Figma/screenshot analysis |
| qa-explore | Codebase scanning | sonnet | Implementation: grep/glob discovery |
| qa-risk | Risk scoring | opus | **Reviewer**: evaluates probability × impact |
| qa-live-explore | Runtime inspection | sonnet | Implementation: browser-based discovery |
| qa-design | Scenario design | sonnet | Implementation: creates test scenarios |
| qa-generate | Test code generation | sonnet | Implementation: writes Playwright files |
| qa-execute | Test runner/healer | sonnet | Implementation: runs and heals tests |
| qa-review-heal | Quality review | opus | **Reviewer**: reviews across 6 dimensions |
| qa-trace | Traceability & gate | opus | **Reviewer**: evaluates coverage and risk |
| qa-ci | CI config generation | sonnet | Implementation: writes workflow YAML |
| qa-selective | Script generation | sonnet | Implementation: writes package.json scripts |

## Pipeline

### Phase 0: Adaptation
1. **qa-adapter** — Sequential. All downstream steps depend on AdapterContext.

   Dispatch via Agent tool:
   ```
   model: sonnet
   description: "Detect tech stack for QA"
   prompt: "Read {SKILL_BASE}/qa-adapter/SKILL.md and execute it.
   Project root: {cwd}.
   Return AdapterContext JSON conforming to the schema in the skill."
   ```
   Wait for result. Extract `AdapterContext` from response.

### Phase 0.5: Codebase Instrumentation (CONDITIONAL)

**Trigger:** Only when `AdapterContext.selectorStrategy.dataTestIdCoverage` is `"none"` or `"low"`.
Skip entirely for `"medium"` or `"high"` — proceed directly to Phase 1.

2. **qa-instrument** — Scope to feature-relevant components, add `data-testid` attributes.

   Dispatch via Agent tool:
   ```
   model: sonnet
   description: "Instrument components with data-testid"
   prompt: "Read {SKILL_BASE}/qa-instrument/SKILL.md and execute it.
   Input: AdapterContext={AdapterContext JSON}, rawUserInput={original user message}.
   Return InstrumentContext JSON conforming to the schema in the skill."
   ```
   Wait for result. If `InstrumentContext.skipped` is `true`, pipeline continues with ARIA fallbacks.
   If instrumentation succeeded, replace `AdapterContext.selectorStrategy` with
   `InstrumentContext.updatedSelectorStrategy` before passing context downstream.

### Phase 1: Requirements Analysis
3. **qa-intake** — Parse user story + AC into StructuredRequirement. Sequential.

   Dispatch via Agent tool:
   ```
   model: sonnet
   description: "Parse requirements into structured format"
   prompt: "Read {SKILL_BASE}/qa-intake/SKILL.md and execute it.
   Input: rawUserInput={original user message}, figmaUrl={url or null}.
   Return StructuredRequirement JSON conforming to the schema in the skill."
   ```
   Wait for result. Extract `StructuredRequirement` from response.

4. **⏸ CHECKPOINT A (conditional)** — Inspect qa-intake output:
   - If **blocking gaps** found: STOP. Present gaps to user, ask for clarification. Resume from step 3.
   - If **warnings only**: Display inline summary and continue without pausing.

   Inline summary format:
   ```
   📋 Requirements parsed:
     Story: [actor] can [action] for [value]
     ACs: [count] parsed ([functional count] functional, [nfr count] NFR)
     Gaps: [count] warnings (listed below)
       ⚠ [gap description] → suggestion: [suggestion]
     Complexity: [low/medium/high]
   ```

### Phase 2: Context Discovery (PARALLEL)
5. **Dispatch 3–4 parallel sub-agents** using the Agent tool in a single message:

   **Agent 1 (qa-visual):**
   ```
   model: sonnet
   description: "Extract UI context from Figma/screenshots"
   prompt: "Read {SKILL_BASE}/qa-visual/SKILL.md and execute it.
   Input: figmaUrl={url}, screenshots={paths}.
   Return VisualContext JSON conforming to the schema in the skill."
   ```

   **Agent 2 (qa-explore):**
   ```
   model: sonnet
   description: "Scan codebase for selectors and routes"
   prompt: "Read {SKILL_BASE}/qa-explore/SKILL.md and execute it.
   Input: StructuredRequirement={StructuredRequirement JSON}, AdapterContext={AdapterContext JSON}.
   Return CodebaseContext JSON conforming to the schema in the skill."
   ```

   **Agent 3 (qa-risk):**
   ```
   model: opus
   description: "Score requirement risk"
   prompt: "Read {SKILL_BASE}/qa-risk/SKILL.md and execute it.
   Input: StructuredRequirement={StructuredRequirement JSON}.
   Return RiskContext JSON conforming to the schema in the skill."
   ```

   **Agent 4 (qa-live-explore) — CONDITIONAL:**
   ```
   model: sonnet
   description: "Inspect running app via Playwright MCP"
   prompt: "Read {SKILL_BASE}/qa-live-explore/SKILL.md and execute it.
   Input: AdapterContext={AdapterContext JSON}, StructuredRequirement={StructuredRequirement JSON}.
   Return LiveContext JSON conforming to the schema in the skill."
   ```
   Only dispatch if `AdapterContext.playwrightMcpAvailable` is `true` AND
   `AdapterContext.devServerUrl` is not `null`. Otherwise skip with warning.

6. **Merge parallel outputs** into MergedContext. If LiveContext is available, apply merge protocol: live-verified selectors override static selectors, IDB schemas and auth bypass are added as separate context.

7. **CHECKPOINT B (informational)** — Display inline summary, do NOT pause:
   ```
   🔍 Context discovery complete:
     Risk: P[0-3] (score [N]) — [justification one-liner]
     Selectors: [data-testid coverage] ([count] found)
     API routes: [count] endpoints discovered
     Factories: [count] existing | Fixtures: [count] existing
     Visual: [figma/screenshot/both/none] — [count] elements extracted
     ⚠ [any critical findings, e.g. "no API seeding"]
   ```
   If `CodebaseContext.selectorWarning` is not null AND `InstrumentContext.skipped` is `true`,
   append it as a prominent warning:
   ```
   ⚠ Selector quality: data-testid instrumentation was skipped.
     Generated selectors will fall back to ARIA roles and CSS classes (fragile).
     CSS-class selectors will be flagged as tech-debt in test comments.
   ```
   Do NOT show this warning when instrumentation succeeded (`InstrumentContext.skipped` is `false`).

### Phase 3: Test Design
8. **qa-design** — Generate test scenarios. Sequential.

   Dispatch via Agent tool:
   ```
   model: sonnet
   description: "Design test scenarios from merged context"
   prompt: "Read {SKILL_BASE}/qa-design/SKILL.md and execute it.
   Input: StructuredRequirement={StructuredRequirement JSON}, VisualContext={VisualContext JSON or null},
   CodebaseContext={CodebaseContext JSON}, RiskContext={RiskContext JSON}.
   Return TestScenarios[] JSON conforming to the schema in the skill."
   ```
   Wait for result. Extract `TestScenarios[]` from response.

9. **⏸ CHECKPOINT C (hard stop — mandatory approval)** — Present scenario table:
   ```
   📝 Test Scenarios — [Feature] ([Priority])

   [count] scenarios designed • [smoke count] smoke • [regression count] regression

     #  Type     Tag          Title
     1  [type]   @[pri] @[stage]  [title]
     2  [type]   @[pri] @[stage]  [title]
     ...

   ⚠ Notes:
     - [any design decisions or gaps]

   [Y] Approve  [A] Add scenario  [R] Revise  [?N] Show detail for scenario N
   ```

   Wait for user response:
   - **Y** → proceed to generation
   - **A "description"** → add scenario, re-display table
   - **R "feedback"** → re-run qa-design with feedback as constraint
   - **?N** → show full detail (preconditions, steps, assertions) for scenario N

### Phase 4: Test Generation
10. **qa-generate** — Produce Playwright .spec.ts files, factories, fixtures. Sequential.

    Dispatch via Agent tool:
    ```
    model: sonnet
    description: "Generate Playwright test files"
    prompt: "Read {SKILL_BASE}/qa-generate/SKILL.md and execute it.
    Input: TestScenarios={TestScenarios JSON}, CodebaseContext={CodebaseContext JSON},
    AdapterContext={AdapterContext JSON}, LiveContext={LiveContext JSON or null}.
    Generate test files following all 12 rules in the skill. Write files to disk."
    ```
    Wait for result. Files are written to `tests/e2e/{feature}/`.

### Phase 4.5: Execute-and-Heal
11. **qa-execute** — Run tests, classify failures, self-heal. Sequential.

    Dispatch via Agent tool:
    ```
    model: sonnet
    description: "Run tests and self-heal failures"
    prompt: "Read {SKILL_BASE}/qa-execute/SKILL.md and execute it.
    Input: AdapterContext={AdapterContext JSON}, StructuredRequirement={StructuredRequirement JSON},
    LiveContext={LiveContext JSON or null}. Feature dir: tests/e2e/{feature}/.
    Run all generated tests. Classify and heal failures up to 3 iterations.
    Return ExecutionResult JSON conforming to the schema in the skill."
    ```
    Wait for result. Extract `ExecutionResult` from response.

12. **Check execution result**:
    - If all tests pass: continue to Phase 5 with inline report.
    - If unhealed failures exist: continue to Phase 5 — failures will be factored into the gate decision as open risks.

### Phase 5: Quality Control
13. **qa-review-heal** — Review generated files across 6 dimensions. Auto-heal anti-patterns. Sequential.

    Dispatch via Agent tool:
    ```
    model: opus
    description: "Review and heal test quality"
    prompt: "Read {SKILL_BASE}/qa-review-heal/SKILL.md and execute it.
    Input: StructuredRequirement={StructuredRequirement JSON}.
    Review all generated test files in tests/e2e/{feature}/ across the 6 dimensions.
    Auto-heal detected anti-patterns by editing files in-place.
    Return ReviewReport JSON conforming to the schema in the skill."
    ```
    Wait for result. Extract `ReviewReport` from response.

14. **Check review score**:
    - If score < 80 or any critical issues remain: pass unhealed issues as constraints to qa-generate. Re-run steps 10–14 (max 1 retry).
    - If still < 80 or critical issues remain after retry: FAIL. Write `_qa-output/review-report.md` and stop.

### Phase 6: Traceability & Gate
15. **qa-trace** — Build traceability matrix and compute gate decision. Sequential.

    Dispatch via Agent tool:
    ```
    model: opus
    description: "Build traceability matrix and gate decision"
    prompt: "Read {SKILL_BASE}/qa-trace/SKILL.md and execute it.
    Input: StructuredRequirement={StructuredRequirement JSON}, RiskContext={RiskContext JSON},
    ReviewReport={ReviewReport JSON}, ExecutionResult={ExecutionResult JSON}.
    Build AC-to-Test traceability matrix and compute gate decision.
    Write _qa-output/trace-matrix.md and _qa-output/gate-decision.md.
    Return GateDecision JSON conforming to the schema in the skill."
    ```
    Wait for result. Extract `GateDecision` from response.

16. **⏸ CHECKPOINT D (hard stop)** — Present gate decision:
    ```
    ╔══════════════════════════════════════════════════╗
    ║  GATE: [PASS/CONCERNS/FAIL]                      ║
    ║                                                  ║
    ║  Coverage: [X]/[Y] ACs  |  Score: [N]/100       ║
    ║  Execution: [X]/[Y] passed | [Z] healed         ║
    ║  Risk: P[N] ([action])  |  Open issues: [count]  ║
    ║                                                  ║
    ║  [if CONCERNS/FAIL: reason list]                 ║
    ║                                                  ║
    ║  [A] APPROVE  [R] REVISE  [X] ABORT              ║
    ╚══════════════════════════════════════════════════╝
    ```

    - **APPROVE** → proceed to Phase 7
    - **REVISE** → user provides feedback → re-run from step 8 (qa-design)
    - **ABORT** → stop, keep files uncommitted

### Phase 7: CI & Execution (on APPROVE only)
17. **Dispatch qa-ci + qa-selective in parallel** using the Agent tool in a single message:

    **Agent 1 (qa-ci):**
    ```
    model: sonnet
    description: "Generate CI workflow"
    prompt: "Read {SKILL_BASE}/qa-ci/SKILL.md and execute it.
    Input: AdapterContext={AdapterContext JSON}.
    Generate .github/workflows/qa.yml with burn-in, sharding, and artifact upload.
    Write the file to disk."
    ```

    **Agent 2 (qa-selective):**
    ```
    model: sonnet
    description: "Generate selective test scripts"
    prompt: "Read {SKILL_BASE}/qa-selective/SKILL.md and execute it.
    Input: AdapterContext={AdapterContext JSON}, generated test tags from prior phases.
    Generate package.json test scripts and diff-based selection config.
    Write changes to package.json."
    ```

18. **Commit** — Pre-check: `git rev-parse --git-dir`.
    - If `tests/e2e/{feature}/` exists: present diff, ask overwrite/merge/abort.
    - If no git repo: skip commit, inform user files are ready.
    - Otherwise: commit all generated files.

## Re-run Scoping

| Checkpoint | On revise, re-run from |
|---|---|
| A (intake) | Phase 1 only (step 3), then continue |
| C (design) | Phase 3 onward (steps 8–18) |
| D (gate) | Phase 3 onward (steps 8–18) |

Phase 0 (adapter), Phase 0.5 (instrument), and Phase 2 (explore/risk/visual) are **never re-run** — codebase context and instrumentation don't change on requirements revise.

## Error Handling

| Failure | Action |
|---------|--------|
| No Figma URL and no screenshots | Skip qa-visual, proceed without VisualContext. Warn user. |
| Figma URL invalid | Fall back to screenshots if provided. Otherwise skip qa-visual. |
| No existing test setup | qa-generate scaffolds Playwright (`pnpm create playwright`). |
| Review score < 60 after retry | FAIL. Write review report. Do not commit. |
| Sub-agent fails | Report error. Continue with available context. |
| `tests/e2e/{feature}/` exists | Present diff. Ask: overwrite, merge, or abort. |
| Playwright not installed after adapter | Pipeline error. Report and stop. |
| Dev server cannot start | Skip live explore. qa-execute starts server if needed. |
| Instrumentation commit fails | Report failure. Files are written. Continue pipeline with `commitSha: null`. |

## Output Artifacts

| Artifact | Location |
|----------|----------|
| Instrumentation commit | git history (separate chore commit, conditional) |
| Test files | `tests/e2e/{feature}/` |
| Factories | `tests/e2e/{feature}/` or `tests/e2e/support/factories/` |
| Fixtures | `tests/e2e/{feature}/` or `tests/e2e/support/fixtures/` |
| Risk report | `_qa-output/risk-report.md` |
| Traceability matrix | `_qa-output/trace-matrix.md` |
| Gate decision | `_qa-output/gate-decision.md` |
| Review report | `_qa-output/review-report.md` |
| CI workflow | `.github/workflows/e2e-tests.yml` |
| Test scripts | `package.json` |
| Playwright config | `playwright.config.ts` |
