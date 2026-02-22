---
Domain: Retry Core
Version: "1.1"
Status: Implemented
Date: 2026-02-22
Spec-ID: "001"
---

# Spec 001: Retry Core

## Overview

The retry core provides the fundamental retry machinery for stamina: the `retry` decorator, the `retry_context` iterator, and the `Attempt` context manager. It implements exponential backoff with jitter, custom backoff hooks, exception matching, and supports sync functions, async functions, sync generators, and async generators.

### Key Capabilities

- Decorator-based retry (`@stamina.retry`)
- Iterator-based retry (`stamina.retry_context`)
- Exponential backoff with configurable parameters
- Custom backoff hooks (callable returning bool/float/timedelta)
- Exception type matching (single, tuple, or callable predicate)
- Sync, async, sync generator, and async generator support
- `datetime.timedelta` support for all time parameters
- Async library detection (asyncio and Trio) via sniffio
- Keyword-only arguments to prevent positional mistakes

## Keywords

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", and "MAY" in this document are to be interpreted as described in RFC 2119.

---

## Requirements

### Requirement RC-1: retry decorator signature

The `retry` function MUST accept only keyword arguments with the following parameters:
- `on`: exception type, tuple of exception types, or backoff hook callable (required, no default)
- `attempts`: maximum total attempts, `int | None`, default `10`
- `timeout`: maximum total time in seconds, `float | dt.timedelta | None`, default `45.0`
- `wait_initial`: minimum backoff before first retry, `float | dt.timedelta`, default `0.1`
- `wait_max`: maximum backoff time, `float | dt.timedelta`, default `5.0`
- `wait_jitter`: maximum random jitter added to backoff, `float | dt.timedelta`, default `1.0`
- `wait_exp_base`: exponential base for backoff computation, `float`, default `2`

**Implementation**: `src/stamina/_core.py:727-840`
**ADR**: ADR-0007

#### Scenario RC-S1: Successful call with no errors

- GIVEN a function decorated with `@stamina.retry(on=Exception)`
- WHEN the function returns successfully on the first call
- THEN the return value MUST be passed through unchanged

**Tests**: `tests/test_sync.py::test_ok`, `tests/test_async.py::test_ok`

#### Scenario RC-S11: Timedelta parameters accepted

- GIVEN time parameters (`timeout`, `wait_initial`, `wait_max`, `wait_jitter`) are `datetime.timedelta` values
- WHEN the decorated function is called
- THEN the timedelta values MUST be converted to seconds and the function MUST behave identically to float parameters

**Tests**: `tests/test_sync.py::test_ok` (parametrized with `dt.timedelta(days=1)`)

---

### Requirement RC-2: retry returns a decorator

The `retry` function MUST return a decorator that wraps the target callable, preserving its signature and metadata via `functools.wraps`.

**Implementation**: `src/stamina/_core.py:842-910`

---

### Requirement RC-3: backoff formula

The backoff for retry attempt number *n* MUST be computed as:

```
min(wait_max, wait_initial * wait_exp_base^(n-1) + random(0, wait_jitter))
```

Since `x^0 = 1`, the first backoff MUST be in the interval `[wait_initial, wait_initial + wait_jitter]`.

**Implementation**: `src/stamina/_core.py:645-661`

#### Scenario RC-S37: Backoff formula verification

- GIVEN `retry_context(on=ValueError, wait_initial=0.1, wait_exp_base=2, wait_jitter=0, wait_max=5.0)`
- WHEN `_backoff_for_attempt_number(n)` is called for n=1, 2, 3
- THEN the result MUST be `0.1`, `0.2`, `0.4` respectively (following `0.1 * 2^(n-1)`)

**Tests**: `tests/test_sync.py::test_backoff_computation_clamps`

---

### Requirement RC-4: backoff clamping

The computed backoff MUST NOT exceed `wait_max`.

**Implementation**: `src/stamina/_core.py:661`

#### Scenario RC-S19: Backoff clamped to wait_max

- GIVEN `retry_context(on=ValueError, wait_max=0.42)`
- WHEN backoff is computed for various attempt numbers
- THEN the backoff MUST never exceed `0.42`

