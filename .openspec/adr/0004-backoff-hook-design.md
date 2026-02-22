---
ADR: "0004"
Title: Backoff hook design — bool/float/timedelta return
Status: Accepted
Date: 2026-02-22
Context: Reverse-engineered from implementation and changelog
---

# ADR-0004: Backoff hook design — bool/float/timedelta return

## Status

Accepted

## Context

The `on` parameter of `retry` originally accepted only exception types or tuples of exception types. This works for simple cases ("retry on ValueError") but not for fine-grained control ("retry on HTTP 503 but not 404") or server-directed backoff ("retry after Retry-After seconds").

The API needed to evolve to support:
1. **Conditional retry** — decide per-exception whether to retry
2. **Custom backoff** — override exponential backoff with a server-specified duration (e.g., Retry-After header)

## Decision

Overload the `on` parameter to also accept a callable ("backoff hook") with a multi-type return value:

```python
BackoffHook = Callable[[Exception], bool | float | dt.timedelta]
```

**Return value semantics:**
- `True` → retry with default exponential backoff
- `False` → do not retry, reraise exception
- `float` → retry with this many seconds as custom backoff (overrides exponential)
- `dt.timedelta` → converted to seconds, same as float
- `None` → treated as `False` with a deprecation warning (future error)

**Key design choices:**

1. **Single parameter, not separate predicates.** Rather than adding `retry_if` and `custom_backoff` parameters, the `on` parameter was extended. This keeps the API surface small.
2. **Custom backoff bypasses the formula.** When a hook returns a float/timedelta, `wait_max`, `wait_jitter`, and `wait_exp_base` do NOT apply. The value is used as-is. This is intentional — server-directed backoff (like Retry-After) should not be modified.
3. **Custom backoff respects testing mode.** Even custom backoffs return `0.0` in testing mode.
4. **Piggyback state for custom backoff.** The hook is called by Tenacity's retry predicate, but the wait decision happens in a separate callback. To bridge them, the custom backoff value is stored on `RetryCallState` via `setattr` and read back in the `wait` callback.
5. **None return warns, doesn't crash.** A hook returning `None` (easy mistake: forgetting `return True`) produces a warning rather than an error, for backwards compatibility.

## Consequences

**Benefits:**
- Single `on` parameter handles three use cases (type matching, conditional retry, custom backoff)
- Natural for the Retry-After pattern: `on=lambda exc: exc.response.headers.get('Retry-After', False)`
- Backwards compatible — existing exception type usage unchanged
- Gradual migration path for None returns (warning now, error later)

**Trade-offs:**
- Overloaded return type — `bool | float | timedelta` requires understanding the semantics of each
- The piggyback state pattern is fragile (relies on Tenacity not using the attribute name)
- Cannot combine custom backoff with jitter (the formula is entirely bypassed)
- Type checking is complex — `ExcOrBackoffHook` union type is hard to narrow

## Evolution

- **v24.3.0** — `on` gained callable support (bool return only)
- **v25.2.0** — `on` callable can return float/timedelta for custom backoff

## Evidence

- `src/stamina/_core.py:62-65` — `BackoffHook` and `ExcOrBackoffHook` type aliases
- `src/stamina/_core.py:71-116` — `_TenacityBackoffCallbackAdapter` (the bridge)
- `src/stamina/_core.py:93-104` — None return warning
- `src/stamina/_core.py:109-110` — timedelta conversion
- `src/stamina/_core.py:619-642` — `_jittered_backoff_for_rcs` reads piggyback state
- `src/stamina/_core.py:770-790` — docstring describing the full `on` semantics
