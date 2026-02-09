# Summarization & Compression

Summarization is optional and offline.

## Rules

- **Never lossy by default** — Full content is preserved unless explicitly configured otherwise
- **Source references must be preserved** — Summaries link back to original documents
- **Output must be traceable to original documents** — Provenance chain is maintained

## When Summarization Applies

- Pre-computed during cache build (not at query time)
- Configured per document type or source
- Never applied implicitly

## Compression

Compression reduces token usage while preserving semantic content:

- Whitespace normalization
- Structural simplification
- Redundancy elimination

Compression must not alter meaning or remove information.
