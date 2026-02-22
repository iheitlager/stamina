## Remediation Roadmap

### Quick Wins (< 1 day each)
1. 🔴 **Audit and categorize external dependencies** — 86 uncategorized dependencies (Technology, score: 0.80)
   - Unknown packages: external:, external:AttributeError, external:Exception, external:RETRIES_TOTAL, external:RuntimeError, external:SystemError, external:TypeError, external:ValueError, external:_config, external:_logging
2. 🟡 **Use specific exception types and log errors** — Exception handling anti-pattern (Operational, score: 0.50)
   - Module: `src/stamina/instrumentation/_hooks.py`
   - Pattern: swallowed exception

### Short-term (1-5 days each)
3. 🔴 **Reduce outgoing dependencies or increase incoming (make module more stable)** — High instability (Architecture, score: 1.00)
   - Module: `docs/conf.py`
   - Instability: 1.00 (threshold: 0.8)
4. 🔴 **Extract sub-functions or simplify conditional logic** — High cyclomatic complexity (Complexity, score: 0.88)
   - Module: `src/stamina/_core.py#13__RetryContextIterator`
   - Cyclomatic complexity: 24 (threshold: 10.0)
5. 🟠 **Create OpenSpec specifications for undocumented modules** — 19 undocumented module(s) (Architecture, score: 0.70)
   - Modules without spec coverage: docs, docs/conf.py, noxfile.py, stamina, stamina/__init__.py, stamina/_config.py, stamina/_core.py, stamina/instrumentation, stamina/typing.py, tests

### Medium-term (1-2 weeks each)
6. 🟡 **Split into smaller, focused units** — Oversized entity (Complexity, score: 0.49)
   - Module: `tests/test_instrumentation.py#09_TestSetOnRetryHooks`
   - Lines of code: 247 (threshold: 200)
7. 🟡 **Verify if still needed; remove if unused to reduce maintenance burden** — Potentially dead code (Complexity, score: 0.41)
   - Module: `src/stamina/_core.py#17_retry.01_retry_decorator`
   - No incoming calls detected (fan-in: 0)

### Strategic (> 2 weeks)
8. 🔴 **Prioritize refactoring to reduce complexity in frequently changed code** — High churn combined with high complexity (Knowledge, score: 1.00)
   - Module: `src/stamina/_core.py`
   - Churn rate: 10.333 commits/day
9. 🔴 **Schedule review and update of stale critical modules** — Stale critical code path (Knowledge, score: 0.90)
   - Module: `src/stamina/__init__.py`
   - Last modified: 1240 days ago
10. 🔴 **Cross-train team members on this module** — Bus factor = 1 (single contributor) (Knowledge, score: 0.80)
   - Module: `docs/conf.py`
   - Primary contributor: Hynek Schlawack
11. 🟡 **Balance workload distribution across team members** — Uneven contributor concentration (Knowledge, score: 0.55)
   - Module: `<project>`
   - Gini coefficient: 0.50

### Summary

Total actions: 11
- Quick Wins (< 1 day each): 2
- Short-term (1-5 days each): 3
- Medium-term (1-2 weeks each): 2
- Strategic (> 2 weeks): 4
