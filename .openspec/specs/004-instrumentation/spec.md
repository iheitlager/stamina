---
Domain: Instrumentation
Version: "1.1"
Status: Implemented
Date: 2026-02-22
Spec-ID: "004"
---

# Spec 004: Instrumentation

## Overview

The instrumentation system provides observability for retry operations. It defines the `RetryHook` protocol, the `RetryDetails` dataclass, and `RetryHookFactory` for delayed initialization. Hooks are auto-detected based on installed packages (Prometheus, structlog, stdlib logging) and are called on every scheduled retry. Hooks MAY return context managers that are entered before the retry wait and exited before the next attempt.

### Key Capabilities

- `RetryHook` protocol for custom retry observers
- `RetryDetails` frozen dataclass with retry metadata
- `RetryHookFactory` for lazy initialization of hooks
- Auto-detection chain: Prometheus > structlog > stdlib logging
- Hook management via `set_on_retry_hooks`/`get_on_retry_hooks`
- Context manager hooks for scoped retry instrumentation
- Built-in Prometheus counter, structlog logger, and stdlib logger backends
- `guess_name()` utility for callable name resolution

## Keywords

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", and "MAY" in this document are to be interpreted as described in RFC 2119.

---

## Requirements

### Requirement IN-1: RetryHook protocol

`RetryHook` MUST be a `typing.Protocol` with a `__call__` method accepting a single `RetryDetails` argument and returning `None | AbstractContextManager[None]`.

**Implementation**: `src/stamina/instrumentation/_data.py:69-88`

---

### Requirement IN-2: RetryDetails dataclass

`RetryDetails` MUST be a frozen dataclass with `__slots__` containing: `name` (str), `args` (tuple), `kwargs` (dict), `retry_num` (int), `wait_for` (float), `waited_so_far` (float), `caused_by` (Exception).

**Implementation**: `src/stamina/instrumentation/_data.py:23-66`

#### Scenario IN-S22: RetryDetails populated correctly

- GIVEN a context manager hook that captures `RetryDetails`
- WHEN a retry is scheduled
- THEN `RetryDetails` MUST contain: the callable's qualified name, empty args/kwargs, `retry_num=1`, `wait_for=0.0` (with wait_max=0), `waited_so_far=0.0`, and the `caused_by` exception

**Tests**: `tests/test_instrumentation.py::TestSetOnRetryHooks::test_context_manager_hooks`

---

### Requirement IN-3: RetryDetails.retry_num starts at 1

`retry_num` MUST start at 1 after the first failure (it represents the retry attempt number, not the total call count).

**Implementation**: `src/stamina/instrumentation/_data.py:63` (field declaration), `src/stamina/instrumentation/_data.py:35-36` (documentation)

---

### Requirement IN-4: RetryDetails.wait_for is per-retry

`wait_for` MUST represent the time in seconds that stamina will wait before the next attempt (the current backoff, not cumulative).

**Implementation**: `src/stamina/_core.py:685`

---

### Requirement IN-5: RetryDetails.waited_so_far is cumulative

`waited_so_far` MUST represent the total time in seconds that stamina has waited so far for the current callable (cumulative, not including the current wait).

**Implementation**: `src/stamina/_core.py:691`

---

### Requirement IN-6: RetryHookFactory

`RetryHookFactory` MUST be a frozen dataclass wrapping a `hook_factory: Callable[[], RetryHook]`. The factory MUST be called on the first scheduled retry to produce the actual hook.

**Implementation**: `src/stamina/instrumentation/_data.py:91-103`
**ADR**: ADR-0005

---

### Requirement IN-7: guess_name utility

`guess_name(obj)` MUST return `"{module}.{qualname}"` for callables. For builtins (module == "builtins"), it MUST return just the qualname. If `__qualname__` is missing, it MUST use `"<unnamed object>"`. If `__module__` is missing, it MUST use `"<unknown module>"`.

**Implementation**: `src/stamina/instrumentation/_data.py:13-20`

#### Scenario IN-S1: guess_name for module-level functions

- GIVEN a function defined at module level
- WHEN `guess_name(func)` is called
- THEN it MUST return `"{module}.{qualname}"` (e.g., `"tests.test_instrumentation.function"`)

**Tests**: `tests/test_instrumentation.py::TestGuessName::test_module_scope`

#### Scenario IN-S2: guess_name for methods

