# Context v0 Release Checklist

This checklist is ordered by risk, not by component. If any item in section 1 can drift, do not release.

---

## 1️⃣ Spec Compliance Freeze (non-negotiable)

### Selection behavior

- [x] Determinism test passes (end-to-end byte compare)
- [x] Ordering = score desc, id asc
- [x] Token counting = ceil(len / 4)
- [x] Documents never truncated
- [x] Budget never exceeded
- [x] Score range enforced [0.0, 1.0]
- [x] Zero budget returns empty selection

### Serialization contract

- [x] Field order matches spec examples
- [x] JSON encoding stable across runs
- [x] No nondeterministic timestamps
- [x] Floats serialized consistently
- [x] UTF-8 only
- [x] Newline at EOF

### Explainability

- [x] `why` always present on selected docs
- [x] Selection metadata complete
- [x] Output reconstructible from response

If any of these can drift → do not release.

---

## 2️⃣ Golden Test Suite (must be green)

### Cache system

- [x] Cache build deterministic across runs
- [x] Manifest identical byte-for-byte
- [x] Document hashing stable

### Selection engine

- [x] `resolve_basic` golden
- [x] `resolve_zero_budget` golden
- [x] Ordering tie-break golden
- [x] End-to-end pipeline golden

### MCP layer

- [x] Success response golden
- [x] Error object goldens
- [x] `context.list_caches` golden
- [x] `context.inspect_cache` golden

### Failure injection

- [x] Missing cache → correct error
- [x] Corrupt manifest → correct error
- [x] Permission denied → `io_error`

Golden failures block release.

---

## 3️⃣ MCP Protocol Surface Lock

### Transport

- [x] stdio JSON-RPC 2.0 only (v0 decision frozen)

### Methods

- [x] `context.resolve`
- [x] `context.list_caches`
- [x] `context.inspect_cache`

### Error schema

- [x] JSON Schema published
- [x] All responses conform
- [x] No undocumented fields
- [x] Stable error codes:
  - [x] `cache_missing`
  - [x] `cache_invalid`
  - [x] `invalid_query`
  - [x] `invalid_budget`
  - [x] `io_error`
  - [x] `internal_error`

### Protocol guarantees

- [x] Response order stable
- [x] Deterministic error messages
- [x] No hidden fields
- [x] No environment-dependent output

---

## 4️⃣ CLI Surface (v0 minimal contract)

### Required commands

- [x] `context build`
- [x] `context resolve`
- [x] Deterministic stdout JSON
- [x] Non-zero exit on failure
- [x] stderr only for human diagnostics

### Optional but recommended

- [x] `context inspect`
- [x] `context serve` explicitly deferred or implemented

CLI must be script-safe.

---

## 5️⃣ Documentation Alignment Pass

All specs must agree on:

- [x] Parameter names (cache, not context_id)
- [x] No provenance in v0
- [x] MCP is part of open core
- [x] Error taxonomy definitions
- [x] stdio transport rationale
- [x] Determinism guarantee statement

If two docs conflict → v0 spec wins.

---

## 6️⃣ Security & Safety Baseline

For stdio MCP server:

- [x] No network listener
- [x] No path traversal outside cache root
- [x] Reject non-UTF8 content
- [x] No execution of document content
- [x] Panic-safe request handling
- [x] All IO errors mapped to safe error objects

Auth explicitly documented as:

Out of scope for stdio v0.

---

## 7️⃣ Performance Guardrails

Not optimization — stability.

- [x] Memory bounded by cache size
- [x] No unbounded allocations from input
- [x] Handles empty cache gracefully
- [x] Handles large documents
- [x] Deterministic sort under load

Optional but nice:

- [ ] Selection latency measured
- [ ] Large cache smoke test

---

## 8️⃣ Packaging & Distribution

### Crates

- [x] `context-core` published or versioned
- [x] `mcp-context-server` version tagged

### Versioning

- [x] Semantic version: 0.1.0
- [x] Protocol version constant in code
- [x] CHANGELOG created

### Artifacts

- [x] Reproducible build
- [x] Release binaries (optional but strong signal)

---

## 9️⃣ Compatibility Contract Statement

Publish this explicitly:

- [x] Context v0 guarantees deterministic selection, stable JSON output, and a frozen MCP interface. Any change to output shape or ordering is a breaking change.

---

## 🔟 Release Gate (final decision check)

You are ready to tag v0 when:

- [x] All golden tests green
- [x] Spec and implementation aligned
- [x] Error schema frozen
- [x] Determinism proven
- [x] No TODOs affecting output
- [ ] You would trust this in CI

If yes → tag and ship.