**Tests**: `tests/test_sync.py::test_backoff_computation_clamps`

---

### Requirement RC-5: last exception reraise

If all retries are exhausted, the decorator MUST reraise the last exception (not wrap it).

**Implementation**: `src/stamina/_core.py:534` (tenacity `reraise=True`)

#### Scenario RC-S38: All retries exhausted reraises last exception

- GIVEN a function decorated with `@stamina.retry(on=ValueError, attempts=2, wait_max=0)`
- WHEN the function raises `ValueError` on every attempt
- THEN the final `ValueError` MUST be reraised directly (not wrapped in a tenacity exception)

**Tests**: `tests/test_sync.py::TestBackoffHookSync::test_backoff_hook_returns_float` (exhaustion path via `attempts=3`)

---

### Requirement RC-6: on parameter -- exception type

When `on` is a single exception type or tuple of exception types, the retry machinery MUST retry only when the raised exception is an instance of the specified type(s).

**Implementation**: `src/stamina/_core.py:493-496`

#### Scenario RC-S2: Retry on matching exception

- GIVEN a function decorated with `@stamina.retry(on=ValueError)`
- WHEN the function raises `ValueError` on the first call and succeeds on the second
- THEN the function MUST be retried and the successful return value MUST be returned

**Tests**: `tests/test_sync.py::test_retries`, `tests/test_async.py::test_retries`

#### Scenario RC-S3: Non-matching exception passes through

- GIVEN a function decorated with `@stamina.retry(on=ValueError)`
- WHEN the function raises `TypeError`
- THEN `TypeError` MUST propagate immediately without retry

**Tests**: `tests/test_sync.py::test_wrong_exception`, `tests/test_async.py::test_wrong_exception`

#### Scenario RC-S4: Exception type tuple matching

- GIVEN the `on` parameter is `(ValueError,)` (a tuple)
- WHEN the function raises `ValueError`
- THEN the function MUST be retried

**Tests**: `tests/conftest.py::_on` (parametrized fixture), `tests/test_sync.py::test_retries`

---

### Requirement RC-7: on parameter -- backoff hook callable

When `on` is a callable (backoff hook), it MUST be called with the raised exception. If it returns `True`, the exception MUST be retried. If it returns `False`, the exception MUST NOT be retried.

**Implementation**: `src/stamina/_core.py:71-116`
**ADR**: ADR-0004

#### Scenario RC-S5: Backoff hook callable matching

- GIVEN the `on` parameter is `lambda exc: isinstance(exc, ValueError)`
- WHEN the function raises `ValueError`
- THEN the hook MUST return `True` and the function MUST be retried

**Tests**: `tests/conftest.py::_on` (parametrized fixture), `tests/test_sync.py::test_retries`

---

### Requirement RC-8: backoff hook returning float

When a backoff hook returns a `float`, that value MUST be used as the backoff duration for the next retry, overriding the default exponential backoff formula. The other backoff parameters (wait_max, wait_jitter, etc.) MUST NOT apply to custom backoffs.

**Implementation**: `src/stamina/_core.py:106-116`, `src/stamina/_core.py:619-642`
**ADR**: ADR-0004

#### Scenario RC-S6: Backoff hook returning float

- GIVEN the `on` parameter is `lambda exc: 0.0`
- WHEN the function raises an exception
- THEN the function MUST be retried with a custom backoff of 0.0 seconds

**Tests**: `tests/test_sync.py::TestBackoffHookSync::test_backoff_hook_returns_float`

#### Scenario RC-S8: Backoff hook extracting backoff from exception

- GIVEN a backoff hook that reads `exc.backoff` and returns it as a float
- WHEN the function raises `CustomBackoffError(backoff=0)`
- THEN the extracted backoff MUST be used, overriding default exponential backoff

**Tests**: `tests/test_sync.py::TestBackoffHookSync::test_backoff_hook_custom_backoff_from_exception`

---

### Requirement RC-9: backoff hook returning timedelta

