# Context Cache

## What Is a Cache?

A cache is a compiled, read-only artifact built from a set of documents.

Think of it as a binary, not a database.

```
cache/
├── manifest.json      # what's in this cache
├── documents/         # individual document files
│   ├── {hash1}.json
│   ├── {hash2}.json
│   └── ...
└── index.json         # lookup structure
```

A cache is **complete**. It contains everything needed to resolve context without accessing original sources.

---

## Inputs: What Goes Into a Cache?

A cache is built from exactly two inputs:

1. **Document set**: All documents to include
2. **Build configuration**: Settings that affect cache structure

```
cache = build(documents, config)
```

### Build Configuration (v0)

```json
{
  "version": "1",
  "hash_algorithm": "sha256"
}
```

v0 has no user-configurable options. Configuration exists for future compatibility.

---

## The Manifest

Every cache has a `manifest.json`:

```json
{
  "cache_version": "sha256:...",
  "build_config": {
    "version": "1",
    "hash_algorithm": "sha256"
  },
  "created_at": "2026-02-05T10:30:00Z",
  "document_count": 42,
  "documents": [
    {
      "id": "docs/deployment.md",
      "version": "sha256:a1b2c3...",
      "file": "documents/a1b2c3.json"
    }
  ]
}
```

The manifest is the authoritative list of documents in the cache.
Files in documents/ without a corresponding manifest entry invalidate the cache.

### Cache Version

The cache version is a hash of:

```
cache_version := sha256(
  build_config_json +
  sorted(document_id + ":" + document_version for each document)
)
```

Same documents + same config → same cache version.

`created_at` is informational only and MUST NOT affect cache version computation. Because `created_at` differs between builds, manifest file bytes are NOT identical across builds of the same inputs — only `cache_version` is. The determinism guarantee applies to cache version and resolution output, not to raw manifest bytes.

---

## Invalidation: What Invalidates a Cache?

A cache is invalid if and only if:

1. **Document added**: New document not in manifest
2. **Document removed**: Document in manifest no longer exists
3. **Document changed**: Document version differs from manifest
4. **Config changed**: Build config differs

A cache is **not** invalidated by:

- Time passing
- Reading from the cache
- Queries made against it
- External events

Invalidation is explicit. The cache does not "expire."

---

## Verification: How Do We Know It's Correct?

### Integrity Check

```bash
context verify --cache ./cache
```

Verification checks:

1. Manifest exists and is valid JSON
2. Cache version matches recomputed hash
3. Every document file exists
4. Every document file hash matches its filename
5. No orphan files in documents/

### Determinism Check

```bash
# Build twice from same sources
context build --sources ./docs --cache ./cache1
context build --sources ./docs --cache ./cache2

# Cache versions must match
jq .cache_version cache1/manifest.json
jq .cache_version cache2/manifest.json
# Must be identical
```

---

## Operations

### Build

```bash
context build --sources ./docs --cache ./cache
```

- Reads all `.md` files from sources
- Computes document versions
- Rejects duplicate document IDs (fatal build error)
- Writes cache atomically (temp dir, then rename)
- Fails if cache dir exists (no implicit overwrite)
- An empty document set produces a valid cache with 0 documents

### Rebuild

```bash
context build --sources ./docs --cache ./cache --force
```

- Same as build, but removes existing cache first

### Inspect

```bash
context inspect --cache ./cache
```

Output:

```json
{
  "cache_version": "sha256:...",
  "document_count": 42,
  "total_bytes": 184729,
  "valid": true
}
```

---

## Storage Format

### Document Files

Each document stored as `documents/{hash}.json`:

```json
{
  "id": "docs/deployment.md",
  "version": "sha256:a1b2c3...",
  "source": "docs/deployment.md",
  "content": "...",
  "metadata": {}
}
```

Filename is first 12 characters of version hash (without `sha256:` prefix).
Filename collisions are treated as fatal build errors.

### Index

`index.json` maps document IDs to files.

`index.json` MUST be serialized with keys sorted lexicographically.

`index.json` and `manifest.json` MUST reference exactly the same set of documents. Any divergence between them invalidates the cache.

```json
{
  "docs/deployment.md": "documents/a1b2c3.json",
  "docs/security.md": "documents/d4e5f6.json"
}
```

---

## Invariants

1. **Deterministic**: Same inputs → same cache version
2. **Complete**: Cache contains all data needed for resolution
3. **Immutable**: Cache is never modified after build
4. **Atomic**: Build either fully succeeds or fully fails
5. **Portable**: Cache can be copied to another machine and used
6. **Unique IDs**: No two documents in a cache may have the same ID. Duplicate IDs are a fatal build error.
7. **Consistent**: `index.json` and `manifest.json` reference exactly the same document set

---

## What This Spec Forbids

- Partial cache builds
- In-place cache modification
- Caches that require source access at query time
- Automatic invalidation or expiration
- Non-filesystem cache storage (for v0)
- Duplicate document IDs within a single build