- GIVEN a bound or unbound method
- WHEN `guess_name(method)` is called
- THEN it MUST return `"{module}.{class}.{method}"` (e.g., `"tests.test_instrumentation.Foo.method"`)

**Tests**: `tests/test_instrumentation.py::TestGuessName::test_module_scope`

#### Scenario IN-S3: guess_name for local functions

- GIVEN a function defined inside another function or method
- WHEN `guess_name(func)` is called
- THEN it MUST include the full `<locals>` qualname path

**Tests**: `tests/test_instrumentation.py::TestGuessName::test_local`

---

### Requirement IN-8: Auto-detection chain

The default hook detection (`get_default_hooks`) MUST follow this chain:
1. If `prometheus_client` is importable, add `PrometheusOnRetryHook`
2. If `structlog` is importable, add `StructlogOnRetryHook`; otherwise add `LoggingOnRetryHook`

All detected hooks MUST be wrapped as `RetryHookFactory` instances for lazy initialization.

**Implementation**: `src/stamina/instrumentation/_hooks.py:31-51`
**ADR**: ADR-0003

#### Scenario IN-S4: Default hooks auto-detected

- GIVEN both `prometheus_client` and `structlog` are installed
- WHEN `get_default_hooks()` is called
- THEN it MUST return 2 hooks (Prometheus + structlog)
- AND GIVEN only `structlog` is installed (no Prometheus)
- WHEN `get_default_hooks()` is called
- THEN it MUST return 1 hook (structlog)
- AND GIVEN neither `structlog` nor `prometheus_client` is installed
- WHEN `get_default_hooks()` is called
- THEN it MUST return 1 hook (stdlib logging)

**Tests**: `tests/test_instrumentation.py::test_get_default_hooks`

#### Scenario IN-S5: structlog detected when available

- GIVEN `structlog` is importable
- WHEN `init_structlog()` is called
- THEN it MUST return a callable (the log_retries hook)

**Tests**: `tests/test_instrumentation.py::test_structlog_detected`

---

### Requirement IN-9: set_on_retry_hooks

`set_on_retry_hooks(hooks)` MUST set the hooks called after a retry is scheduled. Passing `None` MUST reset to the default auto-detected hooks. Passing an empty iterable MUST deactivate instrumentation entirely.

**Implementation**: `src/stamina/instrumentation/_hooks.py:54-69`

#### Scenario IN-S7: set_on_retry_hooks with None resets to defaults

- GIVEN hooks have been customized
- WHEN `set_on_retry_hooks(None)` is called
- THEN `get_on_retry_hooks()` MUST return the auto-detected default hooks

**Tests**: `tests/test_instrumentation.py::TestSetOnRetryHooks::test_none_is_default`

#### Scenario IN-S8: set_on_retry_hooks with empty deactivates

- GIVEN any hook state
- WHEN `set_on_retry_hooks(())` is called
- THEN `get_on_retry_hooks()` MUST return an empty tuple

**Tests**: `tests/test_instrumentation.py::TestSetOnRetryHooks::test_none_is_default`

---

### Requirement IN-10: get_on_retry_hooks

`get_on_retry_hooks()` MUST return the current tuple of finalized `RetryHook` instances. If hooks haven't been initialized yet, calling this MUST trigger initialization (including resolving factories).

**Implementation**: `src/stamina/instrumentation/_hooks.py:72-84`

---

### Requirement IN-11: init_hooks factory resolution

`init_hooks` MUST iterate over the hooks tuple and call `.hook_factory()` on any `RetryHookFactory` instances, replacing them with the resulting `RetryHook`. Non-factory hooks MUST be passed through unchanged.

**Implementation**: `src/stamina/instrumentation/_hooks.py:15-28`

#### Scenario IN-S9: RetryHookFactory initialization

- GIVEN a `RetryHookFactory` wrapping a factory function
- WHEN `set_on_retry_hooks([hook, RetryHookFactory(init)])` is called and hooks are accessed
- THEN the factory MUST be called to produce the actual hook, and direct hooks MUST be passed through unchanged

**Tests**: `tests/test_instrumentation.py::TestSetOnRetryHooks::test_init_hooks`

---

### Requirement IN-12: Delayed initialization

Hook initialization MUST be delayed until the first scheduled retry. The `_Config._init_on_first_retry` method MUST be called on first access of `on_retry` and MUST be thread-safe (acquiring the lock and checking for concurrent initialization).

**Implementation**: `src/stamina/_config.py:110-124`
**ADR**: ADR-0005

---

### Requirement IN-13: Context manager hooks