When a backoff hook returns a `datetime.timedelta`, it MUST be converted to seconds and used as the custom backoff duration.

**Implementation**: `src/stamina/_core.py:109-110`
**ADR**: ADR-0004

#### Scenario RC-S7: Backoff hook returning timedelta

- GIVEN the `on` parameter is `lambda exc: dt.timedelta(seconds=0)`
- WHEN the function raises an exception
- THEN the timedelta MUST be converted to seconds and used as the custom backoff

**Tests**: `tests/test_sync.py::TestBackoffHookSync::test_backoff_hook_returns_timedelta`

---

### Requirement RC-10: backoff hook returning None

When a backoff hook returns `None`, the retry machinery MUST treat it as `False` (no retry) and MUST emit a `UserWarning` indicating the hook returned None and that this will be an error in a future version.

**Implementation**: `src/stamina/_core.py:93-104`
**ADR**: ADR-0004

#### Scenario RC-S9: Backoff hook returning None warns

- GIVEN the `on` parameter is `lambda exc: None`
- WHEN the function raises an exception
- THEN a `UserWarning` MUST be emitted mentioning "Backoff hook" and "returned None", and the exception MUST NOT be retried

**Tests**: `tests/test_sync.py::TestBackoffHookSync::test_backoff_hook_returns_none`

---

### Requirement RC-11: timedelta parameter conversion

All time-related parameters (`timeout`, `wait_initial`, `wait_max`, `wait_jitter`) MUST accept `datetime.timedelta` values and convert them to seconds via `.total_seconds()`.

**Implementation**: `src/stamina/_core.py:500-510`

---

### Requirement RC-12: timeout=0 warning

When `timeout=0` (int, float, or timedelta) is passed, the retry machinery MUST emit a `UserWarning` with the message "timeout=0 means no retries will be attempted" and the warning's filename MUST point to the caller's code.

**Implementation**: `src/stamina/_core.py:512-517`

#### Scenario RC-S12: timeout=0 emits warning

- GIVEN `timeout=0` (int, float, or timedelta)
- WHEN `retry_context` or `retry` is called
- THEN a `UserWarning` MUST be emitted with "timeout=0 means no retries will be attempted" and the filename MUST point to the caller

**Tests**: `tests/test_sync.py::test_timeout_zero_warns`

---

### Requirement RC-13: attempts and timeout combination

The `attempts` and `timeout` parameters MUST be combinable. When both are specified, retries MUST stop when either limit is reached first. When both are `None`, retries MUST continue indefinitely.

**Implementation**: `src/stamina/_core.py:709-724`

#### Scenario RC-S21: attempts=None and timeout=None yields stop_never

- GIVEN `_make_stop(attempts=None, timeout=None)`
- WHEN the stop condition is evaluated
- THEN it MUST be `tenacity.stop_never` (unbounded retries)

**Tests**: `tests/test_sync.py::TestMakeStop::test_never`

---

### Requirement RC-14: sync function decoration

When the decorated callable is a regular (non-generator, non-coroutine) function, the decorator MUST wrap it so that on each retry, the function is called again from scratch.

**Implementation**: `src/stamina/_core.py:892-900`

---

### Requirement RC-15: async function decoration

When the decorated callable is a coroutine function, the decorator MUST wrap it so that on each retry, the coroutine is awaited again from scratch. It MUST support both asyncio and Trio.

**Implementation**: `src/stamina/_core.py:902-908`

---

### Requirement RC-16: sync generator decoration

When the decorated callable is a generator function, the decorator MUST wrap it so that on each retry, the generator is restarted from the beginning. The wrapper MUST support `send()` and `throw()` on the inner generator, and MUST propagate the generator's return value via `StopIteration.value`.

**Implementation**: `src/stamina/_core.py:845-853`

#### Scenario RC-S22: Sync generator retry

- GIVEN a generator function decorated with `@stamina.retry(on=ValueError)`
- WHEN the generator raises `ValueError` before yielding
- THEN the generator MUST be restarted from the beginning

**Tests**: `tests/test_sync.py::TestGeneratorFunctionDecoration::test_retries`

