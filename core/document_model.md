# Document Model

## What Is a Document?

A document is an immutable unit of content with a stable identity.

```
Document {
  id:       string    // stable identifier, derived from source path
  version:  string    // content hash
  source:   string    // origin path (e.g., "docs/deployment.md")
  content:  string    // raw content, UTF-8
  metadata: object    // extracted or provided key-value pairs
}
```

A document is the atomic unit. It cannot be partially included or split.

This constraint applies at the document model level. Higher layers MAY derive secondary representations, but the canonical document remains atomic.

---

## Identity: When Are Two Documents "The Same"?

Two documents are the same if and only if they have the same `id`.

```
same_document(a, b) := a.id == b.id
```

The `id` MUST be derived from the normalized source path relative to a single ingestion root.
Multiple roots are not supported in v0.

Normalization rules:
- Relative to ingestion root
- Forward slashes only
- Lowercase
- No leading `./`

Example: `./Docs/Deployment.md` → `docs/deployment.md`

---

## Versioning: How Is Version Computed?

Version is a SHA-256 hash of the content only.

```
version := "sha256:" + hex(sha256(content))
```

Content Handling:
- Content is hashed exactly as ingested after UTF-8 decoding.
- No newline normalization or trimming is performed.
- Byte-for-byte fidelity is preserved.

Not included in version computation:
- Metadata
- Source path
- File timestamps
- Filesystem attributes

Two files with identical content have identical versions, regardless of where they live.

---

## Metadata: What Is Allowed?

Metadata is a flat key-value map. String keys, string or number values only.

```json
{
  "title": "Deployment Guide",
  "section": "infrastructure",
  "priority": 1
}
```

### Extracted Metadata (automatic, post-v0)

> v0 does not perform automatic metadata extraction. The metadata map contains only explicitly provided entries. The fields below will be extracted in a future version.

| Key | Source |
|-----|--------|
| `title` | First H1 heading, if present |
| `byte_size` | Content length in bytes |
| `line_count` | Number of lines |

### Provided Metadata (explicit, post-v0)

> v0 does not parse frontmatter. Content is ingested as-is, including any frontmatter delimiters. Future versions will parse YAML frontmatter as described below.

Frontmatter YAML is parsed if present:

```markdown
---
section: infrastructure
priority: 1
---
# Deployment Guide
```

### Metadata Rules

1. Metadata does **not** affect version
2. Metadata does **not** affect identity
3. Metadata **may** affect selection (via policy, later)
4. Unknown keys are preserved, not rejected
5. **Conflict Resolution**: Provided metadata overrides extracted metadata for the same key.

---

## Invariants

1. **Immutability**: A document is never modified. Changes create new versions.
2. **Content-addressed versioning**: Same content → same version, always.
3. **Stable identity**: Same source path → same id, always.
4. **UTF-8 only**: Content must be valid UTF-8. Invalid bytes are rejected, not cleaned.
   - Validation occurs at ingestion time.
   - Invalid documents MUST fail ingestion and MUST NOT produce partial artifacts.

---

## Serialization

Documents are serialized as JSON:

```json
{
  "id": "docs/deployment.md",
  "version": "sha256:a1b2c3d4e5f6...",
  "source": "docs/deployment.md",
  "content": "# Deployment\n\nThis guide...",
  "metadata": {
    "title": "Deployment",
    "byte_size": 1842,
    "line_count": 47
  }
}
```

Field order is fixed for determinism: `id`, `version`, `source`, `content`, `metadata`.

---

## What This Spec Forbids

- Mutable documents
- Version based on anything other than content hash
- Nested or complex metadata values
- Binary content
- Documents without content (empty string is allowed)
