---
ADR: "0006"
Title: Testing mode with cap semantics
Status: Accepted
Date: 2026-02-22
Context: Reverse-engineered from implementation and changelog
---

# ADR-0006: Testing mode with cap semantics

## Status

Accepted

## Context

In test suites, retrying with real backoff is undesirable — it makes tests slow and non-deterministic. stamina needed a testing mode that:

1. Eliminates backoff delays (all waits → 0.0)
2. Controls the number of retry attempts
3. Works as both a global setting and a scoped context manager

The challenge: how should testing mode interact with per-decorator `attempts` values?

Consider a decorator `@retry(on=ValueError, attempts=3)`. If testing mode sets `attempts=1`:
- **Override** (cap=False): always use testing's value → 1 attempt, even though the decorator says 3
- **Cap** (cap=True): use the minimum → 1 attempt (because 1 < 3)

But what about `@retry(on=ValueError, attempts=1)` with testing `attempts=5`?
- **Override** (cap=False): use testing's value → 5 attempts (more than the decorator intended!)
- **Cap** (cap=True): use the minimum → 1 attempt (respects the decorator's intent)

## Decision

Support both behaviors via a `cap` parameter on `set_testing`:

```python
set_testing(True, attempts=N, cap=False)  # Override: always use N
set_testing(True, attempts=N, cap=True)   # Cap: use min(N, decorator_attempts)
```

**Key design choices:**

1. **Default is override (cap=False).** This is the simple, predictable behavior — "in testing, always do 1 attempt." Most test suites want this.
2. **Cap mode for edge cases.** When `cap=True`, `min(testing_attempts, user_attempts)` is used. If the user's attempts is `None` (unlimited), the testing value is used. This prevents testing mode from *increasing* retry attempts beyond what the code specifies.
3. **Zero backoff always.** Both modes force all backoff computations to return `0.0`, including custom backoff hooks.
4. **Context manager support.** `set_testing()` returns a context manager that restores the previous state on exit. Context managers can be nested, each level restoring its predecessor.
5. **Separate from set_active.** `set_active(False)` disables retrying entirely (1 attempt, no retry). `set_testing(True)` keeps retrying active but with controlled attempts and zero backoff. This distinction matters for testing retry logic itself.

## Consequences

**Benefits:**
- Simple default (`set_testing(True)`) → 1 attempt, 0 backoff — covers 90% of test needs
- Cap mode prevents testing from accidentally increasing retries
- Context manager enables scoped testing in individual test functions
- Nesting enables fine-grained control (e.g., outer fixture sets `attempts=1`, inner test sets `attempts=3`)

**Trade-offs:**
- Two modes (cap vs override) add conceptual complexity
- `cap=True` with `None` user attempts returns the testing value, which may surprise
- Global state means testing mode affects all retry calls, not just the one being tested
- Cannot set different testing parameters for different retry decorators in the same test

## Evolution

- **v24.3.0** — `set_testing` and `is_testing` added
- **v25.1.0** — `cap` parameter added, context manager support added

## Evidence

- `src/stamina/_config.py:16-45` — `_Testing` class with `get_attempts` method
- `src/stamina/_config.py:42-43` — cap logic: `min(self.attempts, non_testing_attempts or self.attempts)`
- `src/stamina/_config.py:174-195` — `set_testing` function with context manager return
- `src/stamina/_config.py:158-171` — `_RestoreTestingCM` context manager
- `src/stamina/_core.py:552-564` — `_apply_maybe_test_mode_to_tenacity_kw` applies testing overrides
- `src/stamina/_core.py:656-657` — backoff returns 0.0 in testing mode
- `tests/test_config.py:44-116` — `TestTesting` class with cap, nesting, and exception safety tests
