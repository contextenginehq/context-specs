# MCP Error Object — JSON Schema (v0)

This document freezes the MCP error response format for **Context Resolve v0**.

This schema is **normative**. Any MCP implementation claiming v0 compatibility **MUST** produce errors that validate against this schema.

---

## Overview

An MCP error response has exactly one top-level key: `error`.

Successful responses MUST NOT include this object.

---

## JSON Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://context.dev/schemas/mcp/error-v0.json",
  "title": "MCP Error Response v0",
  "type": "object",
  "required": ["error"],
  "additionalProperties": false,

  "properties": {
    "error": {
      "type": "object",
      "required": ["code", "message"],
      "additionalProperties": false,

      "properties": {
        "code": {
          "type": "string",
          "enum": [
            "cache_missing",
            "cache_invalid",
            "invalid_query",
            "invalid_budget",
            "io_error",
            "internal_error"
          ]
        },
        "message": {
          "type": "string",
          "minLength": 1
        }
      }
    }
  }
}
```

---

## Canonical Error Codes (v0)

| Code             | Meaning                                        |
| ---------------- | ---------------------------------------------- |
| `cache_missing`  | Cache path does not exist or is not accessible |
| `cache_invalid`  | Cache exists but violates cache invariants     |
| `invalid_query`  | Query is malformed or invalid                  |
| `invalid_budget` | Budget value is invalid                        |
| `io_error`       | Filesystem or OS-level failure                 |
| `internal_error` | Implementation bug or invariant violation      |

No other values are valid in v0.

### Boundary: `cache_invalid` vs `io_error`

Both codes involve filesystem failures, but they have different semantics:

- **`cache_invalid`** — Structural invariant violation. The cache exists but is unusable due to a spec violation (missing manifest, malformed JSON, hash mismatch, missing document file). Retrying will not help.
- **`io_error`** — Environmental failure. The system cannot operate due to an OS-level problem (permission denied, disk read failure, file locked, disk full). Retry may succeed.

---

## Message Field Rules

The `message` field:

* MUST be a human-readable string
* MUST be stable across runs
* MUST NOT contain diagnostics (paths, hashes, stack traces)

### Recommended canonical messages

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

Implementations MAY use different messages, but message instability is discouraged.

---

## Forbidden Patterns

### Extra fields

```json
{
  "error": {
    "code": "cache_invalid",
    "message": "Cache exists but is invalid",
    "details": {}
  }
}
```

❌ Forbidden — fails schema validation.

---

### Mixed success and error

```json
{
  "documents": [],
  "error": { "code": "internal_error", "message": "Internal error" }
}
```

❌ Forbidden — error responses are exclusive.

---

## Determinism Contract

For identical inputs and system state:

* The same error **code** MUST be returned
* The same error **message** SHOULD be returned
* The serialized JSON MUST be byte-identical

Any deviation is a protocol violation.

---

## Versioning

* This schema applies to **MCP Context Resolve v0**
* Any extension requires a new schema version and `$id`
* Backward-incompatible changes are forbidden in v0
