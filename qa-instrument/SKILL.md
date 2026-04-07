---
name: qa-instrument
description: >
  This skill should be used when dataTestIdCoverage is "none" or "low" after qa-adapter runs. Scopes to
  feature-relevant components, adds data-testid attributes to interactive elements,
  and commits the changes before test generation begins. Phase 0.5 — runs after
  qa-adapter, before qa-intake.
---

# qa-instrument — Codebase Instrumentation

Add `data-testid` attributes to feature-relevant components so that `qa-generate` can
produce resilient `getByTestId()` selectors instead of falling back to CSS classes.

## Inputs

| Input | Source |
|-------|--------|
| `AdapterContext` | qa-adapter (Phase 0) |
| `rawUserInput` | Original user message (user story + ACs as plain text) |

## Trigger Condition

Only runs when `AdapterContext.selectorStrategy.dataTestIdCoverage` is `"none"` or `"low"`.
Skip entirely for `"medium"` or `"high"` — existing coverage is sufficient.

## Naming Convention

All `data-testid` values MUST follow this convention:

```
{feature}-{element-type}
```

Rules:
- **kebab-case only** — no underscores, no camelCase, no dots
- **feature prefix** — derived from the feature name in the user story (e.g. `admin-login`, `checkout`, `user-profile`)
- **element-type suffix** — describes the element's role, not its visual appearance:
  - Inputs: `email-input`, `password-input`, `search-input`, `{field-name}-input`
  - Buttons: `submit-button`, `cancel-button`, `{action}-button`
  - Links: `{destination}-link`, `{action}-link`
  - Forms: `{name}-form`
  - Error messages: `{field}-error`, `form-error`
  - Success/confirmation UI: `success-message`, `confirmation-banner`
  - Loading states: `{name}-spinner`, `loading-indicator`
  - Containers (only when needed for scoping): `{name}-section`, `{name}-panel`
  - List items: `{name}-item`, `{name}-row`, `{name}-card`
  - Modals/dialogs: `{name}-dialog`, `{name}-modal`
  - Toasts/notifications: `{name}-toast`, `error-toast`, `success-toast`
- **No visual descriptors** — never use color, size, or position: not `red-text`, `big-button`, `top-nav`
- **No implementation details** — never expose internal class names or component library internals

Examples:
```tsx
// ✅ Good
data-testid="admin-login-email-input"
data-testid="admin-login-password-input"
data-testid="admin-login-submit-button"
data-testid="admin-login-form-error"
data-testid="admin-login-password-toggle"

// ❌ Bad
data-testid="input1"            // not descriptive
data-testid="btn_primary"       // underscore, visual descriptor
data-testid="LoginEmailField"   // camelCase
data-testid="text-red-600"      // CSS class leaked in
```

## Procedure

### 1. Derive Feature Scope

Extract a feature keyword from `rawUserInput` (e.g. "admin login" → `admin-login`,
"checkout flow" → `checkout`). This becomes the feature prefix for all `data-testid` values.

Use the keyword to scope component discovery — do not instrument the entire codebase,
only the components that the feature touches.

### 2. Discover Feature Components

Using `AdapterContext.framework` to know the file conventions:

- **Next.js / React**: Glob for `app/**/page.tsx`, `app/**/layout.tsx`, `components/**/*.tsx`
  filtered to paths containing the feature keyword or related terms.
- **Vue**: Glob for `src/**/*.vue` filtered to feature-related paths.
- **Angular**: Glob for `src/**/*.component.html` filtered to feature-related paths.

Also grep source files for the feature keyword to catch non-obviously-named components
(e.g. a login form inside `components/auth/AuthCard.tsx`).

Limit to a maximum of **15 files** — if more match, prioritize by proximity to the
feature's primary route/page, then by file size (smaller = more focused).

### 3. Identify Instrumentation Targets

For each discovered component file, read it and identify elements that need `data-testid`:

