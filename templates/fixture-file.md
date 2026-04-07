# Fixture Composition Pattern

Reference pattern for qa-generate skill. Fixtures are composed via `mergeTests()`, never inheritance.

## File Naming

`tests/e2e/{feature}/{feature}.fixtures.ts` (feature-specific)
`tests/e2e/support/fixtures/{name}.fixture.ts` (shared)

## Structure

```typescript
import { test as base, mergeTests } from '@playwright/test';
import { createUser } from '../support/factories/user.factory';

// ── Pure function: reusable logic, no Playwright dependency ──
function generateAuthHeaders(token: string) {
  return { Authorization: `Bearer ${token}` };
}

// ── Single-capability fixture ──
const authFixture = base.extend<{ authenticatedPage: Page }>({
  authenticatedPage: async ({ page, context }, use) => {
    // Setup: create user + get auth token via API
    const user = createUser({ role: 'buyer' });
    const res = await page.request.post('/api/auth/login', {
      data: { email: user.email, password: user.password },
    });
    const { token } = await res.json();

    // Set auth state
    await context.addCookies([{
      name: 'session',
      value: token,
      domain: 'localhost',
      path: '/',
    }]);

    await use(page);

    // Teardown: clean up created user
    await page.request.delete(`/api/users/${user.id}`, {
      headers: generateAuthHeaders(token),
    });
  },
});

const cartFixture = base.extend<{ cartWithItems: { cartId: string } }>({
  cartWithItems: async ({ page }, use) => {
    const cart = await page.request.post('/api/cart', {
      data: { items: [{ sku: 'TEST-001', qty: 1 }] },
    });
    const cartData = await cart.json();

    await use(cartData);

    // Teardown: clear cart
    await page.request.delete(`/api/cart/${cartData.cartId}`);
  },
});

// ── Composition via mergeTests (not inheritance) ──
export const test = mergeTests(authFixture, cartFixture);
export { expect } from '@playwright/test';
```

## Rules Enforced

| Rule | Pattern |
|------|---------|
| Pure functions first | `generateAuthHeaders()` — no Playwright, easily testable |
| Single capability | Each fixture does ONE thing (auth, cart, not both) |
| mergeTests composition | `mergeTests(a, b)` — never `class extends` |
| Auto-cleanup | Every `use()` has matching teardown after it |
| API setup | Fixtures seed via API, not UI navigation |
