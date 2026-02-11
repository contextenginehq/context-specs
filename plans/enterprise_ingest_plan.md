# Enterprise Document Ingestion — Implementation Plan

> Phased plan for implementing the enterprise ingestion pipeline defined in [enterprise_ingest.md](context-specs/core/enterprise_ingest.md).

## Overview

The enterprise ingestion feature introduces a connector framework that enables the Context platform to ingest documentation from enterprise sources (Confluence, Jira, SharePoint, Google Docs) while preserving the platform's determinism invariant. All connector output flows through the existing `Document::ingest` → `CacheBuilder` pipeline.

---

## Phase 0 — Foundation (context-core)

**Goal**: Establish the connector trait, `RawDocument` type, and canonicalization utilities in `context-core` so that all connectors share a common interface.

### 0.1 — Connector trait and types

- [ ] Define `DocumentSource` trait in `context-core::document::source`
- [ ] Define `RawDocument` struct (pre-ingestion document)
- [ ] Define `ConnectorError` error type with variants: `AuthenticationFailed`, `FetchFailed`, `InvalidContent`, `PartialFetch`
- [ ] Add `connector_version` to required metadata fields in `Metadata` documentation

### 0.2 — Canonicalization utilities

- [ ] Create `context-core::document::canonicalize` module
- [ ] Implement `normalize_line_endings()` — convert CRLF/CR to LF
- [ ] Implement `trim_trailing_whitespace()` — per-line trailing whitespace removal
- [ ] Implement `remove_trailing_empty_lines()` — strip trailing blank lines
- [ ] Implement `normalize_unicode_nfc()` — Unicode NFC normalization (use `unicode-normalization` crate)
- [ ] Implement `canonicalize(content: &str) -> String` — applies all rules in deterministic order
- [ ] Add determinism tests: identical input → identical output across platforms

### 0.3 — Filesystem connector (reference implementation)

- [ ] Implement `FilesystemSource` as the reference `DocumentSource`
- [ ] Migrate existing `context-cli build` walk logic to use `FilesystemSource`
- [ ] Verify all existing tests still pass (no regression)
- [ ] Add `source_type: "filesystem"` and `connector_version` metadata

### 0.4 — Ingestion pipeline function

- [ ] Create `context-core::document::ingest_pipeline` module
- [ ] Implement `ingest_from_source(source: &dyn DocumentSource, root: &Path) -> Result<Vec<Document>>`
- [ ] Pipeline: `source.fetch_documents()` → validate UTF-8 → `Document::ingest()` per document
- [ ] Error handling: configurable skip-and-warn vs abort-all (via `IngestPolicy` enum)
- [ ] Add integration tests exercising full pipeline

**Dependencies**: None (pure core work)
**Estimated scope**: ~800 lines of code + ~400 lines of tests

---

## Phase 1 — CLI integration (context-cli)

**Goal**: Expose the connector framework through the CLI so users can build caches from enterprise sources.

### 1.1 — Refactor `build` to use connector pipeline

- [ ] Replace direct walkdir logic in `build.rs` with `FilesystemSource` + `ingest_from_source()`
- [ ] Verify all 12 existing CLI tests pass unchanged
- [ ] Verify determinism: old build path vs new build path produce identical caches

### 1.2 — Add `--source-type` flag

- [ ] Add `--source-type` argument to `build` command (default: `filesystem`)
- [ ] Dispatch to appropriate `DocumentSource` based on flag
- [ ] Add exit code mapping for `ConnectorError` variants

### 1.3 — Connector configuration

- [ ] Support `--connector-config <path>` for passing connector-specific settings (JSON file)
- [ ] Define minimal config schema per connector (auth tokens, endpoints, pagination limits)
- [ ] Validate config at CLI startup before starting ingestion

**Dependencies**: Phase 0 complete
**Estimated scope**: ~300 lines of code + ~200 lines of tests

---

## Phase 2 — Enterprise connectors (new crate: `context-connectors`)

**Goal**: Ship production-ready connectors for the most common enterprise documentation sources.

### 2.1 — Crate scaffolding

- [ ] Create `context-connectors` crate with `context-core` dependency
- [ ] Add `Cargo.toml` with `rust-version = "1.74"`, `default-features = false`
- [ ] Feature flags: `confluence`, `jira`, `sharepoint`, `google-docs` (all optional)
- [ ] Each connector behind its own feature flag to minimize dependency surface

### 2.2 — Confluence connector