**Always instrument:**
- `<input>`, `<textarea>`, `<select>` — all form fields
- `<button>` — all buttons (including icon-only buttons)
- `<a>` — all navigation links relevant to the feature flow
- Error/validation message elements (even if they are `<p>`, `<span>`, `<div>`)
- Loading indicators / spinners
- Success/confirmation messages

**Instrument if they are key interaction containers:**
- Form wrappers (`<form>`)
- Modal/dialog wrappers
- Section containers needed to scope ambiguous child selectors

**Do NOT instrument:**
- Pure layout divs/sections with no interactive descendants
- Third-party component library internals
- Elements already having a `data-testid`
- SVG elements or icon components

### 4. Present Instrumentation Plan

Before modifying any file, display the plan as a table and ask for approval:

```
🔬 INSTRUMENTATION PLAN — [feature-prefix]

[count] file(s) to modify:

  File                                    Elements to instrument
  ──────────────────────────────────────  ─────────────────────────────────────────────
  app/admin/login/page.tsx                email-input, password-input, submit-button,
                                          password-toggle, form-error
  components/auth/AuthCard.tsx            admin-login-card (container)

  [count] data-testid attributes will be added across [count] files.

  Naming prefix: [feature-prefix]

[I] Instrument now   [S] Skip instrumentation (use ARIA fallbacks)
```

Wait for user response:
- **I** → proceed with instrumentation
- **S** → skip this phase entirely. Set `InstrumentContext.skipped: true`. Continue pipeline.

### 5. Apply Instrumentation

For each file in the approved plan:

1. Read the current file content.
2. Add `data-testid` attributes to each identified element.
3. Preserve ALL existing code — indentation, formatting, comments, logic. Only add the
   `data-testid` prop/attribute. Do not refactor, rename, or touch anything else.
4. Write the updated file.

**Framework-specific syntax:**

React/Next.js (TSX/JSX):
```tsx
// Before
<button type="submit" className="btn-primary">Sign In</button>

// After
<button type="submit" className="btn-primary" data-testid="admin-login-submit-button">Sign In</button>
```

Vue (SFC template):
```vue
<!-- Before -->
<button type="submit" class="btn-primary">Sign In</button>

<!-- After -->
<button type="submit" class="btn-primary" data-testid="admin-login-submit-button">Sign In</button>
```

Angular (HTML template):
```html
<!-- Before -->
<button type="submit" class="btn-primary">Sign In</button>

<!-- After -->
<button type="submit" class="btn-primary" data-testid="admin-login-submit-button">Sign In</button>
```

### 6. Commit

After all files are written:

```bash
git add {list of modified files}
git commit -m "chore(qa): add data-testid attributes for [feature-prefix] e2e tests

Instrumented [count] component(s) with [count] data-testid attributes
to enable resilient getByTestId() selectors in Playwright tests.

Files modified:
  - [file1]
  - [file2]
  ...

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

Record the commit SHA in the output.

### 7. Update AdapterContext

Produce an updated `selectorStrategy` block for `AdapterContext`:

```json
{
  "dataTestIdCoverage": "high",
  "ariaRoleCoverage": "[unchanged from adapter]",
  "recommendation": "Use getByTestId() — data-testid attributes added by qa-instrument"
}
```

Downstream skills (`qa-explore`, `qa-generate`) consume this updated context.

## Output

Return an `InstrumentContext` object conforming to `instrument-output.schema.json`.

## Error Handling

| Failure | Action |
|---------|--------|
| No feature-relevant components found | Set `skipped: true`, reason: "no components found for feature scope". Continue pipeline — qa-generate will use ARIA fallbacks. |
| File write fails | Report the failure, skip that file, continue with remaining files. List failed files in output. |
| Git commit fails | Report failure. Files are already written — pipeline continues. Set `commitSha: null`. |
| User selects Skip | Set `skipped: true`. Continue pipeline — qa-generate will use ARIA fallbacks and flag selectors as tech-debt. |

## Knowledge References

- [`../knowledge/selector-resilience.md`](../knowledge/selector-resilience.md) — selector hierarchy and naming rationale
