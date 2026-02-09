# v0: The Context Compiler

## Mental Model

You're building a **compiler**, not a product UI.

```
docs/ → context build → cache/ → context resolve → JSON
```

Source in. Artifact out. Deterministic. Done.

---

## v0 Scope

### What Gets Built

| Component | Description |
|-----------|-------------|
| Static documentation ingestion | Read markdown files from disk |
| Immutable document versions | Content hash = version, never mutate |
| Offline cache build | Compile docs into queryable cache |
| Deterministic context selection | Same inputs → same outputs, always |
| JSON output | Machine-readable, no formatting |
| CLI | Command-line interface for all operations |
| MCP stdio server | Reference MCP server implementing `context.resolve` over stdio |

### What Does NOT Get Built

| Excluded | Why |
|----------|-----|
| Embeddings tuning | Not v0 |
| Learning from usage | Violates determinism |
| Real-time updates | Offline-first |
| Dashboards | Not a product UI |
| SaaS control plane | Not a service |
| HTTP/SSE/WebSocket transport | After stdio works |
| Observability | Paid feature, later |
| Policies | After core works |
| Provenance metadata | Requires semantic definition, deferred |
| Authentication | Stdio delegates to OS process boundary |

---

## The Commands

```bash
# Ingest documents into cache
context ingest ./docs --cache ./cache

# Build/rebuild cache (explicit, not automatic)
context build --cache ./cache

# Resolve context for a query
context resolve --cache ./cache --query "deployment" --budget 4000

# Inspect cache state
context inspect --cache ./cache
```

---

## Output Contract

All output is JSON to stdout. Always.

See `core/mcp/context.resolve.md` for the normative output schema. Example:

```json
{
  "documents": [
    {
      "id": "docs/deployment.md",
      "version": "sha256:a1b2c3...",
      "content": "...",
      "score": 0.95,
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
    "tokens_used": 3847,
    "documents_considered": 42,
    "documents_selected": 3,
    "documents_excluded_by_budget": 9
  }
}
```

Errors go to stderr. Exit codes are meaningful.

---

## Success Criteria

```bash
# Build cache
context ingest ./docs --cache ./cache
context build --cache ./cache

# Resolve twice
context resolve --cache ./cache --query "deployment" > a.json
context resolve --cache ./cache --query "deployment" > b.json

# Must be identical
diff a.json b.json  # Empty

# Rebuild and resolve again
rm -rf ./cache
context ingest ./docs --cache ./cache
context build --cache ./cache
context resolve --cache ./cache --query "deployment" > c.json

# Still identical
diff a.json c.json  # Empty
```

If any diff has output, v0 is broken.

---

## Implementation Constraints

- **Language**: Go, Rust, or TypeScript (pick one, commit)
- **Dependencies**: Minimal (stdlib + hashing + JSON)
- **Configuration**: Flags only, no config files yet
- **Storage**: Filesystem only, no database
- **Network**: None. Zero outbound calls.

---

## What This Proves

| Claim | Evidence |
|-------|----------|
| Deterministic | Diff test passes |
| Auditable | JSON output includes version and scoring explanation |
| Offline-capable | No network required |
| Scriptable | CLI with exit codes |
| Immutable versioning | Content hash in output |

---

## After v0

Only after the diff test passes on every commit:

1. HTTP/SSE transport for MCP
2. Configuration files
3. Provenance metadata
4. Multi-cache federation

Not before.
