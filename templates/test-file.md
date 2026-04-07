# Playwright Test File Pattern

Reference pattern for qa-generate skill. Every generated test file MUST follow this structure.

## File Naming

`tests/e2e/{feature}/{feature}.spec.ts`

## Structure

```typescript
import { test, expect } from './fixtures';
import { createUser, createOrder } from './factories';

test.describe('Checkout Flow', () => {
  test('1.3-E2E-001 buyer completes checkout with saved card @p0 @smoke', async ({
    page,
    apiContext,
  }) => {
    // ── ARRANGE: Seed data via API (never UI) ──
    const user = createUser({ hasSavedCard: true });
    const userRes = await apiContext.post('/api/users', { data: user });
    const userId = (await userRes.json()).id;

    const order = createOrder({ userId, items: 3 });
    await apiContext.post('/api/orders', { data: order });

    // ── INTERCEPT: Register listeners BEFORE navigation ──
    const checkoutResponse = page.waitForResponse(
      resp => resp.url().includes('/api/checkout') && resp.status() === 200
    );

    // ── ACT: Navigate and interact ──
    await page.goto('/checkout');
    await expect(page.getByTestId('checkout-summary')).toBeVisible();

    await page.getByTestId('saved-card-radio').click();
    await page.getByTestId('pay-button').click();
    await checkoutResponse;

    // ── ASSERT: Verify expected outcome ──
    await expect(page.getByTestId('confirmation-message')).toBeVisible();
    await expect(page.getByTestId('order-number')).not.toBeEmpty();
  });

  test('1.3-E2E-002 buyer sees error on declined card @p0 @regression', async ({
    page,
    apiContext,
  }) => {
    // Same ARRANGE → INTERCEPT → ACT → ASSERT pattern
    const user = createUser({ hasSavedCard: true, cardStatus: 'declined' });
    const userRes = await apiContext.post('/api/users', { data: user });

    const declineResponse = page.waitForResponse(
      resp => resp.url().includes('/api/checkout') && resp.status() === 402
    );

    await page.goto('/checkout');
    await page.getByTestId('saved-card-radio').click();
    await page.getByTestId('pay-button').click();
    await declineResponse;

    await expect(page.getByTestId('error-banner')).toBeVisible();
    await expect(page.getByTestId('error-banner')).toContainText('declined');
  });
});
```

## Rules Enforced

| Rule | Pattern |
|------|---------|
| Network-first | `waitForResponse()` registered BEFORE `click()`/`goto()` |
| Selector hierarchy | `getByTestId()` > `getByRole()` > `getByText()` — never CSS class |
| No hard waits | Zero `waitForTimeout()` — use `waitForResponse()` or `toBeVisible()` |
| API seeding | Data created via `apiContext.post()`, never via UI |
| Faker data | `createUser()` uses faker internally — no hardcoded values |
| Tags | `@p0`/`@p1`/`@p2`/`@p3` + `@smoke`/`@regression` in test title |
| Test ID | `{EPIC}.{STORY}-E2E-{SEQ}` format in test title |
