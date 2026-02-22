---
ADR: "0007"
Title: Keyword-only arguments for retry API
Status: Accepted
Date: 2026-02-22
Context: Reverse-engineered from implementation
---

# ADR-0007: Keyword-only arguments for retry API

## Status

Accepted

## Context

The `retry` function accepts 7 parameters, several of which are numeric (`attempts`, `timeout`, `wait_initial`, `wait_max`, `wait_jitter`, `wait_exp_base`). Without enforcement, users could easily write:

```python
# What does this mean?
@stamina.retry(ValueError, 10, 45.0, 0.1, 5.0, 1.0, 2)
```

This is error-prone: is `10` the attempts or the timeout? Is `0.1` the initial wait or the jitter? The parameters are not self-documenting when passed positionally.

Tenacity, which stamina wraps, uses a builder/chaining pattern (`Retrying().retry_if_exception_type().stop_after_attempt()`) to avoid this problem. But that pattern is verbose and hard to type-check.

## Decision

Make all parameters keyword-only by using `*` in the function signature:

```python
def retry(
    *,
    on: ExcOrBackoffHook,
    attempts: int | None = 10,
    timeout: float | dt.timedelta | None = 45.0,
    wait_initial: float | dt.timedelta = 0.1,
    wait_max: float | dt.timedelta = 5.0,
    wait_jitter: float | dt.timedelta = 1.0,
    wait_exp_base: float = 2,
) -> ...:
```

The `on` parameter has **no default**, forcing explicit specification. This prevents the mistake of forgetting to specify what to retry on.

The same keyword-only pattern extends to:
- `retry_context(on, *, attempts, timeout, ...)`
- `RetryingCaller.__call__(on, callable_, /, *args, **kwargs)` — `on` and `callable_` are positional-only here because they're different from the retry config parameters

## Consequences

**Benefits:**
- Every call site is self-documenting: `retry(on=ValueError, attempts=3, timeout=30)`
- Impossible to accidentally swap `attempts` and `timeout` or mix up wait parameters
- No default for `on` — forces intentional exception specification
- IDE autocompletion shows parameter names, making the API discoverable
- Type checkers can validate each parameter independently

**Trade-offs:**
- More verbose call sites (must write `attempts=3` instead of just `3`)
- Cannot use partial application as easily (though `RetryingCaller` fills this role)
- Slight departure from Python conventions where short-parameter functions often allow positional args

## Evidence

- `src/stamina/_core.py:727` — `def retry(*, on, ...)` — bare `*` forces keyword-only
- `src/stamina/_core.py:119` — `def retry_context(on, *, ...)` — `on` is positional, rest keyword-only
- `src/stamina/_core.py:278` — `RetryingCaller.__call__(self, on, callable_, /, ...)` — positional-only for call-style API
- `tests/typing/api.py` — all examples use keyword arguments exclusively
