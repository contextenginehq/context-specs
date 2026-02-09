# MCP Tool Spec — context.list_caches (v0)

## Purpose

Enumerate available caches on the local system.

This is a discovery tool, not an inspection tool.
It does not validate cache correctness beyond existence.

For detailed information, use `context.inspect_cache`.

---

## Determinism Contract

Given the same filesystem state:

- The returned list MUST be identical
- Ordering MUST be stable
- Output MUST be byte-identical across runs

---

## Method Name

```
context.list_caches
```

---

## Input Schema

```json
{
  "type": "object",
  "required": ["root"],
  "additionalProperties": false,
  "properties": {
    "root": {
      "type": "string",
      "description": "Directory containing cache directories"
    }
  }
}
```

---

## Input Semantics

`root` is a directory whose immediate children are considered candidate caches.

Example layout:

```
/var/context/
├── cache-a/
├── cache-b/
└── notes.txt   ← ignored
```

Only immediate subdirectories are considered.

No recursion.

---

## Output Schema

```json
{
  "type": "object",
  "required": ["caches"],
  "additionalProperties": false,
  "properties": {
    "caches": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["path", "has_manifest"],
        "additionalProperties": false,
        "properties": {
          "path": { "type": "string" },
          "has_manifest": { "type": "boolean" }
        }
      }
    }
  }
}
```

---

## Output Semantics

Each entry represents a directory inside `root`.

### `path`

Absolute or root-relative path to cache directory.

Must match filesystem path exactly as resolved by implementation.

### `has_manifest`

True if:

`<path>/manifest.json` exists AND is a regular file

No JSON parsing is performed.

No validation is performed.

This keeps the tool:

- fast
- deterministic
- side-effect free
- clearly scoped

---

## Ordering Rules (frozen)

Caches MUST be sorted by:

1️⃣ `path` ascending (lexicographical, UTF-8 byte order)

No other ordering is permitted.

---

## Inclusion Rules

A directory is included if:

✅ it is an immediate child of `root`
❌ files are ignored
❌ symlink targets are not followed
❌ nested directories are not scanned

This avoids:

- recursion nondeterminism
- filesystem surprises
- performance variability

---

## Error Behavior

### `cache_missing`

`root` does not exist or is not a directory.

### `io_error`

Filesystem read failure when enumerating directory.

No other errors permitted.
