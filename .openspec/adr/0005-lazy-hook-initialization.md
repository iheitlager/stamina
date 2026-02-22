---
ADR: "0005"
Title: Lazy hook initialization via RetryHookFactory
Status: Accepted
Date: 2026-02-22
Context: Reverse-engineered from implementation
---

# ADR-0005: Lazy hook initialization via RetryHookFactory

## Status

Accepted

## Context

stamina auto-detects instrumentation backends (Prometheus, structlog, stdlib logging) and initializes them as retry hooks. However, initializing these backends has costs:

- **prometheus_client**: Creating a `Counter` registers it globally; importing the module is heavy
- **structlog**: Calling `get_logger()` at import time may use an unconfigured processor chain
- **stdlib logging**: `getLogger()` is cheap but still unnecessary if no retries ever occur

Many applications import stamina but may never trigger a retry (happy path). Paying initialization costs at import time is wasteful.

## Decision

Introduce `RetryHookFactory` — a frozen dataclass wrapping a `Callable[[], RetryHook]` — to defer hook initialization until the first scheduled retry.

**Key design choices:**

1. **Two-phase initialization.** `get_default_hooks()` returns `RetryHookFactory` instances (phase 1: detection). `init_hooks()` calls each factory's `hook_factory()` to produce the actual hooks (phase 2: initialization).
2. **Phase 2 triggers on first retry.** `_Config.on_retry` uses a getter that calls `_init_on_first_retry()` on first access. This method acquires a lock, checks for concurrent initialization, resolves all factories, and replaces the getter with a direct lambda.
3. **Double-checked locking.** `_init_on_first_retry` checks `self._get_on_retry == self._init_on_first_retry` inside the lock to handle the case where another thread initialized while waiting.
4. **Mixed hooks supported.** `set_on_retry_hooks` accepts both `RetryHook` instances and `RetryHookFactory` instances in the same tuple. `init_hooks` resolves factories and passes through direct hooks.

## Consequences

**Benefits:**
- Zero import-time cost for instrumentation — prometheus_client and structlog are not imported until needed
- Applications that never retry pay no initialization cost
- structlog's `get_logger()` is called after the user has configured structlog (not at stamina import time)
- Prometheus counter is created only when retries actually happen

**Trade-offs:**
- Added complexity — two types (`RetryHook` and `RetryHookFactory`) instead of one
- The double-checked locking pattern is subtle and easy to get wrong
- First retry has slightly higher latency due to initialization (one-time cost)
- The getter replacement pattern (`self._get_on_retry = lambda: self._on_retry`) is unusual

## Evidence

- `src/stamina/instrumentation/_data.py:91-103` — `RetryHookFactory` dataclass
- `src/stamina/instrumentation/_hooks.py:15-28` — `init_hooks()` resolves factories
- `src/stamina/instrumentation/_hooks.py:31-51` — `get_default_hooks()` returns factories
- `src/stamina/_config.py:76-78` — `_on_retry = None`, `_get_on_retry = self._init_on_first_retry`
- `src/stamina/_config.py:110-124` — `_init_on_first_retry` double-checked locking
- `tests/test_config.py:27-41` — `test_config_init_concurrently` tests the concurrent path