#### Scenario RC-S23: Sync generator retry after yield

- GIVEN a generator function that yields, then raises `ValueError`
- WHEN decorated with `@stamina.retry(on=ValueError)`
- THEN the generator MUST be restarted and yield values from the beginning again

**Tests**: `tests/test_sync.py::TestGeneratorFunctionDecoration::test_retries_also_after_yields`

#### Scenario RC-S24: Sync generator send() forwarding

- GIVEN a generator function decorated with `@stamina.retry`
- WHEN `send(value)` is called on the resulting generator
- THEN the value MUST reach the wrapped generator unchanged

**Tests**: `tests/test_sync.py::TestGeneratorFunctionDecoration::test_forwards_send`

#### Scenario RC-S25: Sync generator throw() forwarding

- GIVEN a generator function decorated with `@stamina.retry`
- WHEN `throw(exc)` is called on the resulting generator
- THEN the exception MUST reach the wrapped generator, and if the wrapped generator raises it, the retry decorator MUST restart the generator

**Tests**: `tests/test_sync.py::TestGeneratorFunctionDecoration::test_forwards_throw`

#### Scenario RC-S26: Sync generator return value propagation

- GIVEN a generator function that yields values and then returns a value
- WHEN decorated with `@stamina.retry`
- THEN the return value MUST be propagated via `StopIteration.value`

**Tests**: `tests/test_sync.py::TestGeneratorFunctionDecoration::test_wrapper_passes_through_return_value`

#### Scenario RC-S39: Sync generator happy path

- GIVEN a generator function decorated with `@stamina.retry(on=Exception)`
- WHEN the generator yields without error
- THEN the yielded values MUST be passed through unchanged

**Tests**: `tests/test_sync.py::TestGeneratorFunctionDecoration::test_ok`

#### Scenario RC-S40: Sync generator non-matching exception

- GIVEN a generator function decorated with `@stamina.retry(on=ValueError)`
- WHEN the generator raises `TypeError`
- THEN `TypeError` MUST propagate immediately without retry

**Tests**: `tests/test_sync.py::TestGeneratorFunctionDecoration::test_wrong_exception`

---

### Requirement RC-17: async generator decoration

When the decorated callable is an async generator function, the decorator MUST wrap it so that on each retry, the async generator is restarted from the beginning. The wrapper MUST support `asend()`, `athrow()`, and `aclose()` on the inner generator. It MUST handle `GeneratorExit` by closing the inner generator.

**Implementation**: `src/stamina/_core.py:855-890`

#### Scenario RC-S27: Async generator retry

- GIVEN an async generator function decorated with `@stamina.retry(on=ValueError)`
- WHEN the generator raises `ValueError` before yielding
- THEN the async generator MUST be restarted from the beginning

**Tests**: `tests/test_async.py::TestAsyncGeneratorFunctionDecoration::test_retries`

#### Scenario RC-S28: Async generator asend() forwarding

- GIVEN an async generator function decorated with `@stamina.retry`
- WHEN `asend(value)` is called on the resulting async generator
- THEN the value MUST reach the wrapped async generator unchanged

**Tests**: `tests/test_async.py::TestAsyncGeneratorFunctionDecoration::test_forwards_asend`

#### Scenario RC-S29: Async generator athrow() forwarding

- GIVEN an async generator function decorated with `@stamina.retry`
- WHEN `athrow(exc)` is called on the resulting async generator
- THEN the exception MUST reach the wrapped async generator, and if the wrapped generator raises it, the retry decorator MUST restart the generator

**Tests**: `tests/test_async.py::TestAsyncGeneratorFunctionDecoration::test_forwards_athrow`

#### Scenario RC-S30: Empty async generator

- GIVEN an async generator function that yields nothing
- WHEN decorated with `@stamina.retry` and iterated
- THEN it MUST produce an empty sequence and exit cleanly

**Tests**: `tests/test_async.py::TestAsyncGeneratorFunctionDecoration::test_stops_when_wrapped_generator_is_empty`

#### Scenario RC-S31: Async generator athrow suppressed

