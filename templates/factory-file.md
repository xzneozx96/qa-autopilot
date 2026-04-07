# Data Factory Pattern

Reference pattern for qa-generate skill. Factories create realistic test data with sensible defaults and easy overrides.

## File Naming

`tests/e2e/{feature}/{feature}.factories.ts` (feature-specific)
`tests/e2e/support/factories/{entity}.factory.ts` (shared)

## Structure

```typescript
import { faker } from '@faker-js/faker';

// ── Types ──
interface User {
  email: string;
  password: string;
  firstName: string;
  lastName: string;
  role: 'buyer' | 'seller' | 'admin';
  hasSavedCard: boolean;
}

interface Order {
  userId: string;
  items: Array<{ sku: string; qty: number; price: number }>;
  shippingAddress: string;
}

// ── Factory: sensible defaults + easy overrides ──
export function createUser(overrides: Partial<User> = {}): User {
  return {
    email: faker.internet.email(),
    password: faker.internet.password({ length: 12 }),
    firstName: faker.person.firstName(),
    lastName: faker.person.lastName(),
    role: 'buyer',
    hasSavedCard: false,
    ...overrides,
  };
}

export function createOrder(overrides: Partial<Order> = {}): Order {
  return {
    userId: faker.string.uuid(),
    items: [
      {
        sku: faker.string.alphanumeric(8).toUpperCase(),
        qty: faker.number.int({ min: 1, max: 5 }),
        price: faker.number.float({ min: 9.99, max: 999.99, fractionDigits: 2 }),
      },
    ],
    shippingAddress: faker.location.streetAddress(true),
    ...overrides,
  };
}
```

## Rules Enforced

| Rule | Pattern |
|------|---------|
| Faker for all values | No hardcoded emails, IDs, or strings |
| Sensible defaults | Factory works with zero arguments |
| Easy overrides | `Partial<T>` spread pattern — override any field |
| Pure functions | No side effects, no API calls — that's the test's job |
| Typed | Full TypeScript interfaces for every entity |
