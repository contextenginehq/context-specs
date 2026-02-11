# ADR-0004: Enterprise Document Ingestion

**Status:** Accepted  
**Date:** 2026-02-11  

## Context / Problem

Enterprises store documentation across multiple platforms: Confluence, Jira, SharePoint/Wiki, Google Docs, and local Markdown files.  

The Context platform must:

- Ingest these sources into **context caches**.
- Preserve **determinism**: identical input produces identical output across runs and platforms.
- Maintain **auditability**: provenance traceable back to the source.
- Enable **extensibility**: easy addition of new sources or formats in future versions.

The current system only supports local Markdown files, which is insufficient for enterprise adoption.

---

## Decision

1. **Canonical Markdown/plaintext output**  
   All enterprise connectors must convert source content to canonical Markdown/plaintext before passing to `Document::ingest`.

2. **Connector interface**  
   Each connector implements the `DocumentSource` trait, returning `RawDocument`s:

   ```rust
   pub trait DocumentSource {
       fn fetch_documents(&self, root: &Path) -> Result<Vec<RawDocument>, ConnectorError>;
   }

   pub struct RawDocument {
       pub relative_path: String,
       pub content: Vec<u8>, // canonical UTF-8
       pub metadata: Metadata,
   }
   ```

3. **Canonicalization rules**  
   Connectors must normalize line endings, trim trailing whitespace, flatten rich text, normalize Unicode, strip volatile metadata, and hash attachments instead of embedding raw binaries.

4. **Metadata & provenance**  
   Required fields: `source_type`, `source_id`, `connector_version`. Optional fields: `source_url`, `fetched_at`. Metadata must not affect document identity or selection determinism.

5. **Determinism contract**  
   A connector satisfies determinism if two fetches of the same source content produce identical `RawDocument`s (byte-for-byte) regardless of run, platform, or API response ordering.

6. **v0 ingestion model**
   - Full rebuild only (no incremental ingestion).
   - Connectors must produce complete document sets each run.
   - Incremental ingestion deferred to a future ADR/spec.

---

## Consequences

- **Positive:** Guarantees deterministic and auditable ingestion from multiple enterprise sources.
- **Positive:** Connectors can evolve independently as long as canonical output is stable.
- **Negative:** Incremental ingestion is postponed; full rebuilds may be more resource-intensive initially.
- **Neutral:** All enterprise ingestion logic is separated from core engine logic.

---

## Alternatives Considered

| Alternative | Reason Rejected |
| :--- | :--- |
| Store raw binaries or native format | Harder to canonicalize, non-deterministic |
| Use HTML/XML directly | Selection engine expects plaintext/Markdown |
| Mix content & metadata in hashing | Breaks determinism |

---

## Related Documents

- [enterprise_ingest.md](../core/enterprise_ingest.md)
- [enterprise_ingest_plan.md](../plans/enterprise_ingest_plan.md)
