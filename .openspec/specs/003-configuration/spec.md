---
Domain: Configuration
Version: "1.1"
Status: Implemented
Date: 2026-02-22
Spec-ID: "003"
---

# Spec 003: Configuration

## Overview

The configuration system provides global state management for stamina: an active/inactive toggle, a testing mode with configurable attempts and cap semantics, and thread-safe state mutations via `threading.Lock`. All configuration functions are idempotent and can be called repeatedly with the same values.

### Key Capabilities

- Global activate/deactivate toggle (`set_active`/`is_active`)
- Testing mode with zero-backoff and configurable attempts (`set_testing`/`is_testing`)
- Cap semantics for testing mode attempts
- Context manager support for `set_testing`
- Thread-safe state mutations

## Keywords

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", and "MAY" in this document are to be interpreted as described in RFC 2119.

---

## Requirements

### Requirement CF-1: set_active function

`set_active(active: bool)` MUST set the global active state. When `False`, all retry mechanisms MUST execute code exactly once without retrying.

**Implementation**: `src/stamina/_config.py:140-146`

#### Scenario CF-S1: Default active state

- GIVEN a fresh stamina configuration
- WHEN `is_active()` is called
- THEN it MUST return `True`

**Tests**: `tests/test_config.py::test_activate_deactivate`

#### Scenario CF-S2: Deactivate and reactivate

- GIVEN stamina is active
- WHEN `set_active(False)` then `set_active(True)` is called
- THEN `is_active()` MUST return `False` after deactivation and `True` after reactivation

**Tests**: `tests/test_config.py::test_activate_deactivate`

---

### Requirement CF-2: is_active function

`is_active()` MUST return the current active state as a boolean. The default state MUST be `True` (active).

**Implementation**: `src/stamina/_config.py:130-137`

---

### Requirement CF-3: set_active idempotency

`set_active` MUST be idempotent: calling it repeatedly with the same value MUST have no side effects.

**Implementation**: `src/stamina/_config.py:146` (simple assignment)

---

### Requirement CF-4: inactive mode -- decorator does not retry

When stamina is inactive, the `retry` decorator MUST call the wrapped function exactly once. If the function raises, the exception MUST propagate immediately.

**Implementation**: `src/stamina/_core.py:571-575` (checks `CONFIG.is_active`)

#### Scenario CF-S3: Inactive -- decorator no retry

- GIVEN stamina is deactivated
- WHEN a `@stamina.retry(on=Exception)` decorated function raises
- THEN the function MUST be called exactly once and the exception MUST propagate

**Tests**: `tests/test_sync.py::test_retry_inactive`, `tests/test_async.py::test_retry_inactive`

---

### Requirement CF-5: inactive mode -- retry_context does not retry

When stamina is inactive, `retry_context` MUST yield exactly one `Attempt`. If the code block raises, the exception MUST propagate immediately.

**Implementation**: `src/stamina/_core.py:571-575`

#### Scenario CF-S4: Inactive -- retry_context no retry

- GIVEN stamina is deactivated
- WHEN a `retry_context` block raises
- THEN the block MUST execute exactly once and the exception MUST propagate

**Tests**: `tests/test_sync.py::test_retry_inactive_block`, `tests/test_async.py::test_retry_inactive_block`

---

### Requirement CF-6: inactive mode -- happy path works

When stamina is inactive and the wrapped code succeeds, the result MUST be returned normally. Deactivation MUST NOT affect the happy path.

**Implementation**: `src/stamina/_core.py:571-575`

#### Scenario CF-S5: Inactive -- decorator happy path

- GIVEN stamina is deactivated
- WHEN a `@stamina.retry(on=Exception)` decorated function succeeds
- THEN the function MUST be called exactly once and the result returned

**Tests**: `tests/test_sync.py::test_retry_inactive_ok`, `tests/test_async.py::test_retry_inactive_ok`

#### Scenario CF-S6: Inactive -- retry_context happy path

- GIVEN stamina is deactivated
- WHEN a `retry_context` block succeeds
- THEN the block MUST execute exactly once

**Tests**: `tests/test_sync.py::test_retry_inactive_block_ok`, `tests/test_async.py::test_retry_inactive_block_ok`

#### Scenario CF-S7: Inactive -- generator no retry

- GIVEN stamina is deactivated
- WHEN a `@stamina.retry` decorated generator raises
- THEN the generator MUST be called exactly once

**Tests**: `tests/test_sync.py::TestGeneratorFunctionDecoration::test_inactive`

#### Scenario CF-S8: Inactive -- generator happy path

- GIVEN stamina is deactivated
- WHEN a `@stamina.retry` decorated generator succeeds
- THEN the generator MUST be called exactly once and yield normally

**Tests**: `tests/test_sync.py::TestGeneratorFunctionDecoration::test_retry_inactive_ok`

#### Scenario CF-S9: Inactive -- async generator no retry

- GIVEN stamina is deactivated
- WHEN a `@stamina.retry` decorated async generator raises
- THEN the async generator MUST be called exactly once