- [ ] Implement `ConfluenceSource` (`DocumentSource` trait)
- [ ] REST API v2 client with authentication (token-based)
- [ ] Pagination with explicit sorting (by page ID, ascending)
- [ ] HTML → Markdown canonicalization (use `html2md` or equivalent, deterministic config)
- [ ] Strip Confluence macros, user mentions, dynamic content
- [ ] Populate `source_type`, `source_id`, `connector_version`, `source_url` metadata
- [ ] Determinism tests: mock API → verify byte-identical output across runs
- [ ] Integration test with recorded API responses

### 2.3 — Jira connector

- [ ] Implement `JiraSource` (`DocumentSource` trait)
- [ ] REST API client with authentication
- [ ] Pagination with explicit sorting (by issue key, ascending)
- [ ] Issue → Markdown conversion (summary, description, comments)
- [ ] Strip volatile fields (reporter avatar URLs, vote counts, watcher counts)
- [ ] Determinism tests with mock API

### 2.4 — SharePoint/Wiki connector

- [ ] Implement `SharePointSource` (`DocumentSource` trait)
- [ ] Microsoft Graph API client with OAuth2 authentication
- [ ] HTML → Markdown canonicalization
- [ ] Determinism tests with mock API

### 2.5 — Google Docs connector

- [ ] Implement `GoogleDocsSource` (`DocumentSource` trait)
- [ ] Google Drive API client with service account authentication
- [ ] Export as plain text / Markdown
- [ ] Determinism tests with mock API

**Dependencies**: Phase 0 complete (Phase 1 not required — connectors work via library API)
**Estimated scope**: ~500-800 lines per connector + tests

---

## Phase 3 — Verification and compatibility (context-compat)

**Goal**: Extend the compatibility harness to cover enterprise ingestion determinism.

### 3.1 — Connector determinism tests

- [ ] Add golden fixtures: recorded API responses → expected `RawDocument` output
- [ ] Verify canonicalization is byte-identical across platforms
- [ ] Add cross-version regression tests for connector output

### 3.2 — End-to-end pipeline tests

- [ ] Enterprise source (mocked) → connector → build → resolve → verify determinism
- [ ] Verify caches built from enterprise sources pass all existing cache invariant tests
- [ ] Schema validation for connector metadata fields

**Dependencies**: Phases 0-2 complete
**Estimated scope**: ~400 lines of tests + fixtures

---

## Phase 4 — MCP server integration (mcp-context-server)

**Goal**: Expose connector metadata through the MCP server for agent-accessible audit trails.

### 4.1 — Metadata passthrough

- [ ] Ensure `context.resolve` includes connector metadata in `SelectedDocument` output
- [ ] Ensure `context.inspect_cache` reports `source_type` distribution
- [ ] No new tools required — existing tools surface connector metadata naturally

### 4.2 — Documentation

- [ ] Update MCP server README with enterprise ingestion workflow
- [ ] Add example: Confluence → cache → agent query

**Dependencies**: Phase 0 complete (metadata already flows through existing pipeline)
**Estimated scope**: ~100 lines of code + documentation

---

## Priority and Sequencing

```
Phase 0 (Foundation)          ←— START HERE
    ↓
Phase 1 (CLI integration)    Phase 2 (Connectors)     ←— parallel
    ↓                             ↓
Phase 3 (Compatibility)      Phase 4 (MCP)             ←— parallel
```

### Recommended order for first connector

1. Phase 0 (all) — establishes the framework
2. Phase 2.2 (Confluence) — highest enterprise demand
3. Phase 1 (CLI) — makes Confluence usable end-to-end
4. Phase 3 (Compat) — locks down determinism guarantees
5. Phase 2.3-2.5 (remaining connectors) — incremental additions

---

## Risk Register

| Risk | Mitigation |
|------|-----------|
| HTML→Markdown conversion non-determinism | Pin converter version, add golden tests, consider vendoring |
| Unicode normalization edge cases | Use well-tested `unicode-normalization` crate, add ICU test vectors |
| API pagination instability | Sort explicitly by ID, never rely on server-side ordering |
| Connector dependency bloat | Feature flags per connector, all optional |
| Breaking existing v0 caches | Filesystem connector must produce identical output to current `build` |

---

## Success Criteria

- [ ] `FilesystemSource` produces byte-identical caches to current `context build`
- [ ] At least one enterprise connector (Confluence) passes full determinism harness
- [ ] All 69+ existing tests continue to pass (37 core + 12 CLI + 20 MCP)
- [ ] Enterprise-sourced caches are indistinguishable from filesystem-sourced caches at the selection layer
- [ ] Connector metadata is visible in `inspect` and `resolve` output for audit