- GIVEN an async generator that catches and suppresses a thrown exception
- WHEN `athrow(exc)` is called
- THEN `StopAsyncIteration` MUST be raised (iteration ends cleanly)

**Tests**: `tests/test_async.py::TestAsyncGeneratorFunctionDecoration::test_athrow_that_gets_suppressed`

#### Scenario RC-S32: Async generator retry after yield

- GIVEN an async generator that yields a value then raises on the second iteration
- WHEN decorated with `@stamina.retry`
- THEN the generator MUST restart and yield from the beginning again

**Tests**: `tests/test_async.py::TestAsyncGeneratorFunctionDecoration::test_retries_also_after_yields`

#### Scenario RC-S33: Async method retry

- GIVEN an async method on a class decorated with `@stamina.retry`
- WHEN the method raises on the first call
- THEN it MUST be retried and succeed on the next attempt

**Tests**: `tests/test_async.py::test_retries_method`, `tests/test_async.py::TestAsyncGeneratorFunctionDecoration::test_retries_method`

#### Scenario RC-S41: Async generator happy path

- GIVEN an async generator function decorated with `@stamina.retry(on=Exception)`
- WHEN the generator yields without error
- THEN the yielded values MUST be passed through unchanged

**Tests**: `tests/test_async.py::TestAsyncGeneratorFunctionDecoration::test_ok`

#### Scenario RC-S42: Async generator non-matching exception

- GIVEN an async generator function decorated with `@stamina.retry(on=ValueError)`
- WHEN the generator raises `TypeError`
- THEN `TypeError` MUST propagate immediately without retry

**Tests**: `tests/test_async.py::TestAsyncGeneratorFunctionDecoration::test_wrong_exception`

---

### Requirement RC-18: retry_context signature

The `retry_context` function MUST accept the same parameters as `retry` (with the same defaults) and MUST return a `_RetryContextIterator` that can be used with both `for` (sync) and `async for` (async) loops.

**Implementation**: `src/stamina/_core.py:119-149`

#### Scenario RC-S13: Sync retry_context block

- GIVEN a `for attempt in stamina.retry_context(on=ValueError)` loop
- WHEN the code block raises `ValueError` on the first iteration
- THEN the block MUST be re-entered on the next iteration, and `attempt.num` MUST increment

**Tests**: `tests/test_sync.py::test_retry_block`

#### Scenario RC-S14: Async retry_context block

- GIVEN an `async for attempt in stamina.retry_context(on=ValueError)` loop
- WHEN the code block raises `ValueError` on the first iteration
- THEN the block MUST be re-entered, and `attempt.num` MUST increment

**Tests**: `tests/test_async.py::test_retry_block`

#### Scenario RC-S15: Backoff hook with retry_context (sync)

- GIVEN a backoff hook `lambda exc: 0.0` passed to `retry_context`
- WHEN the code block raises an exception
- THEN the custom backoff of 0.0 MUST be used

**Tests**: `tests/test_sync.py::TestBackoffHookSync::test_backoff_hook_with_retry_context`

#### Scenario RC-S16: Backoff hook with retry_context (async)

- GIVEN a backoff hook `lambda exc: 0.0` passed to async `retry_context`
- WHEN the code block raises an exception
- THEN the custom backoff of 0.0 MUST be used

**Tests**: `tests/test_async.py::TestBackoffHookAsync::test_backoff_hook_with_async_retry_context`

---

### Requirement RC-19: retry_context yields Attempt objects

The `retry_context` iterator MUST yield `Attempt` objects that serve as context managers. Each `Attempt` MUST be entered via `with attempt:` to execute the retryable code block.

**Implementation**: `src/stamina/_core.py:570-586`, `src/stamina/_core.py:152-216`

---

### Requirement RC-20: Attempt.num property

Each `Attempt` object MUST expose a `num` property returning the 1-based attempt number.

**Implementation**: `src/stamina/_core.py:176-181`

---

### Requirement RC-21: Attempt.next_wait property

