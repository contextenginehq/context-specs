# Context Observability

> **Note:** This is a paid component.

## Purpose

Provide visibility into context system behavior without altering it.

## Signals

- **Context hit rate** — Percentage of queries served from cache
- **Cache freshness** — Age of cache relative to source documents
- **Token utilization** — How efficiently token budgets are used

## Constraints

Observability must not alter context selection behavior.

## Principle

Observability is read-only. It measures and reports but never influences the context selection algorithm or cache behavior.
