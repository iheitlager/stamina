---
Domain: Configuration
Version: "1.0"
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

### Requirement CF-2: is_active function

`is_active()` MUST return the current active state as a boolean. The default state MUST be `True` (active).

**Implementation**: `src/stamina/_config.py:130-137`

### Requirement CF-3: set_active idempotency

`set_active` MUST be idempotent: calling it repeatedly with the same value MUST have no side effects.

**Implementation**: `src/stamina/_config.py:146` (simple assignment)

### Requirement CF-4: inactive mode — decorator does not retry

When stamina is inactive, the `retry` decorator MUST call the wrapped function exactly once. If the function raises, the exception MUST propagate immediately.

**Implementation**: `src/stamina/_core.py:571-575` (checks `CONFIG.is_active`)

### Requirement CF-5: inactive mode — retry_context does not retry

When stamina is inactive, `retry_context` MUST yield exactly one `Attempt`. If the code block raises, the exception MUST propagate immediately.

**Implementation**: `src/stamina/_core.py:571-575`

### Requirement CF-6: inactive mode — happy path works

When stamina is inactive and the wrapped code succeeds, the result MUST be returned normally. Deactivation MUST NOT affect the happy path.

**Implementation**: `src/stamina/_core.py:571-575`

### Requirement CF-7: set_testing function

`set_testing(testing: bool, *, attempts: int = 1, cap: bool = False)` MUST enable or disable testing mode. When `testing=True`, a `_Testing` object MUST be created with the given `attempts` and `cap`. When `testing=False`, testing MUST be disabled (set to `None`).

**Implementation**: `src/stamina/_config.py:174-195`

### Requirement CF-8: is_testing function

`is_testing()` MUST return `True` when testing mode is enabled, `False` otherwise. The default state MUST be `False` (not testing).

**Implementation**: `src/stamina/_config.py:149-155`

### Requirement CF-9: testing mode — zero backoff

When testing mode is active, all backoff computations MUST return `0.0` regardless of configured wait parameters.

**Implementation**: `src/stamina/_core.py:656-657`

### Requirement CF-10: testing mode — attempts override

When testing mode is active with `cap=False`, the `attempts` parameter from `set_testing` MUST replace the user-configured attempts value.

**Implementation**: `src/stamina/_config.py:45`

### Requirement CF-11: testing mode — cap semantics

When testing mode is active with `cap=True`, the effective attempts MUST be `min(testing_attempts, user_attempts)`. If the user's attempts is `None` (unlimited), the testing value MUST be used.

**Implementation**: `src/stamina/_config.py:42-43`

### Requirement CF-12: testing mode — test mode applied to tenacity

When testing mode is active, the `_RetryContextIterator` MUST override the tenacity stop condition to `stop_after_attempt(testing.get_attempts(self._attempts))`.

**Implementation**: `src/stamina/_core.py:552-564`

### Requirement CF-13: set_testing as context manager

`set_testing` MUST return a context manager. On `__exit__`, the previous testing state MUST be restored, regardless of whether an exception occurred.

**Implementation**: `src/stamina/_config.py:158-171`, `src/stamina/_config.py:192-195`

### Requirement CF-14: set_testing context manager nesting

`set_testing` context managers MUST support nesting. Each level MUST restore the state from before its corresponding `set_testing` call.

**Implementation**: `src/stamina/_config.py:158-171`

### Requirement CF-15: thread-safe is_active mutation

Setting `is_active` MUST acquire the `_Config.lock` before mutating state.

**Implementation**: `src/stamina/_config.py:84-87`

### Requirement CF-16: thread-safe testing mutation

Setting `testing` MUST acquire the `_Config.lock` before mutating state.

**Implementation**: `src/stamina/_config.py:93-96`

### Requirement CF-17: thread-safe on_retry mutation

Setting `on_retry` MUST acquire the `_Config.lock` before mutating state.

**Implementation**: `src/stamina/_config.py:102-108`

### Requirement CF-18: public API exports

`set_active`, `is_active`, `set_testing`, and `is_testing` MUST be exported from the top-level `stamina` module.

**Implementation**: `src/stamina/__init__.py:6`, `src/stamina/__init__.py:18-31`

---

## Scenarios

### Scenario CF-S1: Default active state

**Given** a fresh stamina configuration
**When** `is_active()` is called
**Then** it MUST return `True`

**Tests**: `tests/test_config.py::test_activate_deactivate`

### Scenario CF-S2: Deactivate and reactivate

**Given** stamina is active
**When** `set_active(False)` then `set_active(True)` is called
**Then** `is_active()` MUST return `False` after deactivation and `True` after reactivation

**Tests**: `tests/test_config.py::test_activate_deactivate`

### Scenario CF-S3: Inactive — decorator no retry

**Given** stamina is deactivated
**When** a `@stamina.retry(on=Exception)` decorated function raises
**Then** the function MUST be called exactly once and the exception MUST propagate

**Tests**: `tests/test_sync.py::test_retry_inactive`, `tests/test_async.py::test_retry_inactive`

### Scenario CF-S4: Inactive — retry_context no retry

**Given** stamina is deactivated
**When** a `retry_context` block raises
**Then** the block MUST execute exactly once and the exception MUST propagate

**Tests**: `tests/test_sync.py::test_retry_inactive_block`, `tests/test_async.py::test_retry_inactive_block`

### Scenario CF-S5: Inactive — decorator happy path

**Given** stamina is deactivated
**When** a `@stamina.retry(on=Exception)` decorated function succeeds
**Then** the function MUST be called exactly once and the result returned

