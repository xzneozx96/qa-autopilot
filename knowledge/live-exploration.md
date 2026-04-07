# Live Exploration Patterns

## Principle

Static code analysis (grep/glob) discovers what selectors and data shapes *exist in source*.
Live browser inspection discovers how they *behave at runtime*. Live findings override
static findings where they conflict.

## DOM Inspection

### Accessible Name Inheritance

A `div[role="button"]` with no explicit `aria-label` inherits its accessible name from
child text content and child `aria-label` attributes. This means:

```html
<div role="button">
  <button aria-label="Remove from Workbook">
    <svg />
  </button>
  <button aria-label="Play pronunciation of 你好">
    <svg />
  </button>
  <p>你好</p>
</div>
```

The outer div's accessible name becomes:
`"Remove from Workbook Play pronunciation of 你好 你好"`

This causes `getByRole('button', { name: /remove from workbook/i })` to match BOTH
the outer div AND the inner button — a strict mode violation.

### Disambiguation Strategies

1. **Scope by parent:** `page.getByRole('tabpanel', { name: /workbook/i }).getByLabel('Remove')`
2. **Use `title` attribute:** `page.getByTitle('Remove from Workbook')` — only matches elements with an explicit `title`
3. **Use `data-testid`:** `page.getByTestId('remove-word-btn')` — always unique if present

### Snapshot-Based Discovery

Use `browser_snapshot` to get the full accessibility tree. Walk the tree to find:
- Elements with `matchCount > 1` for the same accessible name
- Nested interactive elements (button inside button, link inside button)
- Missing labels (elements with no accessible name)

## IDB Schema Inspection

### Why Static Analysis Fails

TypeScript types describe the *application model*, not the *database schema*:
- `keyPath` can be `null` (out-of-line keys) — you can't tell from the type alone
- Field names may differ between TypeScript and the stored record (e.g., migrations)
- Array-valued stores (segments stored as `Segment[]` keyed by lessonId) look like
  single-object stores in TypeScript

### Runtime Inspection Pattern

Open each store, read the `keyPath` and `autoIncrement` from the store metadata,
then read one sample record to get actual field names. This is definitive.

## Auth Bypass Patterns

Common patterns to search for in `sessionStorage` / `localStorage`:
- `*_trial`, `*_bypass`, `*_demo` keys
- JWT tokens or session cookies that can be pre-set
- Feature flags stored client-side

The goal is to find the minimal `page.addInitScript()` that skips the auth gate,
so tests can navigate directly to the feature page.
