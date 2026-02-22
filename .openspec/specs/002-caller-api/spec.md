---
Domain: Caller API
Version: "1.0"
Status: Implemented
Date: 2026-02-22
Spec-ID: "002"
---

# Spec 002: Caller API

## Overview

The Caller API provides a convenience layer over the retry core (Spec 001) for retrying callable invocations without decorators. It offers `RetryingCaller` and `AsyncRetryingCaller` for unbound usage, plus `BoundRetryingCaller` and `BoundAsyncRetryingCaller` for pre-bound exception types via the `.on()` method.

### Key Capabilities

- Retry arbitrary callables without decorating them
- Pre-bind exception types for repeated use
- Reusable instances (each call creates a fresh retry context)
- Full argument passthrough to wrapped callables
- Useful repr for debugging

## Keywords

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", and "MAY" in this document are to be interpreted as described in RFC 2119.

---

## Requirements

### Requirement CA-1: RetryingCaller constructor

`RetryingCaller` MUST accept the same retry configuration parameters as `retry` (except `on`): `attempts`, `timeout`, `wait_initial`, `wait_max`, `wait_jitter`, `wait_exp_base`, with identical defaults.

**Implementation**: `src/stamina/_core.py:228-254` (BaseRetryingCaller)

### Requirement CA-2: RetryingCaller.__call__ signature

`RetryingCaller.__call__` MUST accept `on` and `callable_` as positional-only parameters, followed by `*args` and `**kwargs`. The `on` parameter MUST accept the same types as the `retry` decorator's `on` parameter.

**Implementation**: `src/stamina/_core.py:278-302`

### Requirement CA-3: RetryingCaller argument passthrough

When `RetryingCaller.__call__` is invoked with `(on, callable_, *args, **kwargs)`, it MUST call `callable_(*args, **kwargs)` and pass all arguments through unchanged.

**Implementation**: `src/stamina/_core.py:298-300`

### Requirement CA-4: RetryingCaller uses retry_context internally

`RetryingCaller.__call__` MUST use `retry_context` internally, creating a fresh iterator on each call. Instances MUST be reusable across multiple calls.

**Implementation**: `src/stamina/_core.py:298-302`

### Requirement CA-5: RetryingCaller.on binding

`RetryingCaller.on(on)` MUST return a `BoundRetryingCaller` pre-bound to the specified exception type or backoff hook. The `on` parameter MUST be positional-only.

**Implementation**: `src/stamina/_core.py:304-314`

### Requirement CA-6: BoundRetryingCaller.__call__

`BoundRetryingCaller.__call__` MUST accept `callable_` as a positional-only parameter, followed by `*args` and `**kwargs`. It MUST delegate to its parent `RetryingCaller.__call__` with the pre-bound `on` value.

**Implementation**: `src/stamina/_core.py:346-353`

### Requirement CA-7: AsyncRetryingCaller

`AsyncRetryingCaller` MUST behave identically to `RetryingCaller` except that `__call__` MUST be an async method that awaits the callable. It MUST use `async for` with `retry_context` internally.

**Implementation**: `src/stamina/_core.py:356-387`

### Requirement CA-8: BoundAsyncRetryingCaller

`BoundAsyncRetryingCaller` MUST behave identically to `BoundRetryingCaller` except that `__call__` MUST be an async method. It MUST delegate to `AsyncRetryingCaller.__call__` with the pre-bound `on` value.

**Implementation**: `src/stamina/_core.py:390-428`

### Requirement CA-9: RetryingCaller repr

`RetryingCaller` and `AsyncRetryingCaller` repr MUST follow the format `<ClassName(key=value, ...)>` with parameters sorted alphabetically, excluding `on`.

**Implementation**: `src/stamina/_core.py:256-262`

### Requirement CA-10: BoundRetryingCaller repr

`BoundRetryingCaller` repr MUST follow the format `<BoundRetryingCaller(exception_name, <parent_repr>)>` where `exception_name` is the `guess_name()` of the bound `on` value.

**Implementation**: `src/stamina/_core.py:341-344`

### Requirement CA-11: BoundAsyncRetryingCaller repr

`BoundAsyncRetryingCaller` repr MUST follow the format `<BoundAsyncRetryingCaller(exception_name, <parent_repr>)>`.

**Implementation**: `src/stamina/_core.py:414-415`