**Tests**: `tests/test_sync.py::test_retry_inactive_ok`, `tests/test_async.py::test_retry_inactive_ok`

### Scenario CF-S6: Inactive — retry_context happy path

**Given** stamina is deactivated
**When** a `retry_context` block succeeds
**Then** the block MUST execute exactly once

**Tests**: `tests/test_sync.py::test_retry_inactive_block_ok`, `tests/test_async.py::test_retry_inactive_block_ok`

### Scenario CF-S7: Inactive — generator no retry

**Given** stamina is deactivated
**When** a `@stamina.retry` decorated generator raises
**Then** the generator MUST be called exactly once

**Tests**: `tests/test_sync.py::TestGeneratorFunctionDecoration::test_inactive`

### Scenario CF-S8: Inactive — generator happy path

**Given** stamina is deactivated
**When** a `@stamina.retry` decorated generator succeeds
**Then** the generator MUST be called exactly once and yield normally

**Tests**: `tests/test_sync.py::TestGeneratorFunctionDecoration::test_retry_inactive_ok`

### Scenario CF-S9: Inactive — async generator no retry

**Given** stamina is deactivated
**When** a `@stamina.retry` decorated async generator raises
**Then** the async generator MUST be called exactly once

**Tests**: `tests/test_async.py::TestAsyncGeneratorFunctionDecoration::test_retry_inactive`

### Scenario CF-S10: Inactive — async generator happy path

**Given** stamina is deactivated
**When** a `@stamina.retry` decorated async generator succeeds
**Then** the async generator MUST be called exactly once and yield normally

**Tests**: `tests/test_async.py::TestAsyncGeneratorFunctionDecoration::test_retry_inactive_ok`

### Scenario CF-S11: Testing mode — set and reset

**Given** testing mode is not active
**When** `set_testing(True, attempts=3)` is called, then `set_testing(False)`
**Then** `is_testing()` MUST return `True` after setting and `False` after resetting, and retries MUST be limited to 3 attempts with zero backoff

**Tests**: `tests/test_sync.py::test_testing_mode`, `tests/test_async.py::test_testing_mode`

### Scenario CF-S12: Testing mode — cap=True with lower user attempts

**Given** `_Testing(2, cap=True)`
**When** `get_attempts(1)` is called (user specified 1)
**Then** it MUST return `1` (the lower value)

**Tests**: `tests/test_config.py::TestTesting::test_cap_true`

### Scenario CF-S13: Testing mode — cap=True with higher user attempts

**Given** `_Testing(2, cap=True)`
**When** `get_attempts(3)` is called (user specified 3)
**Then** it MUST return `2` (the testing cap)

**Tests**: `tests/test_config.py::TestTesting::test_cap_true`

### Scenario CF-S14: Testing mode — cap=False ignores user attempts

**Given** `_Testing(2, cap=False)`
**When** `get_attempts(1)` or `get_attempts(3)` is called
**Then** it MUST always return `2` (the testing value)

**Tests**: `tests/test_config.py::TestTesting::test_cap_false`

### Scenario CF-S15: Testing mode — cap=True with None user attempts

**Given** `_Testing(100, cap=True)`
**When** `get_attempts(None)` is called (user specified unlimited)
**Then** it MUST return `100` (the testing value, since `None` means unlimited)

**Tests**: `tests/test_config.py::TestTesting::test_cap_true_with_none`

### Scenario CF-S16: Testing mode — context manager basic

**Given** testing mode is not active
**When** `with set_testing(True, attempts=3):` is entered and exited
**Then** `is_testing()` MUST be `True` inside and `False` after

**Tests**: `tests/test_config.py::TestTesting::test_context_manager`

### Scenario CF-S17: Testing mode — context manager sync retry

**Given** a sync `retry_context` inside `with set_testing(True, attempts=3):`
**When** the block always raises `ValueError`
**Then** exactly 3 attempts MUST be made with zero backoff, and testing state MUST be restored on exit

**Tests**: `tests/test_sync.py::test_testing_mode_context`

### Scenario CF-S18: Testing mode — context manager async retry

**Given** an async `retry_context` inside `with set_testing(True, attempts=3):`
**When** the block always raises `ValueError`
**Then** exactly 3 attempts MUST be made with zero backoff, and testing state MUST be restored on exit

**Tests**: `tests/test_async.py::test_testing_mode_context`

### Scenario CF-S19: Testing mode — nested context managers

**Given** nested `set_testing` context managers
**When** the inner context sets `attempts=5, cap=True` and the outer sets `attempts=3`
**Then** each level MUST have its own settings and MUST restore the previous level on exit

**Tests**: `tests/test_config.py::TestTesting::test_context_manager_nested`

### Scenario CF-S20: Testing mode — context manager exception safety

**Given** a `set_testing(True)` context manager
**When** an exception is raised inside the context
**Then** testing state MUST be restored to the previous state on exit

**Tests**: `tests/test_config.py::TestTesting::test_context_manager_exception`

### Scenario CF-S21: Concurrent config initialization

**Given** `_Config._init_on_first_retry` is called
**When** another thread has already initialized hooks while waiting for the lock
**Then** the method MUST detect that initialization already occurred and not re-initialize

**Tests**: `tests/test_config.py::test_config_init_concurrently`

### Scenario CF-S22: conftest auto-reset

**Given** the `_reset_config` autouse fixture
**When** each test begins
**Then** stamina MUST be active and retry hooks MUST be reset to defaults

**Tests**: `tests/conftest.py::_reset_config`

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
