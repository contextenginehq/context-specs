# Enterprise Document Ingestion

> Specification for ingesting enterprise documentation into the Context platform, preserving determinism, auditability, and extensibility.

## 1. Objective

Define a standardized, deterministic process for ingesting enterprise documentation from various sources (Confluence, Jira, Wiki, Google Docs, SharePoint) into Context caches.

The ingestion process must ensure:

- **Determinism**: Identical input → identical output across platforms and runs.
- **Auditability**: Traceable back to the original source, including content hash and metadata.
- **Extensibility**: Easy addition of new document sources or formats in the future.

---

## 2. Supported Sources

| Source          | Input Method                 | Output Format |
|-----------------|------------------------------|---------------|
| Markdown files  | Local filesystem             | Markdown      |
| Confluence      | API / XML / HTML export      | Markdown      |
| Jira            | API / JSON export            | Markdown      |
| SharePoint/Wiki | API / HTML export            | Markdown      |
| Google Docs     | API / Export                 | Markdown      |

> All sources must be converted to **canonical Markdown/plaintext** before being fed into `context build`.

---

## 3. Canonicalization Rules

Enterprise connectors MUST canonicalize content **before** it reaches `Document::ingest`. The core engine hashes content byte-for-byte as received (see [Document Model](document_model.md#versioning-how-is-version-computed)), so canonicalization is the connector's responsibility.

To guarantee deterministic output across connector runs:

- Normalize line endings to LF (`\n`)
- Trim trailing whitespace on all lines
- Remove empty trailing lines
- Normalize Unicode to NFC
- Strip volatile metadata fields (author, last-modified timestamp, view count)
- Flatten rich text elements (tables, embedded images, links) into deterministic Markdown
- For attachments/binaries: store hash references in metadata, do not embed raw content

### 3.1 What This Forbids

- Connectors MUST NOT rely on API response ordering (paginate and sort explicitly)
- Connectors MUST NOT include timestamps, session IDs, or request-specific data in content
- Connectors MUST NOT perform lossy encoding conversions (source must be valid UTF-8 per [Document Model](document_model.md#invariants))

---

## 4. Document Identity & Metadata

### 4.1 Identity

Document identity follows the existing [Document Model](document_model.md#identity-when-are-two-documents-the-same) rules:

- `id` is derived from the **normalized source path** relative to the ingestion root
- `version` is `sha256(canonical_content)` — computed by `Document::ingest`, not by the connector
- Connectors provide the `id` and raw content; the core engine computes the version

### 4.2 Source Metadata

Enterprise connectors populate the document `metadata` map with provenance fields. Per the Document Model spec, metadata is a flat key-value map (string keys, string or number values only).

Required connector metadata fields:

| Field              | Type   | Description                                        |
|--------------------|--------|----------------------------------------------------|
| `source_type`      | string | Connector identifier, e.g. `confluence`, `jira`    |
| `source_id`        | string | Origin system identifier, e.g. page ID, ticket key |
| `connector_version`| string | Version of the connector that produced this output  |

Optional fields (for audit trail only):

| Field              | Type   | Description                                          |
|--------------------|--------|------------------------------------------------------|
| `source_url`       | string | Original URL of the source document                  |
| `fetched_at`       | string | ISO 8601 timestamp of when the connector fetched it  |

### 4.3 Metadata Rules

Per the existing platform invariants:

- Metadata MUST NOT affect version computation
- Metadata MUST NOT affect document identity
- Metadata MUST NOT influence selection scoring (v0)
- `fetched_at` is for audit purposes only and MUST NOT be used in any deterministic path

> Optional metadata is strictly audit/logging only and MUST NOT be used in hash, version, or selection computations.

---

## 5. Connector Specification

### 5.1 Interface

All source connectors implement:

```rust
pub trait DocumentSource {
    /// Fetch and canonicalize all documents from this source.
    /// Returns documents ready for `Document::ingest`.
    fn fetch_documents(&self, root: &Path) -> Result<Vec<RawDocument>, ConnectorError>;
}

/// Pre-ingestion document produced by a connector.
pub struct RawDocument {
    pub relative_path: String,   // becomes the DocumentId via normalization
    pub content: Vec<u8>,        // canonical UTF-8 content
    pub metadata: Metadata,      // connector-provided metadata
}
```

Example canonicalized output:

```json
{
  "relative_path": "team/architecture.md",
  "content": "## Deployment Architecture\nAll services are containerized...\n",
  "metadata": {
    "source_type": "confluence",
    "source_id": "DOC-12345",
    "connector_version": "confluence-connector/1.0.0",
    "source_url": "https://confluence.example.com/pages/DOC-12345",
    "fetched_at": "2026-02-11T12:34:56Z"
  }
}
```

Connector responsibilities:

- API authentication and authorization
- Pagination with deterministic ordering (sort explicitly, never rely on API default order)
- Canonicalization of content (Section 3)
- Population of connector metadata (Section 4.2)
- Error handling and retries (with idempotent retry semantics)

### 5.2 Connector Versioning

- Each connector MUST declare a version string (e.g. `confluence-connector/1.0.0`)
- The connector version is stored per-document in the `connector_version` metadata field
- Connector updates MUST NOT change the canonical output for identical source content
- If a connector's canonicalization logic changes, the connector version MUST be bumped

Connector versions follow semver conventions:

| Change type                              | Version bump |
|------------------------------------------|--------------|
| Bug fix (no output change)               | Patch        |
| New metadata fields or source support    | Minor        |
| Canonicalization logic change            | Major        |

### 5.3 Determinism Contract

A connector satisfies the determinism contract if and only if:

```
fetch(source_state_A) at time T1 == fetch(source_state_A) at time T2
```

Where `source_state_A` represents the same underlying content. The connector MUST strip all time-variant data so that two fetches of the same source content produce byte-identical `RawDocument` output.

---

## 6. Ingestion Pipeline

The full pipeline from enterprise source to context cache:

```
Enterprise Source
    ↓ connector (fetch + canonicalize)
RawDocument[]
    ↓ Document::ingest (validate UTF-8, compute content hash)
Document[]
    ↓ context build (or CacheBuilder API)
Context Cache
    ↓ context inspect (validate integrity)
Verified Artifact
```

### 6.1 Error Handling

| Failure                     | Behavior                                         |
|-----------------------------|--------------------------------------------------|
| Invalid UTF-8 from source   | Connector MUST reject; do not pass to `ingest`   |

All content must be valid UTF-8; connectors MUST NOT attempt to silently repair encoding issues.

| API authentication failure  | Connector returns `ConnectorError`; build aborts  |
| Partial fetch (pagination)  | Connector MUST NOT produce partial document sets  |
| Single document failure     | Configurable: skip-and-warn or abort-all          |

### 6.2 Incremental Ingestion (post-v0)

> v0 does not support incremental ingestion. Every build produces a complete cache from the full document set. Incremental ingestion will be specified in a future version.

When implemented, incremental ingestion must preserve the invariant: the resulting cache must be byte-identical to a full rebuild from the same source state.

---

## 7. Relationship to Existing Specs

| Spec | Relationship |
|------|-------------|
| [Document Model](document_model.md) | Connectors produce documents conforming to this model |
| [Context Cache](context_cache.md) | Connector output feeds into the cache build pipeline |
| [INVARIANT](../INVARIANT.md) | All connector behavior is subject to the determinism invariant |
| [Breaking Change Policy](v0_breaking_change_policy.md) | Connector output changes follow the same compatibility rules |

---

## 8. What This Spec Forbids

- Connectors that embed volatile data (timestamps, view counts, session IDs) in document content
- Connectors that produce different output for identical source content across runs
- Connectors that bypass `Document::ingest` to construct documents directly
- Binary content in document bodies (references via metadata only)
- Connector-specific selection scoring or ranking logic
