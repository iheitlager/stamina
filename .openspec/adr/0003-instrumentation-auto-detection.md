---
ADR: "0003"
Title: Auto-detection chain for instrumentation backends
Status: Accepted
Date: 2026-02-22
Context: Reverse-engineered from implementation
---

# ADR-0003: Auto-detection chain for instrumentation backends

## Status

Accepted

## Context

stamina aims to be "observable by default" — retries should be logged or metered without users writing any instrumentation code. However, different projects use different observability stacks: some use Prometheus for metrics, some use structlog for structured logging, and some use only Python's stdlib logging.

The question: how should stamina decide which instrumentation to activate?

## Decision

Use an import-based auto-detection chain with a fixed priority order:

```
1. prometheus_client importable? → Add PrometheusOnRetryHook
2. structlog importable?         → Add StructlogOnRetryHook
   else                          → Add LoggingOnRetryHook
```

**Key design choices:**

1. **Prometheus and structlog are additive.** If both are installed, both hooks are active (metrics AND logging). This is intentional — they serve different purposes.
2. **structlog vs logging is exclusive.** If structlog is available, it replaces stdlib logging (not both). structlog is strictly superior when available.
3. **Detection happens at hook init time, not import time.** The `get_default_hooks()` function returns `RetryHookFactory` instances. The actual `import prometheus_client` / `import structlog` happens inside the factory functions, deferred until the first retry is scheduled.
4. **Users can override completely.** `set_on_retry_hooks()` replaces the auto-detected hooks. Passing `None` resets to defaults. Passing `()` disables instrumentation entirely.

## Consequences

**Benefits:**
- Zero-configuration observability — install structlog or prometheus_client, and stamina uses them automatically
- No import-time cost — heavy libraries like prometheus_client are only imported when actually needed
- Users who don't want instrumentation can disable it with one call
- The fallback to stdlib logging means retries are always visible somewhere

**Trade-offs:**
- "Magic" behavior — instrumentation changes based on installed packages, which can surprise users
- No way to use stdlib logging when structlog is installed (without custom hooks)
- The detection order is fixed and not configurable
- If prometheus_client is installed but not configured (no registry), the counter is still created

## Evidence

- `src/stamina/instrumentation/_hooks.py:31-51` — `get_default_hooks()` detection chain
- `src/stamina/instrumentation/_hooks.py:54-69` — `set_on_retry_hooks()` override mechanism
- `src/stamina/instrumentation/_prometheus.py:67` — `PrometheusOnRetryHook = RetryHookFactory(init_prometheus)`
- `src/stamina/instrumentation/_structlog.py:35` — `StructlogOnRetryHook = RetryHookFactory(init_structlog)`
- `src/stamina/instrumentation/_logging.py:40` — `LoggingOnRetryHook = RetryHookFactory(init_logging)`
