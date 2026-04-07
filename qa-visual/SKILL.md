---
name: qa-visual
description: "This skill should be used when extracting UI context from Figma URLs or screenshots — interactive elements, page flows, states, and component-to-testid mappings for test generation."
---

# qa-visual — UI Context Extraction

Phase 2 sub-agent. Extracts UI context from **Figma designs**, **screenshots**, or **both** to feed the test generation pipeline.

## When to Use

Invoke when the orchestrator provides a Figma URL, screenshot file paths, or both. If no visual input is provided, return `null` — the orchestrator handles missing UI context gracefully.

## Input Sources

| Source | How | Confidence |
|--------|-----|-----------|
| Figma URL | Figma MCP server or `GET /v1/files/:key/nodes` via WebFetch | High — exact component names, annotations, flows |
| Screenshot paths | Read tool (vision) to analyze image content | Medium — inferred from visible text and layout |
| Both | Figma for structure + screenshot for visual verification | Highest |
| Neither | Return `null` | — |

## Extraction Targets

1. **Interactive elements** — buttons, inputs, forms, links, dropdowns, modals.
2. **Component-to-testid mapping** — derive `data-testid` values from component/label names.
3. **Page flows** — which pages connect, step sequences, start/end pages.
4. **UI states** — loading, error, empty, success, disabled — with trigger conditions.
5. **Responsive breakpoints** — mobile, tablet, desktop frames.

## data-testid Naming Convention

- Convert component or visible label to **lowercase kebab-case**.
- Strip namespace prefixes (e.g. `Atoms/PayButton` → `pay-button`).
- Compound names split on camelCase and `/`: `UserProfileCard` → `user-profile-card`.
- Append element type suffix when ambiguous: `search-input`, `submit-button`.

## ARIA Role Mapping

Map element types to suggested ARIA roles: buttons → `button`, inputs → `textbox`, links → `link`, dropdowns → `combobox`, modals → `dialog`, forms → `form`.

See [selector-resilience.md](../knowledge/selector-resilience.md) for selector strategy.

## Implementation by Source

**Figma URL:**
1. Prefer Figma MCP server if available.
2. Fallback: WebFetch + Figma API (`GET /v1/files/:key/nodes`).
3. Walk node tree, classify elements, extract labels, build output.

**Screenshots:**
1. Use the Read tool to view each image file.
2. Identify interactive elements from visual appearance and labels.
3. Infer element types (button, input, link) from visual cues.
4. Extract visible text → derive suggestedTestId via naming convention.
5. Identify visible states (loading spinners, error banners, empty states).
6. Infer page flows from multiple sequential screenshots.
7. Mark `confidence: "inferred"` on all extracted elements.

**Both:** Run Figma extraction first, then verify/supplement with screenshot analysis.

## Output Schema

Return a `VisualContext` object (see `schemas/visual-output.schema.json`):

- `source` — `"figma"`, `"screenshot"`, or `"both"`
- `pages[]` — name, url, elements (type, label, suggestedTestId, ariaRole, confidence), states
- `flows[]` — name, ordered steps, startPage, endPage
- `breakpoints[]` — responsive breakpoint labels

Return `null` if no visual input provided or all sources fail.