**Tests**: `tests/test_async.py::TestAsyncGeneratorFunctionDecoration::test_retry_inactive`

#### Scenario CF-S10: Inactive -- async generator happy path

- GIVEN stamina is deactivated
- WHEN a `@stamina.retry` decorated async generator succeeds
- THEN the async generator MUST be called exactly once and yield normally

**Tests**: `tests/test_async.py::TestAsyncGeneratorFunctionDecoration::test_retry_inactive_ok`

---

### Requirement CF-7: set_testing function

`set_testing(testing: bool, *, attempts: int = 1, cap: bool = False)` MUST enable or disable testing mode. When `testing=True`, a `_Testing` object MUST be created with the given `attempts` and `cap`. When `testing=False`, testing MUST be disabled (set to `None`).

**Implementation**: `src/stamina/_config.py:174-195`
**ADR**: ADR-0006

#### Scenario CF-S11: Testing mode -- set and reset

- GIVEN testing mode is not active
- WHEN `set_testing(True, attempts=3)` is called, then `set_testing(False)`
- THEN `is_testing()` MUST return `True` after setting and `False` after resetting, and retries MUST be limited to 3 attempts with zero backoff

**Tests**: `tests/test_sync.py::test_testing_mode`, `tests/test_async.py::test_testing_mode`

---

### Requirement CF-8: is_testing function

`is_testing()` MUST return `True` when testing mode is enabled, `False` otherwise. The default state MUST be `False` (not testing).

**Implementation**: `src/stamina/_config.py:149-155`

---

### Requirement CF-9: testing mode -- zero backoff

When testing mode is active, all backoff computations MUST return `0.0` regardless of configured wait parameters.

**Implementation**: `src/stamina/_core.py:656-657`

---

### Requirement CF-10: testing mode -- attempts override

When testing mode is active with `cap=False`, the `attempts` parameter from `set_testing` MUST replace the user-configured attempts value.

**Implementation**: `src/stamina/_config.py:45`
**ADR**: ADR-0006

#### Scenario CF-S14: Testing mode -- cap=False ignores user attempts

- GIVEN `_Testing(2, cap=False)`
- WHEN `get_attempts(1)` or `get_attempts(3)` is called
- THEN it MUST always return `2` (the testing value)

**Tests**: `tests/test_config.py::TestTesting::test_cap_false`

---

### Requirement CF-11: testing mode -- cap semantics

When testing mode is active with `cap=True`, the effective attempts MUST be `min(testing_attempts, user_attempts)`. If the user's attempts is `None` (unlimited), the testing value MUST be used.

**Implementation**: `src/stamina/_config.py:42-43`
**ADR**: ADR-0006

#### Scenario CF-S12: Testing mode -- cap=True with lower user attempts

- GIVEN `_Testing(2, cap=True)`
- WHEN `get_attempts(1)` is called (user specified 1)
- THEN it MUST return `1` (the lower value)

**Tests**: `tests/test_config.py::TestTesting::test_cap_true`

#### Scenario CF-S13: Testing mode -- cap=True with higher user attempts

- GIVEN `_Testing(2, cap=True)`
- WHEN `get_attempts(3)` is called (user specified 3)
- THEN it MUST return `2` (the testing cap)

**Tests**: `tests/test_config.py::TestTesting::test_cap_true`

#### Scenario CF-S15: Testing mode -- cap=True with None user attempts

- GIVEN `_Testing(100, cap=True)`
- WHEN `get_attempts(None)` is called (user specified unlimited)
- THEN it MUST return `100` (the testing value, since `None` means unlimited)

**Tests**: `tests/test_config.py::TestTesting::test_cap_true_with_none`

---

### Requirement CF-12: testing mode -- test mode applied to tenacity

When testing mode is active, the `_RetryContextIterator` MUST override the tenacity stop condition to `stop_after_attempt(testing.get_attempts(self._attempts))`.

**Implementation**: `src/stamina/_core.py:552-564`

---

### Requirement CF-13: set_testing as context manager

`set_testing` MUST return a context manager. On `__exit__`, the previous testing state MUST be restored, regardless of whether an exception occurred.

**Implementation**: `src/stamina/_config.py:158-171`, `src/stamina/_config.py:192-195`

#### Scenario CF-S16: Testing mode -- context manager basic

- GIVEN testing mode is not active
- WHEN `with set_testing(True, attempts=3):` is entered and exited
- THEN `is_testing()` MUST be `True` inside and `False` after

**Tests**: `tests/test_config.py::TestTesting::test_context_manager`

#### Scenario CF-S17: Testing mode -- context manager sync retry

- GIVEN a sync `retry_context` inside `with set_testing(True, attempts=3):`
- WHEN the block always raises `ValueError`
- THEN exactly 3 attempts MUST be made with zero backoff, and testing state MUST be restored on exit

**Tests**: `tests/test_sync.py::test_testing_mode_context`

#### Scenario CF-S18: Testing mode -- context manager async retry