When a `RetryHook` returns a context manager (an instance of `AbstractContextManager`), the context manager MUST be `__enter__`'d when the retry is scheduled and MUST be `__exit__`'d before the next retry attempt begins.

**Implementation**: `src/stamina/_core.py:697-702`, `src/stamina/_core.py:566-568`

#### Scenario IN-S10: Context manager hook -- sync function

- GIVEN hooks that return context managers are registered
- WHEN a sync `@stamina.retry` decorated function retries
- THEN the context manager MUST be entered when retry is scheduled and exited before the next attempt, and `RetryDetails` MUST be passed correctly

**Tests**: `tests/test_instrumentation.py::TestSetOnRetryHooks::test_context_manager_hooks`

#### Scenario IN-S11: Context manager hook -- async function

- GIVEN hooks that return context managers are registered
- WHEN an async `@stamina.retry` decorated function retries
- THEN the context manager MUST be entered and exited correctly

**Tests**: `tests/test_instrumentation.py::TestSetOnRetryHooks::test_context_manager_hooks_async`

#### Scenario IN-S12: Context manager hook -- sync generator

- GIVEN hooks that return context managers are registered
- WHEN a sync generator `@stamina.retry` decorated function retries
- THEN the context manager MUST be entered and exited correctly

**Tests**: `tests/test_instrumentation.py::TestSetOnRetryHooks::test_context_manager_hooks_with_sync_generator_function`

#### Scenario IN-S13: Context manager hook -- async generator

- GIVEN hooks that return context managers are registered
- WHEN an async generator `@stamina.retry` decorated function retries
- THEN the context manager MUST be entered and exited correctly

**Tests**: `tests/test_instrumentation.py::TestSetOnRetryHooks::test_context_manager_hooks_with_async_generator_function`

---

### Requirement IN-14: Prometheus backend

The Prometheus backend MUST create a `Counter` named `"stamina_retries_total"` with labels `("callable", "retry_num", "error_type")`. On each retry, it MUST increment the counter with the callable name, retry number, and error type name.

**Implementation**: `src/stamina/instrumentation/_prometheus.py:19-45`

#### Scenario IN-S6: Prometheus counter available when installed

- GIVEN `prometheus_client` is installed
- WHEN `get_prometheus_counter()` is called
- THEN it MUST return a non-None `Counter` object
- AND GIVEN `prometheus_client` is not installed
- WHEN `get_prometheus_counter()` is called
- THEN it MUST return `None`

**Tests**: `tests/test_instrumentation.py::test_get_prometheus_counter`

---

### Requirement IN-15: Prometheus counter singleton

The Prometheus counter MUST be created at most once (singleton pattern). Subsequent calls to `init_prometheus` MUST reuse the existing counter.

**Implementation**: `src/stamina/instrumentation/_prometheus.py:28-33`

#### Scenario IN-S23: Prometheus counter singleton behavior

- GIVEN `init_prometheus()` has been called once
- WHEN `init_prometheus()` is called again
- THEN the global `RETRIES_TOTAL` counter MUST NOT be recreated (the `if RETRIES_TOTAL is None` guard prevents it)

**Tests**: `tests/test_instrumentation.py::test_get_prometheus_counter` (calls `get_on_retry_hooks()` which triggers finalization, then verifies counter identity)

---

### Requirement IN-16: get_prometheus_counter

`get_prometheus_counter()` MUST return the Prometheus `Counter` if active, or `None` if Prometheus instrumentation is not initialized. Calling it MUST trigger hook finalization if not already done.

**Implementation**: `src/stamina/instrumentation/_prometheus.py:48-64`

---

### Requirement IN-17: structlog backend

The structlog backend MUST log at `warning` level to a logger named `"stamina"`. The log event MUST be `"stamina.retry_scheduled"` with keyword arguments: `callable`, `args`, `kwargs`, `retry_num`, `caused_by`, `wait_for` (rounded to 2 decimals), `waited_so_far` (rounded to 2 decimals).

**Implementation**: `src/stamina/instrumentation/_structlog.py:10-33`

#### Scenario IN-S18: structlog -- sync decorator logging

- GIVEN structlog is installed and configured with `LogCapture`
- WHEN a sync `@stamina.retry` decorated function retries
- THEN a warning log entry MUST be emitted with event `"stamina.retry_scheduled"`, the callable's qualified name, args, kwargs, retry_num, caused_by, wait_for, and waited_so_far

