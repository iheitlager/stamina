---
ADR: "0001"
Title: Wrap Tenacity instead of building retry from scratch
Status: Accepted
Date: 2026-02-22
Context: Reverse-engineered from implementation
---

# ADR-0001: Wrap Tenacity instead of building retry from scratch

## Status

Accepted

## Context

Building a production-grade retry library requires solving many non-trivial problems: exponential backoff computation, jitter, timeout tracking, attempt counting, async sleep coordination (asyncio + Trio), and correct exception handling across sync/async/generator variants. Tenacity is a mature, battle-tested library that solves all of these.

However, Tenacity's API is extremely flexible — it exposes dozens of parameters, multiple composition patterns, and low-level primitives. This flexibility makes it powerful but also error-prone for common use cases. Most developers need the same thing: retry on exception with exponential backoff, jitter, and sensible limits.

## Decision

Wrap Tenacity as an internal implementation detail, exposing a much smaller, opinionated API surface.

**Key design choices:**

1. **Tenacity is never exposed to users.** It is imported as `_t` (private alias). No Tenacity types appear in stamina's public API.
2. **stamina controls all Tenacity configuration.** Users set high-level parameters (`wait_initial`, `wait_max`, `wait_jitter`, `wait_exp_base`) and stamina translates them into Tenacity's `wait`, `stop`, `retry`, and `before_sleep` callbacks.
3. **`reraise=True` always.** Tenacity's default wraps exceptions in `RetryError` — stamina always reraises the original exception.
4. **Custom backoff via piggyback state.** Rather than using Tenacity's `wait` API directly for custom backoff hooks, stamina attaches custom backoff values to `RetryCallState` via `setattr` (`_CUSTOM_BACKOFF_ATTR`), then reads them back in the `wait` callback. This avoids fighting Tenacity's wait pipeline.
5. **Async sleep abstraction.** stamina provides `_smart_sleep` that detects the running async library (via sniffio) and dispatches to `asyncio.sleep` or `trio.sleep`. This is passed to Tenacity's `AsyncRetrying(sleep=...)`.

## Consequences

**Benefits:**
- Inherits Tenacity's correctness and edge-case handling (thread safety, signal handling, etc.)
- Dramatically smaller API surface for users (7 parameters vs Tenacity's 20+)
- All keyword-only arguments prevent positional argument mistakes
- Can evolve the public API independently of Tenacity's internals

**Trade-offs:**
- Coupled to Tenacity's internal types (`RetryCallState`, `AttemptManager`, `Retrying`, `AsyncRetrying`)
- The piggyback state pattern (`_CUSTOM_BACKOFF_ATTR`) is acknowledged as "naughty" in the source code — it works but relies on Tenacity not using that attribute name
- Testing requires understanding both stamina's API and Tenacity's behavior
- Version upgrades of Tenacity may require internal changes even if stamina's public API doesn't change

## Evidence

- `src/stamina/_core.py:30` — `import tenacity as _t` (private alias)
- `src/stamina/_core.py:528` — `reraise=True` always set
- `src/stamina/_core.py:114` — piggyback state: `setattr(retry_state, _CUSTOM_BACKOFF_ATTR, custom_backoff)`
- `src/stamina/_core.py:44-57` — `_smart_sleep` async library detection
- `pyproject.toml` — tenacity listed as hard dependency