- GIVEN an async `retry_context` inside `with set_testing(True, attempts=3):`
- WHEN the block always raises `ValueError`
- THEN exactly 3 attempts MUST be made with zero backoff, and testing state MUST be restored on exit

**Tests**: `tests/test_async.py::test_testing_mode_context`

#### Scenario CF-S20: Testing mode -- context manager exception safety

- GIVEN a `set_testing(True)` context manager
- WHEN an exception is raised inside the context
- THEN testing state MUST be restored to the previous state on exit

**Tests**: `tests/test_config.py::TestTesting::test_context_manager_exception`

---

### Requirement CF-14: set_testing context manager nesting

`set_testing` context managers MUST support nesting. Each level MUST restore the state from before its corresponding `set_testing` call.

**Implementation**: `src/stamina/_config.py:158-171`

#### Scenario CF-S19: Testing mode -- nested context managers

- GIVEN nested `set_testing` context managers
- WHEN the inner context sets `attempts=5, cap=True` and the outer sets `attempts=3`
- THEN each level MUST have its own settings and MUST restore the previous level on exit

**Tests**: `tests/test_config.py::TestTesting::test_context_manager_nested`

---

### Requirement CF-15: thread-safe is_active mutation

Setting `is_active` MUST acquire the `_Config.lock` before mutating state.

**Implementation**: `src/stamina/_config.py:84-87`
**ADR**: ADR-0002

#### Scenario CF-S23: Thread-safe active state mutation

- GIVEN the `_Config` class with a `Lock`
- WHEN `is_active.setter` is invoked
- THEN the lock MUST be acquired before setting `_is_active`

**Tests**: `tests/test_config.py::test_config_init_concurrently` (validates lock-based initialization)

---

### Requirement CF-16: thread-safe testing mutation

Setting `testing` MUST acquire the `_Config.lock` before mutating state.

**Implementation**: `src/stamina/_config.py:93-96`
**ADR**: ADR-0002

---

### Requirement CF-17: thread-safe on_retry mutation

Setting `on_retry` MUST acquire the `_Config.lock` before mutating state.

**Implementation**: `src/stamina/_config.py:102-108`
**ADR**: ADR-0002

---

### Requirement CF-18: public API exports

`set_active`, `is_active`, `set_testing`, and `is_testing` MUST be exported from the top-level `stamina` module.

**Implementation**: `src/stamina/__init__.py:6`, `src/stamina/__init__.py:18-31`

---

### Requirement CF-19: conftest auto-reset fixture

The `_reset_config` autouse fixture MUST reset stamina to active state and default hooks before each test, ensuring test isolation.

**Implementation**: `tests/conftest.py:21-27`

#### Scenario CF-S22: conftest auto-reset

- GIVEN the `_reset_config` autouse fixture
- WHEN each test begins
- THEN stamina MUST be active and retry hooks MUST be reset to defaults

**Tests**: `tests/conftest.py::_reset_config`

#### Scenario CF-S21: Concurrent config initialization

- GIVEN `_Config._init_on_first_retry` is called
- WHEN another thread has already initialized hooks while waiting for the lock
- THEN the method MUST detect that initialization already occurred and not re-initialize

**Tests**: `tests/test_config.py::test_config_init_concurrently`

---

## Related Specifications

- [Spec 001: Retry Core](../001-retry-core/spec.md) -- Configuration governs retry behavior (RC-25, RC-26)

---

## Metadata

### Implementation Files

| File | Lines | Description |
|------|-------|-------------|
| `src/stamina/_config.py` | 196 | `_Config`, `_Testing`, `set_active`, `is_active`, `set_testing`, `is_testing`, `_RestoreTestingCM` |
| `src/stamina/__init__.py` | 6, 18-31 | Public exports |

### Test Coverage

| Test File | Relevant Tests |
|-----------|----------------|
| `tests/test_config.py` | `test_activate_deactivate`, `test_config_init_concurrently`, `TestTesting::test_cap_true`, `TestTesting::test_cap_false`, `TestTesting::test_cap_true_with_none`, `TestTesting::test_context_manager`, `TestTesting::test_context_manager_nested`, `TestTesting::test_context_manager_exception` |
| `tests/test_sync.py` | `test_retry_inactive`, `test_retry_inactive_block`, `test_retry_inactive_ok`, `test_retry_inactive_block_ok`, `test_testing_mode`, `test_testing_mode_context`, `TestGeneratorFunctionDecoration::test_inactive`, `TestGeneratorFunctionDecoration::test_retry_inactive_ok` |
| `tests/test_async.py` | `test_retry_inactive`, `test_retry_inactive_block`, `test_retry_inactive_ok`, `test_retry_inactive_block_ok`, `test_testing_mode`, `test_testing_mode_context`, `test_retry_blocks_can_be_disabled`, `TestAsyncGeneratorFunctionDecoration::test_retry_inactive`, `TestAsyncGeneratorFunctionDecoration::test_retry_inactive_ok` |
| `tests/conftest.py` | `_reset_config` autouse fixture |
