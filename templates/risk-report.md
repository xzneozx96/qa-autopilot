# Risk Report Template

Reference template for qa-trace skill output.

## Format

```markdown
# Risk Report — {Feature Name}

**Generated:** {timestamp}
**Requirement:** {user story summary}

## Risk Score

| Factor | Value | Rationale |
|--------|-------|-----------|
| Probability | {1-3} | {why this probability} |
| Impact | {1-3} | {why this impact} |
| **Score** | **{P×I}** | |
| **Priority** | **{P0/P1/P2/P3}** | |
| **Action** | **{BLOCK/MITIGATE/MONITOR/DOCUMENT}** | |

## Risk Categories

{List applicable: TECH, SEC, PERF, DATA, BUS, OPS}

## Risk Factors

| Factor | Assessment |
|--------|-----------|
| Revenue impact | {description} |
| User impact | {description} |
| Security risk | {yes/no + detail} |
| Compliance required | {yes/no + detail} |
| Previous failures | {yes/no + detail} |
| Complexity | {low/medium/high + why} |
| Usage frequency | {description} |

## AC Completeness Gaps

{Table of gaps found in acceptance criteria, or "None — all AC well-defined"}

| AC ID | Gap Description |
|-------|----------------|
| {id} | {what's missing or vague} |

## Recommendations

{Bullet list of recommendations based on risk level}
```
