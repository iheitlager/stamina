# Due Diligence Report: stamina
**Date**: 2026-02-22 | **Overall Risk**: RED

## Risk Overview

| Dimension | Score | Level | Key Finding |
|-----------|-------|-------|-------------|
| Technology | 0.80 | 🔴 CRITICAL | 86 uncategorized dependencies |
| Architecture | 0.76 | 🟠 HIGH | High instability |
| Complexity | 0.39 | 🟡 MEDIUM | High cyclomatic complexity |
| Knowledge | 0.70 | 🟠 HIGH | High churn combined with high complexity |
| Operational | 0.50 | 🟡 MEDIUM | Exception handling anti-pattern |

## Top Risks

1. **High churn combined with high complexity** — Churn rate: 10.333 commits/day; Cyclomatic complexity: 72; Total commits: 62
2. **High churn combined with high complexity** — Churn rate: 4.167 commits/day; Cyclomatic complexity: 59; Total commits: 25
3. **High instability** — Instability: 1.00 (threshold: 0.8); Afferent coupling (Ca): 0; Efferent coupling (Ce): 5

## Recommendations

1. Prioritize refactoring to reduce complexity in frequently changed code
2. Reduce outgoing dependencies or increase incoming (make module more stable)
