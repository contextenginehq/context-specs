# CLI Specification

## Commands

| Command   | Description                              |
|-----------|------------------------------------------|
| `build`   | Build context cache from source documents |
| `resolve` | Resolve context for a query              |
| `inspect` | Inspect cache state and metadata         |

> v0 uses a single `build` command that ingests source documents and produces a cache in one step. There is no separate `ingest` command. Future versions MAY split ingestion and compilation if an intermediate format is needed.

> The MCP server is provided as a standalone executable (`mcp-context-server`) and is not embedded in the CLI for v0. Future versions MAY provide a `context serve` convenience wrapper without altering protocol behavior.

---

## Command Reference

### `context build`

Build a context cache from source documents.

```bash
context build --sources <path> --cache <path> [--force]
```

| Flag | Type | Required | Description |
|------|------|----------|-------------|
| `--sources` | path | yes | Directory containing `.md` source files |
| `--cache` | path | yes | Output cache directory |
| `--force` | flag | no | Remove existing cache before building |

- Reads all `.md` files recursively from the sources directory
- Fails if `--cache` directory exists (unless `--force` is given)
- Output is a complete, self-contained cache directory

### `context resolve`

Resolve context for a query against a built cache.

```bash
context resolve --cache <path> --query <string> --budget <integer> [--format json|pretty]
```

| Flag | Type | Required | Description |
|------|------|----------|-------------|
| `--cache` | path | yes | Path to a built cache directory |
| `--query` | string | yes | Search query (empty string is allowed) |
| `--budget` | integer | yes | Maximum token budget (minimum: 0) |
| `--format` | string | no | Output format: `json` (default) or `pretty` |

- Output is written to stdout as JSON
- Output is byte-identical across runs for the same cache, query, and budget
- See `core/mcp/context.resolve.md` for the normative output schema

### `context inspect`

Inspect cache state and metadata.

```bash
context inspect --cache <path>
```

| Flag | Type | Required | Description |
|------|------|----------|-------------|
| `--cache` | path | yes | Path to a built cache directory |

Output:

```json
{
  "cache_version": "sha256:...",
  "document_count": 42,
  "total_bytes": 184729,
  "valid": true
}
```

- Loads the manifest and reports cache metadata
- The `valid` field indicates whether the cache passes integrity checks (manifest exists, document files exist, hashes match filenames, no orphan files)

---

## Requirements

CLI must:

- **Be scriptable** — All commands work non-interactively
- **Emit JSON by default** — Machine-readable output for automation
- **Support CI usage** — Exit codes and output suitable for CI pipelines

## Output Formats

- JSON (default for `resolve`, `inspect`)
- Human-readable (`--format pretty` where applicable)

## Error Output

Error messages are written to stderr as plain text. The error output format is implementation-defined and not part of the determinism contract.

## Exit Codes

| Code | Meaning                           | MCP Error Code      |
|------|-----------------------------------|---------------------|
| 0    | Success                           | (none)              |
| 1    | Usage error (bad arguments)       | (none)              |
| 2    | Invalid query                     | `invalid_query`     |
| 3    | Invalid budget                    | `invalid_budget`    |
| 4    | Cache missing                     | `cache_missing`     |
| 5    | Cache invalid                     | `cache_invalid`     |
| 6    | I/O error                         | `io_error`          |
| 7    | Internal error                    | `internal_error`    |

Exit codes 2-7 are **frozen** for v0 and map 1:1 to MCP error codes. Exit code 1 is reserved for argument parsing and usage errors.