**Tests**: `tests/test_structlog.py::test_decorator_sync`

#### Scenario IN-S19: structlog -- async decorator logging

- GIVEN structlog is installed and configured with `LogCapture`
- WHEN an async `@stamina.retry` decorated function retries
- THEN a warning log entry MUST be emitted with the same fields

**Tests**: `tests/test_structlog.py::test_decorator_async`

#### Scenario IN-S20: structlog -- sync context block logging

- GIVEN structlog is installed and configured
- WHEN a sync `retry_context` block retries
- THEN a warning log entry MUST be emitted with callable `"<context block>"`

**Tests**: `tests/test_structlog.py::test_context_sync`

#### Scenario IN-S21: structlog -- async context block logging

- GIVEN structlog is installed and configured
- WHEN an async `retry_context` block retries
- THEN a warning log entry MUST be emitted with callable `"<context block>"`

**Tests**: `tests/test_structlog.py::test_context_async`

---

### Requirement IN-18: stdlib logging backend

The stdlib logging backend MUST log at a configurable level (default `WARNING`/30) to a logger named `"stamina"`. The log message MUST be `"stamina.retry_scheduled"` with `extra` dict containing: `stamina.callable`, `stamina.args`, `stamina.kwargs`, `stamina.retry_num`, `stamina.caused_by`, `stamina.wait_for` (rounded to 2 decimals), `stamina.waited_so_far` (rounded to 2 decimals).

**Implementation**: `src/stamina/instrumentation/_logging.py:10-37`

#### Scenario IN-S14: stdlib logging -- sync retry logged

- GIVEN structlog is not available and default hooks are active
- WHEN a sync function retries
- THEN a log record MUST be emitted at WARNING level with logger "stamina" and message "stamina.retry_scheduled"

**Tests**: `tests/test_instrumentation.py::TestLogging::test_sync`

#### Scenario IN-S15: stdlib logging -- async retry logged

- GIVEN structlog is not available and default hooks are active
- WHEN an async function retries
- THEN a log record MUST be emitted at WARNING level with logger "stamina" and message "stamina.retry_scheduled"

**Tests**: `tests/test_instrumentation.py::TestLogging::test_async`

#### Scenario IN-S16: stdlib logging -- sync generator retry logged

- GIVEN structlog is not available and default hooks are active
- WHEN a sync generator function retries
- THEN a log record MUST be emitted

**Tests**: `tests/test_instrumentation.py::TestLogging::test_sync_generator_function`

#### Scenario IN-S17: stdlib logging -- async generator retry logged

- GIVEN structlog is not available and default hooks are active
- WHEN an async generator function retries
- THEN a log record MUST be emitted

**Tests**: `tests/test_instrumentation.py::TestLogging::test_async_generator_function`

---

## Related Specifications

- [Spec 001: Retry Core](../001-retry-core/spec.md) -- Uses `guess_name()` (RC-23), context block name (RC-24)
- [Spec 003: Configuration](../003-configuration/spec.md) -- Hook state managed via `_Config.on_retry`

---

## Metadata

### Implementation Files

| File | Lines | Description |
|------|-------|-------------|
| `src/stamina/instrumentation/__init__.py` | 25 | Public API exports |
| `src/stamina/instrumentation/_data.py` | 103 | `RetryDetails`, `RetryHook`, `RetryHookFactory`, `guess_name` |
| `src/stamina/instrumentation/_hooks.py` | 85 | `init_hooks`, `get_default_hooks`, `set_on_retry_hooks`, `get_on_retry_hooks` |
| `src/stamina/instrumentation/_logging.py` | 40 | `init_logging`, `LoggingOnRetryHook` |
| `src/stamina/instrumentation/_prometheus.py` | 67 | `init_prometheus`, `get_prometheus_counter`, `PrometheusOnRetryHook` |
| `src/stamina/instrumentation/_structlog.py` | 35 | `init_structlog`, `StructlogOnRetryHook` |
| `src/stamina/_core.py` | 664-706 | `_make_before_sleep` (hook invocation), `_exit_cms` |

### Test Coverage

| Test File | Relevant Tests |
|-----------|----------------|
| `tests/test_instrumentation.py` | `TestGuessName`, `test_get_default_hooks`, `test_get_prometheus_counter`, `test_structlog_detected`, `TestLogging`, `TestSetOnRetryHooks` |
| `tests/test_structlog.py` | `test_decorator_sync`, `test_decorator_async`, `test_context_sync`, `test_context_async` |
