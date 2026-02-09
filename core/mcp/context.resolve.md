# MCP Protocol Mapping: Context Resolve

## Purpose

This document defines how the **Context Resolve** capability is exposed via the Model Context Protocol (MCP).

The MCP mapping is a **pure projection** of the `context resolve` semantics:

* Same inputs
* Same outputs
* Same determinism guarantees

The CLI remains the reference implementation; MCP is a transport/interface layer.

---

## MCP Tool Name

```
context.resolve
```

The name is version-stable. Semantic changes require a new tool or versioned parameters.

---

## Tool Description (for MCP registry)

> Resolve a natural-language query against a pre-built context cache and return a deterministic, explainable set of documents within a token budget.

---

## Input Schema

### MCP Tool Arguments

```json
{
  "cache": {
    "type": "string",
    "description": "Path or identifier of a built context cache"
  },
  "query": {
    "type": "string",
    "description": "Natural-language query"
  },
  "budget": {
    "type": "integer",
    "minimum": 0,
    "description": "Maximum token budget"
  }
}
```

### Input Rules

* All inputs are required
* Inputs are validated before execution
* Invalid inputs result in MCP tool error

---

## Output Schema

The MCP tool returns **exactly** the serialized `SelectionResult` object.

```json
{
  "documents": [
    {
      "id": "string",
      "version": "string",
      "content": "string",
      "score": "number",
      "tokens": "integer",
      "why": {
        "query_terms": ["string"],
        "term_matches": "integer",
        "total_words": "integer"
      }
    }
  ],
  "selection": {
    "query": "string",
    "budget": "integer",
    "tokens_used": "integer",
    "documents_considered": "integer",
    "documents_selected": "integer",
    "documents_excluded_by_budget": "integer"
  }
}
```

Field names, structure, and ordering MUST match the CLI output exactly.

v0 does not include provenance data. Future versions MAY include optional provenance metadata without affecting selection semantics.

---

## Determinism Contract

For any MCP client:

```text
call(context.resolve, { cache: C, query: Q, budget: B })
```

The returned JSON MUST be byte-identical to:

```bash
context resolve --cache C --query Q --budget B
```

Given the same cache contents.

---

## Error Mapping

Errors reflect facts about the world, not recovery strategies.

### Error Types (v0, frozen)

| Error Code        | Meaning                                                   |
| ----------------- | --------------------------------------------------------- |
| `cache_missing`   | Cache path does not exist or is not a directory          |
| `cache_invalid`   | Cache exists but violates invariants                      |
| `invalid_query`   | Query cannot be processed as specified                    |
| `invalid_budget`  | Budget is invalid (e.g., negative or exceeds hard max)    |
| `io_error`        | Filesystem or OS error                                    |
| `internal_error`  | Implementation violated spec or invariants                |

### MCP Error Object Schema (v0)

Every MCP failure response MUST have exactly this structure:

```json
{
  "error": {
    "code": "cache_invalid",
    "message": "Cache exists but is invalid"
  }
}
```

No additional top-level keys are allowed in an error response.

#### Required Fields

| Field   | Type   | Description                                        |
| ------- | ------ | -------------------------------------------------- |
| `code`  | string | Machine-readable error identifier                  |
| `message` | string | Human-readable, stable, non-diagnostic description |

Both fields are mandatory.

### Example MCP Error

```json
{
  "error": {
    "code": "cache_invalid",
    "message": "Cache exists but is invalid"
  }
}
```

### Message Field Rules

The `message` field:

* MUST be a human-readable string
* MUST NOT include file paths, stack traces, hash values, or OS error codes
* SHOULD be stable across versions
* SHOULD be short (one sentence)

#### Recommended Canonical Messages (v0)

```json
{
  "cache_missing": "Cache does not exist",
  "cache_invalid": "Cache exists but is invalid",
  "invalid_query": "Query is invalid",
  "invalid_budget": "Budget is invalid",
  "io_error": "I/O error occurred",
  "internal_error": "Internal error"
}
```

### Determinism in Errors

Given the same inputs and system state, the error code and message MUST be identical byte-for-byte.

### Forbidden Patterns

* Extra fields inside `error` (diagnostics do not belong in MCP)
* Mixed success + error payloads
* Nested or namespaced error codes (exact string match only)

### Recovery Policy

`context.resolve` must never auto-recover.

* `cache_missing` must not trigger build
* `cache_invalid` must not trigger rebuild

Recovery is a caller decision.

---

## Transport (v0)

The MCP server MUST use standard MCP transport: JSON-RPC 2.0 over stdin/stdout.

HTTP, SSE, WebSocket, or other transports are explicitly out of scope for v0.

Future versions MAY define additional transports.

Transport choice MUST NOT affect request semantics or response determinism.

---

## Streaming

### v0 Policy

* MCP streaming is **not supported** for `context.resolve`
* The tool returns a single complete response

Rationale:

* Determinism
* Simplicity
* Budget correctness

Streaming may be introduced in a future version under a new tool name.

---

## Security and Isolation

* The MCP tool performs no network access
* The tool only reads from the specified cache
* No environment variables affect resolution

This makes the tool safe for:

* on-prem deployments
* air-gapped systems
* regulated environments

---

## Relationship to Open-Core

### Open Source (Core)

* Tool schema
* Resolution semantics
* Deterministic selection

### Commercial Extensions (Out of Scope)

* Multi-cache federation
* Remote cache addressing
* Access control
* Usage accounting

These MUST NOT alter the semantics of `context.resolve`.

---

## Invariants

1. CLI and MCP outputs are identical
2. No hidden parameters
3. No side effects
4. Deterministic across machines

---

## What This Spec Forbids

* MCP-only behavior differences
* Streaming partial documents
* Non-JSON output
* Implicit cache mutation
* Transport-specific scoring logic