### Requirement CA-12: Caller API public exports

The following classes MUST be exported from `stamina`: `RetryingCaller`, `AsyncRetryingCaller`, `BoundRetryingCaller`, `BoundAsyncRetryingCaller`.

**Implementation**: `src/stamina/__init__.py:7-15`

---

## Scenarios

### Scenario CA-S1: RetryingCaller happy path via .on()

**Given** a `RetryingCaller` bound to `BaseException` via `.on()`
**When** the callable returns successfully
**Then** the return value MUST be passed through

**Tests**: `tests/test_sync.py::TestRetryingCaller::test_ok`

### Scenario CA-S2: RetryingCaller retries and passes arguments

**Given** a `RetryingCaller(wait_max=0).on(ValueError)`
**When** the callable raises `ValueError` on first call, then returns `(args, kwargs)`
**Then** the caller MUST retry, and positional/keyword arguments MUST be passed through unchanged

**Tests**: `tests/test_sync.py::TestRetryingCaller::test_retries`

### Scenario CA-S3: RetryingCaller repr format

**Given** `RetryingCaller(attempts=42, timeout=13.0, wait_initial=23, wait_max=123, wait_jitter=0.42, wait_exp_base=666)`
**When** `repr()` is called
**Then** it MUST return `<RetryingCaller(attempts=42, timeout=13.0, wait_exp_base=666, wait_initial=23, wait_jitter=0.42, wait_max=123)>`

**Tests**: `tests/test_sync.py::TestRetryingCaller::test_repr`

### Scenario CA-S4: BoundRetryingCaller repr format

**Given** a `RetryingCaller` `rc` bound to `ValueError` via `.on()`
**When** `repr()` is called on the bound caller
**Then** it MUST return `<BoundRetryingCaller(ValueError, <rc_repr>)>`

**Tests**: `tests/test_sync.py::TestRetryingCaller::test_repr`

### Scenario CA-S5: AsyncRetryingCaller happy path via .on()

**Given** an `AsyncRetryingCaller` bound to `BaseException` via `.on()`
**When** the async callable returns successfully
**Then** the return value MUST be passed through

**Tests**: `tests/test_async.py::TestAsyncRetryingCaller::test_ok`

### Scenario CA-S6: AsyncRetryingCaller retries and passes arguments

**Given** an `AsyncRetryingCaller(wait_max=0).on(ValueError)`
**When** the async callable raises `ValueError` on first call, then returns `(args, kwargs)`
**Then** the caller MUST retry, and arguments MUST be passed through unchanged

**Tests**: `tests/test_async.py::TestAsyncRetryingCaller::test_retries`

### Scenario CA-S7: AsyncRetryingCaller repr format

**Given** `AsyncRetryingCaller(attempts=42, timeout=13.0, wait_initial=23, wait_max=123, wait_jitter=0.42, wait_exp_base=666)`
**When** `repr()` is called
**Then** it MUST return `<AsyncRetryingCaller(attempts=42, timeout=13.0, wait_exp_base=666, wait_initial=23, wait_jitter=0.42, wait_max=123)>`

**Tests**: `tests/test_async.py::TestAsyncRetryingCaller::test_repr`

### Scenario CA-S8: BoundAsyncRetryingCaller repr format

**Given** an `AsyncRetryingCaller` `arc` bound to `ValueError` via `.on()`
**When** `repr()` is called on the bound caller
**Then** it MUST return `<BoundAsyncRetryingCaller(ValueError, <arc_repr>)>`

**Tests**: `tests/test_async.py::TestAsyncRetryingCaller::test_repr`

---

## Metadata

### Implementation Files

| File | Lines | Description |
|------|-------|-------------|
| `src/stamina/_core.py` | 228-428 | BaseRetryingCaller, RetryingCaller, BoundRetryingCaller, AsyncRetryingCaller, BoundAsyncRetryingCaller |
| `src/stamina/__init__.py` | 7-15 | Public exports |

### Test Coverage

| Test File | Relevant Tests |
|-----------|----------------|
| `tests/test_sync.py` | `TestRetryingCaller::test_ok`, `TestRetryingCaller::test_retries`, `TestRetryingCaller::test_repr` |
| `tests/test_async.py` | `TestAsyncRetryingCaller::test_ok`, `TestAsyncRetryingCaller::test_retries`, `TestAsyncRetryingCaller::test_repr` |
