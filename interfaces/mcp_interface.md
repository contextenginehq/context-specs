# MCP Interface Specification

> This document describes MCP integration for v0. Future versions MAY extend response structure.

## Purpose

Expose context resolution to agents via the Model Context Protocol.

MCP is the first-class integration surface for agents.

## Transport

v0 uses stdio only (JSON-RPC 2.0, newline-delimited).

HTTP, SSE, and WebSocket transports are out of scope for v0.

## Core Tools

| Tool               | Description                              |
|--------------------|------------------------------------------|
| `context.resolve`        | Resolve context for a query              |
| `context.list_caches`    | List available context caches            |
| `context.inspect_cache`  | Inspect cache state and metadata         |

## context.resolve

See `core/mcp/context.resolve.md` for the normative specification.

### Inputs

| Parameter | Type    | Description                              |
|-----------|---------|------------------------------------------|
| `cache`   | string  | Identifier of the context cache          |
| `query`   | string  | The query to resolve context for         |
| `budget`  | integer | Token budget constraint (minimum: 0)     |

### Outputs

| Field        | Description                              |
|--------------|------------------------------------------|
| `documents`  | Ordered document bundle                  |
| `selection`  | Selection metadata                       |

v0 does not include provenance data. Future versions MAY include optional provenance metadata without affecting selection semantics.

## context.list_caches

### Purpose

Enumerate available context caches under the server's cache root.

### Inputs

No parameters required.

### Output Schema

```json
{
  "caches": [
    {
      "path": "string",
      "has_manifest": "boolean"
    }
  ]
}
```

### Behavior

- Enumerates immediate subdirectories of the configured cache root
- For each subdirectory, checks whether `manifest.json` exists as a regular file
- No JSON parsing or validation is performed — this is a discovery tool, not a validation tool
- Results are sorted by `path` ascending (UTF-8 byte order) for determinism

### Errors

| Error Code   | Condition                              |
|--------------|----------------------------------------|
| `io_error`   | Cache root is not accessible           |

---

## context.inspect_cache

### Purpose

Return metadata and optional integrity status for a specific cache.

### Inputs

| Parameter | Type   | Description                        |
|-----------|--------|------------------------------------|
| `cache`   | string | Identifier of the context cache    |

### Output Schema

```json
{
  "cache_version": "string",
  "document_count": "integer",
  "total_bytes": "integer",
  "valid": "boolean"
}
```

### Behavior

- Resolves the cache name against the configured cache root (with traversal protection)
- Loads the cache manifest and returns structural metadata
- Sets `valid` to `false` if the manifest is missing, unparseable, or lacks required fields
- `total_bytes` is the sum of file sizes in the cache directory (non-recursive, no symlinks)

### Errors

| Error Code      | Condition                                    |
|-----------------|----------------------------------------------|
| `cache_missing` | Cache path does not exist or is not a directory |
| `cache_invalid` | Manifest exists but is not valid JSON        |
| `io_error`      | Filesystem error reading manifest            |

---

## Protocol Requirements

- All responses must be deterministic
- Errors must be explicit and typed (see `core/mcp/error_schema.md`)

## Authentication

For stdio transport, authentication is delegated to the operating system process boundary and is out of scope for v0.
