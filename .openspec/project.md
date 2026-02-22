---
Project: stamina
Description: Production-grade retry library for Python with sensible defaults
License: MIT
Author: Hynek Schlawack
Repository: https://github.com/hynek/stamina
Python: ">=3.10"
Status: Implemented
---

# Project: stamina

## Overview

**stamina** is a production-grade retry library for Python that wraps [Tenacity](https://tenacity.readthedocs.io/) with an opinionated, ergonomic API. It provides sensible defaults for retry behavior, built-in instrumentation, and first-class async support including Trio.

## Philosophy

- **Sensible defaults over configuration**: retry parameters have battle-tested defaults
- **Ergonomic API**: keyword-only arguments prevent misconfiguration
- **Observable by default**: automatic instrumentation via Prometheus, structlog, or stdlib logging
- **Testing-friendly**: global toggle and testing mode with zero-backoff
- **Async-native**: first-class support for asyncio and Trio

## Architecture

The library is organized into 4 cohesive domains:

| # | Domain | Spec | Description |
|---|--------|------|-------------|
| 1 | Retry Core | [001-retry-core](specs/001-retry-core/spec.md) | `retry` decorator, `retry_context` iterator, `Attempt` context manager, backoff formula |
| 2 | Caller API | [002-caller-api](specs/002-caller-api/spec.md) | `RetryingCaller`, `AsyncRetryingCaller`, bound variants, `.on()` binding |
| 3 | Configuration | [003-configuration](specs/003-configuration/spec.md) | Global toggle, testing mode, thread-safe state |
| 4 | Instrumentation | [004-instrumentation](specs/004-instrumentation/spec.md) | Retry hooks, auto-detection chain, Prometheus/structlog/logging backends |

## Key Dependencies

- **tenacity**: Underlying retry engine (wrapped, not exposed)
- **prometheus_client** (optional): Prometheus metrics
- **structlog** (optional): Structured logging
- **sniffio** (optional): Async library detection for Trio support

## Source Layout

```
src/stamina/
├── __init__.py                    # Public API re-exports
├── _core.py                       # retry, retry_context, Attempt, Caller classes
├── _config.py                     # Global configuration, testing mode
├── typing.py                      # Public type re-exports
├── py.typed                       # PEP 561 marker
└── instrumentation/
    ├── __init__.py                # Instrumentation public API
    ├── _data.py                   # RetryDetails, RetryHook protocol, RetryHookFactory
    ├── _hooks.py                  # Hook management, auto-detection
    ├── _logging.py                # stdlib logging backend
    ├── _prometheus.py             # Prometheus backend
    └── _structlog.py              # structlog backend
```

## Test Layout

```
tests/
├── conftest.py                    # Fixtures: reset config, on parametrize, anyio backends
├── test_sync.py                   # Sync retry, generators, backoff hooks, caller tests
├── test_async.py                  # Async retry, async generators, async caller tests
├── test_config.py                 # Configuration and testing mode tests
├── test_instrumentation.py        # Hook management, guess_name, context manager hooks
├── test_structlog.py              # structlog integration tests
├── test_packaging.py              # Version metadata test
└── typing/                        # Static type-checking stubs
```
