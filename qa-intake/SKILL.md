---
name: qa-intake
description: >
  This skill should be used when a user story and acceptance criteria need to be parsed into a
  structured, machine-readable StructuredRequirement. Extracts actor/action/value,
  normalises acceptance criteria into precondition/action/expected-result triples,
  surfaces NFR criteria, flags spec gaps, and assigns a complexity signal.
---

## Inputs

| Field | Required | Notes |
|---|---|---|
| User story text | yes | Natural language, any format |
| Acceptance criteria | yes | Given/When/Then **or** bullet points |
| Figma URL | no | Passed through to output |

## Processing Rules

1. **User story** — extract WHO (actor), WHAT (action), WHY (value).
2. **Acceptance criteria** — parse each AC into `precondition -> action -> expectedResult`. Tag as `functional` or `nfr`.
3. **NFR criteria** — identify security, performance, reliability, maintainability requirements. Reference [nfr-criteria](../knowledge/nfr-criteria.md) for category definitions and thresholds.
4. **Gap detection** — flag missing or weak specs. Apply severity rules below.
5. **Complexity** — assign `low | medium | high` based on AC count, NFR breadth, and integration surface. Consult [quality-readiness-checklist](../knowledge/adr-quality-readiness-checklist.md).

## Gap Severity Rules

**blocking** — the spec cannot be tested as-is:
- No error/failure paths defined at all
- Vague success criteria (e.g. "works correctly", "handles gracefully")
- Missing actor (no WHO in the story)

**warning** — testable but coverage will be shallow:
- Only happy-path ACs, no sad-path
- No edge cases described
- No performance or load expectations stated

## Output Schema — `StructuredRequirement`

```typescript
type StructuredRequirement = {
  id: string;
  userStory: { actor: string; action: string; value: string; raw: string };
  acceptanceCriteria: Array<{
    id: string;
    precondition: string;
    action: string;
    expectedResult: string;
    type: 'functional' | 'nfr';
    raw: string;
  }>;
  nfrCriteria: Array<{
    category: 'security' | 'performance' | 'reliability' | 'maintainability';
    requirement: string;
    threshold?: string;
  }>;
  gaps: Array<{
    description: string;
    severity: 'blocking' | 'warning';
    suggestion: string;
  }>;
  complexity: 'low' | 'medium' | 'high';
  figmaUrl?: string;
};
```

## Example

**Raw input:**

> As a shopper, I want to add items to my cart so that I can purchase them later.
>
> AC: Given I am on a product page, when I click "Add to Cart", then the item appears in my cart.

**Structured output (single AC shown):**

```json
{
  "id": "req-001",
  "userStory": {
    "actor": "shopper",
    "action": "add items to cart",
    "value": "purchase them later",
    "raw": "As a shopper, I want to add items to my cart so that I can purchase them later."
  },
  "acceptanceCriteria": [
    {
      "id": "ac-001",
      "precondition": "User is on a product page",
      "action": "Click 'Add to Cart'",
      "expectedResult": "Item appears in the cart",
      "type": "functional",
      "raw": "Given I am on a product page, when I click \"Add to Cart\", then the item appears in my cart."
    }
  ],
  "nfrCriteria": [],
  "gaps": [
    {
      "description": "No error path for out-of-stock or quantity limit",
      "severity": "blocking",
      "suggestion": "Add AC for adding an out-of-stock item and exceeding max quantity."
    },
    {
      "description": "No edge cases (duplicate add, zero-price item)",
      "severity": "warning",
      "suggestion": "Define behaviour when the same item is added twice."
    }
  ],
  "complexity": "low"
}
```
