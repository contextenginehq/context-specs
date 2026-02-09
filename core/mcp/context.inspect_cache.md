# MCP Tool Spec — context.inspect_cache (v0)

## Purpose

Return structural and integrity information about a specific cache.

This tool provides:

- cache identity
- structural validity
- basic size metrics

It does not expose document content.

---

## Determinism Contract

Given identical cache bytes on disk:

- Output MUST be byte-identical
- Field order MUST be stable
- No timestamps from the runtime environment may appear
- No filesystem metadata (mtime, inode, etc.) may appear

---

## Method Name

```
context.inspect_cache
```

---

## Input Schema

```json
{
  "type": "object",
  "required": ["cache"],
  "additionalProperties": false,
  "properties": {
    "cache": {
      "type": "string",
      "description": "Path to cache directory"
    }
  }
}
```

---

## Output Schema (normative)

```json
{
  "type": "object",
  "required": [
    "cache_version",
    "document_count",
    "total_bytes",
    "valid"
  ],
  "additionalProperties": false,
  "properties": {
    "cache_version": { "type": "string" },
    "document_count": { "type": "integer", "minimum": 0 },
    "total_bytes": { "type": "integer", "minimum": 0 },
    "valid": { "type": "boolean" }
  }
}
```

Field order MUST be exactly:

1. `cache_version`
2. `document_count`
3. `total_bytes`
4. `valid`

---

## Output Field Semantics

### `cache_version`

Value from:

`manifest.json → cache_version`

If manifest cannot be parsed:

- `cache_version = ""`
- `valid = false`

This ensures deterministic output even for corrupted caches.

### `document_count`

Value from:

`manifest.json → document_count`

If manifest unreadable:

- `document_count = 0`
- `valid = false`

No filesystem scanning is used to compute this number.

The manifest is authoritative.

### `total_bytes`

Total size in bytes of all files under the cache directory, computed by a deterministic, non-recursive directory walk that ignores filesystem metadata and uses only file sizes. Symlink targets are not followed.

If any file read fails, `valid = false` and `total_bytes` MUST be `0`.

### `valid`

`true` iff:

- `manifest.json` exists and can be parsed
- `document_count` and `cache_version` are present
- `total_bytes` computed without IO errors

---

## Error Behavior

### `cache_missing`

`cache` does not exist or is not a directory.

### `io_error`

Filesystem read failure when accessing the cache directory.

No other errors permitted.
