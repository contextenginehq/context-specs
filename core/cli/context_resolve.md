# Context Resolve CLI

## Purpose

`context resolve` is the primary query interface for the Context system.

It resolves a natural-language query against a pre-built context cache and returns a deterministic, explainable document selection suitable for:

* LLM prompting
* MCP tool responses
* Debugging and inspection
* Offline / on‑prem usage

`resolve` is **pure with respect to its inputs**: same cache, query, and budget always produce the same output.

---

## Command Synopsis

```bash
context resolve \
  --cache <path> \
  --query <string> \
  --budget <tokens> \
  [--format json|pretty]
```

---

## Required Arguments

### `--cache <path>`

Path to a built context cache directory.

Requirements:

* Must exist
* Must contain `manifest.json`, `index.json`, and `documents/`
* Must pass cache verification

Failure modes:

* Path does not exist → error
* Path is not a valid cache → error

---

### `--query <string>`

The query string used for document selection.

Rules:

* Treated as opaque UTF‑8 text
* Lowercased and split on whitespace internally
* Empty query is allowed (results in score 0.0 for all documents)

---

### `--budget <integer>`

Maximum token budget for selected documents.

Rules:

* Must be ≥ 0
* Budget of `0` always returns an empty document list
* Token counting uses the v0 approximation

---

## Optional Arguments

### `--format <json|pretty>`

Controls output formatting.

| Value    | Behavior                          |
| -------- | --------------------------------- |
| `json`   | Compact JSON, no extra whitespace |
| `pretty` | Pretty‑printed JSON (default)     |

Output content is identical in both modes; only formatting differs.

---

## Output

### Successful Resolution

On success, `context resolve` writes a single JSON object to **stdout**.

The output is a serialized `SelectionResult`:

```json
{
  "documents": [
    {
      "id": "docs/deployment.md",
      "version": "sha256:...",
      "content": "...",
      "score": 0.92,
      "tokens": 847,
      "why": {
        "query_terms": ["deployment"],
        "term_matches": 12,
        "total_words": 156
      }
    }
  ],
  "selection": {
    "query": "deployment",
    "budget": 4000,
    "tokens_used": 3241,
    "documents_considered": 42,
    "documents_selected": 3,
    "documents_excluded_by_budget": 9
  }
}
```

Field order is **fixed** and must not change within a major version.

---

## Error Handling

All errors are written to **stderr**.

On error:

* No output is written to stdout
* Exit code is non‑zero

### Error Categories

| Category         | Description                             | Exit Code | MCP Code         |
| ---------------- | --------------------------------------- | --------- | ---------------- |
| Invalid query    | Query is malformed or invalid           | 2         | `invalid_query`  |
| Invalid budget   | Budget value is invalid                 | 3         | `invalid_budget` |
| Cache missing    | Cache path is missing or not a directory| 4         | `cache_missing`  |
| Cache invalid    | Cache exists but violates invariants    | 5         | `cache_invalid`  |
| IO error         | Filesystem access failure               | 6         | `io_error`       |
| Internal error   | Invariant violation (should not happen) | 7         | `internal_error` |

### Example Error Output

```text
error: invalid cache at ./cache
reason: missing manifest.json
```

---

## Determinism Contract

`context resolve` MUST satisfy:

```bash
context resolve --cache C --query Q --budget B > a.json
context resolve --cache C --query Q --budget B > b.json

diff a.json b.json  # must be empty
```

This holds across:

* multiple invocations
* different machines
* different operating systems

Provided the cache contents are identical.

---

## Non‑Goals (v0)

`context resolve` intentionally does **not**:

* Modify caches
* Perform network access
* Use embeddings or ML models
* Truncate documents
* Stream partial output
* Return raw scores without documents

---

## Relationship to MCP

`context resolve` defines the **authoritative resolution semantics**.

MCP integrations MUST:

* Call equivalent resolution logic
* Return identical JSON structures
* Preserve ordering, scoring, and explainability

The CLI is the reference implementation.

---

## Invariants

1. Deterministic output
2. No side effects
3. Full explainability
4. Strict budget enforcement
5. Portable output

---

## What This Spec Forbids

* Non‑deterministic output
* Partial document inclusion
* Hidden scoring logic
* Output formats other than JSON
* Cache mutation during resolution
