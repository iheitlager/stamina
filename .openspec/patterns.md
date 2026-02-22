---
Title: Design Patterns Reference
Date: 2026-02-22
Context: Reverse-engineered from implementation
---

# Design Patterns Reference

This document catalogs well-implemented design patterns found in stamina's codebase. Each pattern is identified by its standard name, with the specific implementation details and rationale.

---

## 1. Null Object Pattern

**Where**: `src/stamina/_core.py:431-447`

**Implementation**: `_LazyNoAsyncRetry` and `_STOP_NO_RETRY`

When stamina is deactivated (`set_active(False)`), the retry machinery still needs to execute code once. Rather than adding `if config.is_active` branches throughout the codebase, stamina uses null objects that conform to the same interface but do nothing:

```python
_STOP_NO_RETRY = _t.stop_after_attempt(1)

class _LazyNoAsyncRetry:
    """Allows us a free null object pattern using non-retries and avoid None."""
    def __aiter__(self):
        return _t.AsyncRetrying(
            reraise=True, stop=_STOP_NO_RETRY, sleep=_smart_sleep
        ).__aiter__()
```

The `_RetryContextIterator.__iter__` checks `CONFIG.is_active` once at the top and either enters the full retry loop or yields a single no-retry attempt. The async path uses `_LAZY_NO_ASYNC_RETRY` as a default that is replaced with a real `AsyncRetrying` when active.

**Why it works well**: Eliminates None checks and conditional branches in the hot path. The inactive codepath uses the same Tenacity machinery (just limited to 1 attempt), so behavior is consistent.

---

## 2. Protocol-based Extension (Structural Typing)

**Where**: `src/stamina/instrumentation/_data.py:69-88`

**Implementation**: `RetryHook(Protocol)`

```python
class RetryHook(Protocol):
    def __call__(
        self, details: RetryDetails
    ) -> None | AbstractContextManager[None]: ...
```

Rather than requiring hooks to inherit from a base class, stamina defines `RetryHook` as a `typing.Protocol`. Any callable matching the signature — a function, a lambda, a class with `__call__` — satisfies the protocol without explicit registration.

**Why it works well**: Maximum flexibility for users. A simple function `def my_hook(details): ...` works. A class with state works. A contextmanager-decorated function works. No imports from stamina needed to create a hook — just match the signature.

---

## 3. Lazy Factory (RetryHookFactory)

**Where**: `src/stamina/instrumentation/_data.py:91-103`, `src/stamina/instrumentation/_hooks.py:15-28`

**Implementation**: `RetryHookFactory` wrapping `Callable[[], RetryHook]`

```python
@dataclass(frozen=True)
class RetryHookFactory:
    hook_factory: Callable[[], RetryHook]

# Usage:
PrometheusOnRetryHook = RetryHookFactory(init_prometheus)
StructlogOnRetryHook = RetryHookFactory(init_structlog)
LoggingOnRetryHook = RetryHookFactory(init_logging)
```

Hook initialization (importing prometheus_client, creating Counters, calling `structlog.get_logger()`) is deferred until the first retry is actually scheduled. The factory is resolved by `init_hooks()` which calls `hook_factory()` on factories and passes through direct hooks.

**Why it works well**: Import cost of optional dependencies (prometheus_client, structlog) is paid only when retries actually occur. Applications on the happy path never pay the cost. structlog's logger is created after the user has configured their processor chain.

---

## 4. Piggyback State

**Where**: `src/stamina/_core.py:68`, `src/stamina/_core.py:114`, `src/stamina/_core.py:619-642`

**Implementation**: `_CUSTOM_BACKOFF_ATTR` on Tenacity's `RetryCallState`

```python
_CUSTOM_BACKOFF_ATTR = "_stamina_custom_backoff"

# In the retry predicate (called by Tenacity to decide whether to retry):
setattr(retry_state, _CUSTOM_BACKOFF_ATTR, custom_backoff)

# In the wait callback (called by Tenacity to decide how long to wait):
if (custom_backoff := getattr(rcs, _CUSTOM_BACKOFF_ATTR, None)) is not None:
    delattr(rcs, _CUSTOM_BACKOFF_ATTR)
    return custom_backoff
```

Tenacity separates the "should we retry?" decision (retry predicate) from the "how long to wait?" decision (wait callback). stamina's backoff hooks need to communicate a custom wait value from the predicate to the wait callback. Rather than using module-level global state, the custom backoff is attached to Tenacity's existing `RetryCallState` object using `setattr`.

**Why it works well**: No global state, no thread-safety concerns, no additional data structures. The value lives on the object that's already being passed through the pipeline. The `delattr` after reading ensures it's consumed exactly once. The source code acknowledges this is "naughty but better than global state."

---

## 5. Context Manager as Return Value (Dual-Use API)

**Where**: `src/stamina/_config.py:158-171`, `src/stamina/_config.py:174-195`

**Implementation**: `set_testing()` returns `_RestoreTestingCM`

```python
def set_testing(testing, *, attempts=1, cap=False):
    old = CONFIG.testing
    CONFIG.testing = _Testing(attempts, cap) if testing else None
    return _RestoreTestingCM(old)

class _RestoreTestingCM:
    def __init__(self, old):
        self.old = old
    def __enter__(self):
        pass  # State already set
    def __exit__(self, *_):
        CONFIG.testing = self.old
```

`set_testing()` has a dual-use design:
- **Fire-and-forget**: `stamina.set_testing(True)` — sets testing mode, return value ignored
- **Scoped**: `with stamina.set_testing(True):` — sets testing mode, restores on exit

The context manager's `__enter__` is a no-op because the state mutation happens in `set_testing()` itself (before the `with` block). `__exit__` restores the captured previous state.

**Why it works well**: One function serves two use patterns without API duplication. The nesting behavior falls out naturally — each `set_testing` captures its predecessor. Exception safety is guaranteed by `__exit__` always running.

---

## 6. Decorator with Callable Type Detection

**Where**: `src/stamina/_core.py:842-910`

**Implementation**: `retry_decorator` inspects the wrapped callable

```python
def retry_decorator(wrapped):
    name = guess_name(wrapped)

    if isgeneratorfunction(wrapped):
        @wraps(wrapped)
        def sync_gen_inner(*args, **kw):
            for attempt in retry_ctx.with_name(name, args, kw):
                with attempt:
                    return (yield from wrapped(*args, **kw))
        return sync_gen_inner

    if isasyncgenfunction(wrapped):
        # ... async generator wrapper with asend/athrow/aclose support
        return async_gen_inner

    if not iscoroutinefunction(wrapped):
        # ... sync function wrapper
        return sync_inner

    # ... async function wrapper
    return async_inner
```

A single `@stamina.retry(on=...)` decorator handles four distinct callable types:
1. **Sync functions** — called directly
2. **Async functions** — awaited
3. **Sync generators** — `yield from` delegation with send/throw support
4. **Async generators** — full asend/athrow/aclose protocol

The detection uses `inspect` module functions and follows a specific order: generators before async (since async generators are both).

**Why it works well**: Users don't need to know which wrapper to use — one decorator handles everything. The detection order is correct (generator checks before coroutine checks, because async generators are also coroutine-like). Each wrapper uses `functools.wraps` to preserve the original function's metadata.