Each `Attempt` object MUST expose a `next_wait` property returning the estimated backoff in seconds before the next attempt if this attempt fails. This value MUST NOT include jitter (it is a lower bound). When the `Attempt` has no next_wait function, it MUST return `0.0`.

**Implementation**: `src/stamina/_core.py:184-203`

#### Scenario RC-S17: Attempt.next_wait reflects backoff

- GIVEN an `Attempt` object from `retry_context`
- WHEN `attempt.next_wait` is accessed
- THEN it MUST return the jitter-less lower bound of the backoff for the *next* attempt

**Tests**: `tests/test_sync.py::test_next_wait`, `tests/test_async.py::test_next_wait`

#### Scenario RC-S18: Attempt.next_wait based on attempt number

- GIVEN an `Attempt` with a next_wait_fn
- WHEN `attempt.next_wait` is called for attempt numbers 1, 2, ...
- THEN the result MUST be based on the current attempt number

**Tests**: `tests/test_sync.py::test_attempt_next_wait`

---

### Requirement RC-22: Attempt repr

The `Attempt` object MUST have a repr of the form `<Attempt num=N, next_wait=W>`.

**Implementation**: `src/stamina/_core.py:173-174`

#### Scenario RC-S20: Attempt repr format

- GIVEN an `Attempt` from a retry_context
- WHEN `repr(attempt)` is called
- THEN it MUST return `<Attempt num=N, next_wait=W>`

**Tests**: `tests/test_sync.py::test_retry_block`

---

### Requirement RC-23: callable name guessing

The retry decorator MUST resolve the callable's name using `guess_name()`, which constructs `"{module}.{qualname}"`. For builtins, only the qualname is used.

**Implementation**: `src/stamina/instrumentation/_data.py:13-20`

See also: [Spec 004: Instrumentation](../004-instrumentation/spec.md) (IN-7)

---

### Requirement RC-24: retry_context name for context blocks

When `retry_context` is used directly (not via the decorator), the callable name MUST be set to `"<context block>"`.

**Implementation**: `src/stamina/_core.py:146`

See also: [Spec 004: Instrumentation](../004-instrumentation/spec.md) (IN-S20, IN-S21)

---

### Requirement RC-25: inactive mode bypass

When stamina is deactivated (`set_active(False)`), both the `retry` decorator and `retry_context` iterator MUST execute the wrapped code exactly once without retrying, regardless of exceptions.

**Implementation**: `src/stamina/_core.py:571-575`, `src/stamina/_core.py:589`

See also: [Spec 003: Configuration](../003-configuration/spec.md) (CF-4, CF-5, CF-6)

---

### Requirement RC-26: testing mode backoff

When testing mode is active, all backoff computations MUST return `0.0` (no waiting).

**Implementation**: `src/stamina/_core.py:656-657`, `src/stamina/_core.py:631-633`

#### Scenario RC-S10: Custom backoff in test mode

- GIVEN testing mode is active and the `on` parameter is `lambda exc: 10.0`
- WHEN the function raises an exception
- THEN the custom backoff MUST be overridden to `0.0` (no waiting in test mode)

**Tests**: `tests/test_sync.py::TestBackoffHookSync::test_backoff_hook_in_test_mode_returns_zero`

---

### Requirement RC-27: keyword-only arguments

The `retry` function MUST use keyword-only arguments (via `*` in the signature). The `on` parameter has no default, forcing explicit specification. This prevents positional argument errors where exception types could be confused with numeric parameters.

**Implementation**: `src/stamina/_core.py:727` (`def retry(*, on, ...)`)
**ADR**: ADR-0007

---

### Requirement RC-28: async sleep library detection

The async retry path MUST detect the running async library and dispatch to the correct sleep function. When asyncio is detected, it MUST use `asyncio.sleep`. When Trio is detected, it MUST use `trio.sleep`. Detection SHOULD use sniffio when available, falling back to asyncio if sniffio is not installed.

**Implementation**: `src/stamina/_core.py:36-57`
**ADR**: ADR-0001

#### Scenario RC-S34: Async retry_context disabled

- GIVEN stamina is deactivated
- WHEN an async `retry_context` block raises
- THEN exactly one attempt MUST be made and the exception MUST propagate

