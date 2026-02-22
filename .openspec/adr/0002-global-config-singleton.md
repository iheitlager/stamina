---
ADR: "0002"
Title: Global singleton config with thread-safe Lock
Status: Accepted
Date: 2026-02-22
Context: Reverse-engineered from implementation
---

# ADR-0002: Global singleton config with thread-safe Lock

## Status

Accepted

## Context

stamina needs a way to manage global state: whether retrying is active, whether testing mode is enabled, and which instrumentation hooks to call. This state must be accessible from any call site (decorators, context managers, caller objects) and must be safe for concurrent access.

The alternatives considered (inferred from design):

1. **Dependency injection** — pass config to every retry call. Clean but burdensome for users; every `@retry` would need a config parameter.
2. **Thread-local storage** — per-thread config. Avoids locks but complicates async (where multiple coroutines share a thread).
3. **Module-level singleton with Lock** — single global config, explicit locking on mutations.

## Decision

Use a module-level singleton `CONFIG = _Config(Lock())` with `threading.Lock` for all mutations.

**Key design choices:**

1. **Single `_Config` instance.** Created at module import time in `_config.py`. All public functions (`set_active`, `is_active`, `set_testing`, `is_testing`) operate on this singleton.
2. **Lock on writes, not reads.** Properties like `is_active` and `testing` read without locking (atomic in CPython due to GIL). Setters acquire the lock.
3. **Delayed hook initialization.** `_Config._on_retry` starts as `None`. The `on_retry` property getter triggers `_init_on_first_retry()` on first access, which acquires the lock and double-checks initialization hasn't happened concurrently.
4. **No reset method.** The conftest fixture manually resets state between tests by calling `set_active(True)` and `set_on_retry_hooks(None)`.

## Consequences

**Benefits:**
- Zero-configuration for users — retrying works immediately after import
- Simple mental model — one global state, not per-thread or per-context
- `set_active(False)` disables retrying everywhere instantly (useful for testing)
- Lock prevents race conditions in multi-threaded initialization

**Trade-offs:**
- Global mutable state — harder to test in isolation (requires reset fixture)
- Not async-safe in theory (Lock blocks the event loop), though mutations are rare and fast
- Cannot have different retry configurations for different parts of an application
- Testing mode affects all retries globally, not per-decorator

## Evidence

- `src/stamina/_config.py:127` — `CONFIG = _Config(Lock())`
- `src/stamina/_config.py:84-87` — `is_active` setter acquires lock
- `src/stamina/_config.py:93-96` — `testing` setter acquires lock
- `src/stamina/_config.py:110-124` — `_init_on_first_retry` double-checked locking
- `tests/conftest.py:21-27` — `_reset_config` autouse fixture