**Tests**: `tests/test_async.py::test_retry_blocks_can_be_disabled`

#### Scenario RC-S36: Async backoff hook returns float

- GIVEN an async function with `on=lambda exc: 0.0`
- WHEN the function raises an exception
- THEN the custom backoff of 0.0 MUST be used (overriding default exponential backoff)

**Tests**: `tests/test_async.py::TestBackoffHookAsync::test_backoff_hook_returns_float_async`

---

### Requirement RC-29: version metadata

The `stamina.__version__` attribute MUST return the installed package version via `importlib.metadata.version("stamina")`. Accessing it MUST NOT emit deprecation warnings.

**Implementation**: `src/stamina/__init__.py:34-41`

#### Scenario RC-S35: Version metadata accessible

- GIVEN stamina is installed
- WHEN `stamina.__version__` is accessed
- THEN it MUST return the correct version string without warnings

**Tests**: `tests/test_packaging.py::test_version`

---

### Requirement RC-30: public type re-exports

The `stamina.typing` module MUST re-export `RetryDetails` and `RetryHook` for public use. These MUST be identical to the types in `stamina.instrumentation`.

**Implementation**: `src/stamina/typing.py:7-10`

#### Scenario RC-S43: Type re-exports are identical

- GIVEN `stamina.typing` and `stamina.instrumentation` are imported
- WHEN `RetryDetails` and `RetryHook` are accessed from both modules
- THEN they MUST be the same objects (`stamina.typing.RetryDetails is stamina.instrumentation.RetryDetails`)

**Tests**: `tests/typing/api.py` (static type verification via mypy)

---

### Requirement RC-31: static type safety

The public API MUST be statically type-checkable. A `py.typed` marker file MUST be present for PEP 561 compliance. Type stubs (`tests/typing/api.py`) MUST verify that all public API signatures are correctly typed, including: `retry` decorator on sync/async/generator/async generator functions, `retry_context` with `Attempt.num` (int) and `Attempt.next_wait` (float), `RetryingCaller`/`AsyncRetryingCaller` with typed argument passthrough, `set_active`/`is_active`/`set_testing`/`is_testing`, and `set_on_retry_hooks`/`get_on_retry_hooks`.

**Implementation**: `src/stamina/py.typed`, `tests/typing/api.py`

---

## Related Specifications

- [Spec 003: Configuration](../003-configuration/spec.md) -- RC-25, RC-26 depend on configuration state
- [Spec 004: Instrumentation](../004-instrumentation/spec.md) -- RC-23 uses `guess_name()` from instrumentation

---

## Metadata

### Implementation Files

| File | Lines | Description |
|------|-------|-------------|
| `src/stamina/_core.py` | 911 | All retry core logic |
| `src/stamina/__init__.py` | 42 | Public API exports, version metadata |
| `src/stamina/typing.py` | 10 | Public type re-exports |
| `src/stamina/instrumentation/_data.py` | 103 | `guess_name()`, `RetryDetails`, `RetryHook` protocol |

### Test Coverage

| Test File | Relevant Tests |
|-----------|----------------|
| `tests/test_sync.py` | `test_ok`, `test_retries`, `test_wrong_exception`, `test_retry_block`, `test_next_wait`, `test_backoff_computation_clamps`, `test_testing_mode`, `test_timeout_zero_warns`, `test_attempt_next_wait`, `TestMakeStop`, `TestBackoffHookSync`, `TestGeneratorFunctionDecoration` |
| `tests/test_async.py` | `test_ok`, `test_retries`, `test_retries_method`, `test_wrong_exception`, `test_retry_block`, `test_next_wait`, `test_testing_mode`, `test_retry_blocks_can_be_disabled`, `TestBackoffHookAsync`, `TestAsyncGeneratorFunctionDecoration` |
| `tests/test_packaging.py` | `test_version` |
| `tests/typing/api.py` | Static type-checking stubs (not executed, verified by mypy) |
| `tests/conftest.py` | `_on` fixture (parametrizes exception matching strategies) |
