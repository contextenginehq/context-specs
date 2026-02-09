# Spec Review — Shortcomings and Implementation Plans

> Review date: 2026-02-08
> Scope: All specs in context-specs/ analyzed against the context-core and mcp-context-server implementations.

---

## 1. INVARIANT.md

### Shortcomings

**S1.1 — Forbids floating point comparison but implementation uses f32.**
The "What This Forbids" section lists "Floating point comparison in ranking." The implementation scores documents as `f32` and sorts via `partial_cmp`. The invariant holds because identical inputs produce identical f32 values and identical comparison results on the same platform — but the spec language is misleading. The intent is to forbid *non-deterministic* float comparisons (e.g., comparing floats from different sources or across platforms), not all float comparisons.

**S1.2 — Cross-platform determinism is unstated.**
The spec says "always the same result" without scoping it to a platform. IEEE 754 f32 division can produce different results on different architectures (x87 extended precision vs SSE, ARM vs x86). The implementation's simple `term_matches as f32 / total_words as f32` is likely safe, but the spec should make the platform scope explicit.

**S1.3 — JSON serialization of floats is not addressed.**
`serde_json` serializes `f32` deterministically within a single version, but different JSON libraries may serialize the same float differently (e.g., `0.5` vs `5e-1`). The spec should state that serialization determinism is required within a given implementation version, not across implementations.

### Implementation Plan

No code changes. Spec clarifications only:

1. Replace "Floating point comparison in ranking" with "Non-deterministic comparison in ranking (e.g., unordered float operations, random tie-breaking)"
2. Add scope: "Determinism is guaranteed within the same implementation version on the same platform. Cross-platform byte-identical output is a goal but not guaranteed for floating-point values."
3. Add: "JSON serialization MUST produce identical bytes for identical inputs within the same implementation."

---

## 2. milestone_zero.md

### Shortcomings

**S2.1 — Output contract contradicts normative specs.**
The example output uses `"metadata"` as the top-level key, but the normative `context.resolve.md` and `context_selection.md` use `"selection"`. The example also includes `cache_version` and omits `tokens`, `why`, and `documents_excluded_by_budget`. This creates confusion about which spec is authoritative.

**S2.2 — Claims "JSON output includes provenance" but provenance is excluded from v0.**
The "What This Proves" table says `Auditable | JSON output includes provenance`. This contradicts the explicit v0 provenance exclusion in `context.resolve.md` and `mcp_interface.md`.

**S2.3 — `ingest` + `build` are separate steps but implementation is single-step.**
The spec shows `context ingest` then `context build` as separate commands. The implementation's `CacheBuilder::build()` takes documents directly — there is no separate ingestion stage that persists state. The two-step model implies an intermediate ingestion store that doesn't exist.

**S2.4 — Missing `verify` command.**
`context_cache.md` specifies `context verify --cache ./cache` but milestone_zero.md doesn't list it.

### Implementation Plan

Spec updates only:

1. Replace the output contract example to match `context.resolve.md` exactly (use `"selection"` key, include `tokens`, `why`, `documents_excluded_by_budget`)
2. Change "JSON output includes provenance" → "JSON output includes version and scoring explanation"
3. Decide: Is `ingest` a separate command, or does `build` include ingestion? If single-step, update to `context build --sources ./docs --cache ./cache`. If two-step, spec the intermediate ingestion format.
4. Add `verify` to the command list, or fold verification into `inspect`.

---

## 3. document_model.md

### Shortcomings

**S3.1 — Automatic metadata extraction not scoped to a version.**
The spec lists `title`, `byte_size`, `line_count` as "Extracted Metadata (automatic)" without stating whether this is v0 or post-v0. The implementation doesn't extract any of these.

**S3.2 — Frontmatter parsing not scoped to a version.**
"Frontmatter YAML is parsed if present" is stated as a current requirement, not a future one. No YAML parser exists in the implementation.

**S3.3 — Malformed frontmatter behavior undefined.**
What happens when `---` delimiters exist but the YAML between them is invalid? Error? Ignore? Treat as content?

**S3.4 — Frontmatter interaction with content hash undefined.**
If frontmatter is parsed and stripped from content, the hash changes. If frontmatter remains in content, the extracted metadata is redundant with what's in the content field. The spec doesn't address this.

**S3.5 — No maximum document size.**
A 1 GB markdown file would be ingested without limits. Should there be a maximum, or is this explicitly unbounded?

**S3.6 — Serialization field order relies on implementation detail.**
"Field order is fixed for determinism" is enforced by serde's struct field declaration order, which is an implementation detail. Reordering struct fields would silently break the spec. No test enforces this.

### Implementation Plan

Two options depending on v0 scope decision:

**Option A: Defer metadata extraction and frontmatter to post-v0**

1. Add version scoping to the metadata section: "v0: No automatic metadata extraction. `byte_size` and `line_count` will be extracted in a future version."
2. Add version scoping to frontmatter: "v0: Frontmatter is not parsed. Content is ingested as-is. Future versions will parse YAML frontmatter."
3. Add to "What This Spec Forbids": "Metadata extraction or content transformation that changes the content hash (v0)"

**Option B: Implement for v0**

1. Add `byte_size` and `line_count` extraction to `Document::ingest()` (trivial — computed from content post-UTF8-validation)
2. Add `title` extraction: scan for first `# ` line after optional frontmatter
3. Add YAML frontmatter parsing to `parser.rs`:
   - Add `yaml-rust` or `serde_yaml` dependency
   - Parse content between first `---\n` and second `---\n`
   - Merge parsed key-value pairs into metadata (provided overrides extracted per spec rule 5)
   - Define: invalid YAML between delimiters → treat entire content as body (no error)
   - Define: frontmatter is NOT stripped from content (content hash includes frontmatter)
4. Add golden test asserting document JSON field order: `id`, `version`, `source`, `content`, `metadata`

---

## 4. context_cache.md

### Shortcomings

**S4.1 — Index format doesn't match implementation.**
The spec shows `index.json` as a flat map: `{"docs/deployment.md": "documents/a1b2c3.json"}`. The implementation's `CacheIndex { entries: BTreeMap<DocumentId, String> }` serializes as `{"entries": {"docs/...": "..."}}` — a nested object with an `entries` wrapper key. This is a spec violation.

**S4.2 — Duplicate document IDs not addressed.**
If two source files normalize to the same ID (e.g., `docs/Deploy.md` and `docs/deploy.md` on a case-insensitive filesystem), what happens? The spec says IDs are unique but doesn't specify the error case.

**S4.3 — `context verify` not in CLI spec.**
This spec references `context verify --cache ./cache` but the CLI spec (`cli_spec.md`) lists `ingest`, `build`, `serve`, `inspect` — no `verify`.

**S4.4 — Build reads "all `.md` files" — markdown-only?**
The Build section says "Reads all `.md` files from sources." This implies markdown-only ingestion, but the document model doesn't restrict content type. Is this a v0 constraint, or illustrative?

**S4.5 — Orphan file handling underspecified.**
Verification checks for orphan files in `documents/` but doesn't specify what to do about them. Is it an error? A warning? What code maps to it — `cache_invalid`?

**S4.6 — No document version verification on load.**
The verification spec says to check "Every document file hash matches its filename" but the implementation's `load_documents()` only checks document ID, not version/hash.

### Implementation Plan

1. **Fix index.json format** (code change in `context-core`):
   - Add `#[serde(transparent)]` to `CacheIndex` struct, OR
   - Change `CacheBuilder::build()` to write `&index.entries` directly instead of `&index`
   - Add a golden test asserting flat map format
   - Verify existing tests still pass

2. **Add duplicate ID detection** (code change in `context-core`):
   - In `CacheBuilder::build()`, after sorting by ID, check adjacent entries for duplicate IDs
   - Return `CacheBuildError::DuplicateDocumentId(String)` if found

3. **Add document version verification to `load_documents()`** (code change in `context-core`):
   - After loading each document, recompute `DocumentVersion::from_content(doc.content.as_bytes())`
   - Compare against manifest entry's version
   - Return error on mismatch

4. **Resolve `verify` vs `inspect` in CLI spec** (spec change):
   - Either add `verify` to CLI spec, or fold verification into `inspect` with a `--verify` flag

5. **Add v0 scope note for file types** (spec change):
   - "v0 ingests `.md` files only. Future versions may support additional formats."

6. **Spec orphan handling** (spec change):
   - "Orphan files in `documents/` MUST cause verification to return `cache_invalid`."

---

## 5. context_selection.md

### Shortcomings

**S5.1 — Two inconsistent output schemas.**
The "Output" section shows: `query`, `budget`, `tokens_used`, `documents_selected`, `documents_considered`.
The "Explainability" section shows: `documents_considered`, `documents_selected`, `documents_excluded_by_score`, `documents_excluded_by_budget`.
The normative `context.resolve.md` shows: `query`, `budget`, `tokens_used`, `documents_considered`, `documents_selected`, `documents_excluded_by_budget`.
These three are all different.

**S5.2 — `documents_excluded_by_score` contradicts v0 behavior.**
v0 doesn't exclude by score ("Documents with score 0.0 MAY be selected"). This field would always be 0. It's premature — belongs in a future version with score thresholds.

**S5.3 — Empty query behavior contradicts MCP server.**
`context_selection.md`: "If `query_terms` is empty after normalization, all documents receive score 0.0."
`context_resolve.md` (CLI): "Empty query is allowed."
MCP server implementation: rejects empty/whitespace queries with `invalid_query`.
These need to agree.

**S5.4 — "Floating point comparison in ranking" forbidden is misleading.**
Same as S1.1. The spec forbids what the implementation necessarily does. The intent is to forbid non-deterministic comparison, not float comparison per se.

**S5.5 — "Word" is not defined precisely.**
`split(lowercase(content), whitespace)` — what is whitespace? ASCII space/tab/newline? Unicode whitespace categories? Rust's `split_whitespace()` uses Unicode, which means exotic whitespace characters (NBSP, ideographic space, etc.) split words differently than ASCII-only.

### Implementation Plan

1. **Align the output schema** (spec change):
   - Update the "Output" section to match `context.resolve.md` exactly (the normative spec)
   - Remove `documents_excluded_by_score` from the explainability section
   - Add a note: "Future versions MAY add `documents_excluded_by_score` when score-based filtering is introduced"

2. **Resolve empty query policy** (spec change + possible code change):
   - Decision needed: should empty queries be allowed or rejected?
   - If allowed: update MCP server to remove the empty query rejection, return all documents at score 0.0
   - If rejected: update `context_selection.md` and `context_resolve.md` to say empty queries are invalid

3. **Define "word"** (spec change):
   - Add: "Whitespace is defined as Unicode whitespace (Unicode category `Zs`, plus `\t`, `\n`, `\r`, `\x0C`). This matches Rust's `str::split_whitespace()` and Python's `str.split()`."

---

## 6. context.resolve.md (MCP tool spec)

### Shortcomings

**S6.1 — "No environment variables affect resolution" is inaccurate.**
`CONTEXT_CACHE_ROOT` affects cache path resolution in the MCP server. The intent is that env vars don't affect the *selection algorithm*, but the statement is too broad.

**S6.2 — CLI parity claim references a CLI that doesn't exist.**
"The returned JSON MUST be byte-identical to `context resolve --cache C --query Q --budget B`" — but no CLI binary exists. The CLI spec describes it, but it hasn't been built.

**S6.3 — No spec for `list_caches` or `inspect_cache`.**
These are listed in `mcp_interface.md` as core tools but have no normative specification document.

### Resolution

1. **S6.1** — Spec clarification still pending (low priority, wording issue only).
2. **S6.2 — Resolved.** CLI now exists (`context-cli` crate). Both CLI and MCP serialize `context-core::SelectionResult` via `serde_json` — parity by construction.
3. **S6.3 — Resolved.** Normative specs exist at `core/mcp/context.list_caches.md` and `core/mcp/context.inspect_cache.md`. Both tools are implemented in `mcp-context-server`.

---

## 7. error_schema.md

### Shortcomings

**S7.1 — Not all failure modes are mapped to error codes.**
Unspecified mappings:
- Timeout during resolution → currently `internal_error`, should this be a distinct code?
- Very large budget causing OOM → unspecified
- Cache name containing path traversal → currently `cache_missing`, reasonable but not documented
- Message size exceeding limit → currently returns JSON-RPC parse error, not an MCP error

**S7.2 — No guidance on when `internal_error` should fire.**
The code says "Implementation bug or invariant violation" but the MCP server also uses it for timeouts and task join failures. These are operational failures, not bugs.

### Implementation Plan

1. **Document edge case mappings** (spec change):
   - Add a section "Edge Case Mappings" with a table:
     - Timeout → `internal_error` (server operational failure)
     - Path traversal attempt → `cache_missing` (treated as nonexistent)
     - Oversized message → JSON-RPC parse error (protocol level, not MCP level)

2. **Consider adding `timeout` error code for post-v0** (spec note):
   - "Future versions MAY add a `timeout` error code for long-running operations."

---

## 8. context_resolve.md (CLI spec)

### Shortcomings

**S8.1 — CLI doesn't exist.**
This is a detailed spec for a `context resolve` command that hasn't been built. It's the most detailed CLI spec but the CLI crate doesn't exist.

**S8.2 — Empty query is allowed here but rejected by MCP server.**
"Empty query is allowed (results in score 0.0 for all documents)" contradicts the MCP server's `invalid_query` response for empty queries.

**S8.3 — Error output format underspecified.**
Shows a freeform text example but doesn't define whether errors are structured JSON or plain text on stderr.

**S8.4 — `--format pretty` is default but determinism testing requires compact.**
If `pretty` is default, `diff a.json b.json` in the success criteria depends on pretty-printing being deterministic. This is true for serde_json, but the spec should note it.

### Resolution — ALL RESOLVED

1. **S8.1 — CLI built.** `context-cli` crate implements `context build`, `context resolve`, `context inspect`. Binary name `context`. Frozen exit codes 0-7. 12 integration tests. `--format` defaults to `json` (compact).
2. **S8.2 — Resolved.** MCP server's empty query rejection was removed (see must-fix S5.3). Both CLI and MCP now allow empty queries.
3. **S8.3 — Implemented.** Errors are plain text on stderr, prefixed with `error:`. Tested in `tests/exit_codes.rs`.
4. **S8.4 — Resolved by design.** CLI defaults to `--format json` (compact), not `pretty`. Both formats produce deterministic output via serde_json.

---

## 9. mcp_interface.md

### Shortcomings

**S9.1 — `list_caches` and `inspect_cache` have zero specification.**
Listed as "Core Tools" but have no input schema, output schema, error handling, or behavioral spec. The implementation stubs return `[]` and `{"status":"ok"}`.

**S9.2 — Very minimal.**
54 lines. Mostly a pointer to `context.resolve.md`. Doesn't add much value on its own.

### Implementation Plan

1. **Write `list_caches` spec** (new section or document):
   - Input: none (or optional `root` parameter)
   - Output: `[{"name": "string", "cache_version": "string", "document_count": integer}]`
   - Behavior: enumerate subdirectories of cache root that contain valid `manifest.json`
   - Errors: `io_error` if cache root is not accessible

2. **Write `inspect_cache` spec** (new section or document):
   - Input: `{"cache": "string"}`
   - Output: `{"cache_version": "string", "document_count": integer, "build_config": {...}, "created_at": "string", "valid": boolean}`
   - Behavior: load manifest, optionally run verification checks
   - Errors: `cache_missing`, `cache_invalid`, `io_error`

3. **Implement handlers** in `mcp-context-server` once specs are written

---

## 10. cli_spec.md

### Shortcomings

**S10.1 — Extremely thin (38 lines).**
Lists commands and exit codes but doesn't specify argument formats, behavior, or output for any command except exit codes.

**S10.2 — Missing `resolve` command.**
`context resolve` is the primary query interface but isn't listed. The list shows `ingest`, `build`, `serve`, `inspect`.

**S10.3 — Missing `verify` command.**
Referenced in `context_cache.md` but not here.

**S10.4 — Exit code 1 undefined.**
Codes go 0, 2, 3, 4, 5, 6, 7. What is exit code 1? General error? Argument parse failure?

**S10.5 — No argument specifications.**
`ingest`, `build`, `serve`, `inspect` have no argument formats defined.

### Implementation Plan

Expand the spec:

1. **Add `resolve` to command list** with link to `core/cli/context_resolve.md`
2. **Define exit code 1**: "Argument parsing or usage error"
3. **Add argument specs for each command**:
   - `ingest --sources <path> --cache <path>` — read documents from sources into cache staging
   - `build --sources <path> --cache <path> [--force]` — build cache from source documents
   - `resolve --cache <path> --query <string> --budget <integer> [--format json|pretty]`
   - `inspect --cache <path> [--verify]` — show cache metadata, optionally run integrity checks
   - `serve [--cache-root <path>]` — start MCP stdio server
4. **Decide whether `ingest` and `build` are separate or merged** (see S2.3)

---

## 11. summarization_and_compression.md

### Shortcomings

**S11.1 — Not scoped to any version.**
Says nothing about whether this is v0, post-v0, or aspirational. The implementation has only placeholder files.

**S11.2 — Contradicts content-addressed versioning.**
"Compression reduces token usage" via "Whitespace normalization, Structural simplification, Redundancy elimination" — all of these change the content, which changes the document version hash. The spec doesn't address how compression interacts with the immutable document model.

**S11.3 — Extremely thin (26 lines).**
States rules and principles but nothing actionable or testable.

### Implementation Plan

Spec rewrite:

1. **Add version scope**: "Summarization and compression are not part of v0. The following defines requirements for future versions."
2. **Address versioning interaction**: "Compressed/summarized content is a derived representation stored alongside the original. The canonical document version is always based on the original content. Derived representations carry their own version identifier."
3. **Define the model**: Original document → derived representations (summary, compressed). Selection can use derived representations for budgeting but must reference the original document's ID and version.

---

## 12. security_model.md

### Shortcomings

**S12.1 — No mention of input validation or path traversal protection.**
The MCP server implements path traversal protection and message size limits, but the security model doesn't specify these requirements.

**S12.2 — No mention of resource limits.**
Timeouts, message size limits, and budget caps are security-relevant but not addressed.

### Implementation Plan

Expand the spec:

1. Add "Input Validation" section:
   - "Cache identifiers MUST be validated against path traversal attacks"
   - "Message size MUST be bounded to prevent resource exhaustion"
   - "Operation timeouts MUST be enforced to prevent denial of service"
2. Add "Resource Limits" section:
   - "Implementations SHOULD enforce configurable timeouts on resolution operations"
   - "Implementations SHOULD enforce maximum message sizes"

---

## 13. http_api.md

### Shortcomings

**S13.1 — Stub only.** "To be specified." This is fine for v0 (HTTP is out of scope).

### Implementation Plan

None for v0. Add a note: "HTTP API is out of scope for v0. See milestone_zero.md."

---

## Specs with no shortcomings requiring action

The following specs are adequate for their purpose and scope:

- **agents.md** — Meta instructions for AI agents. Well-written.
- **vision.md** — Product vision. Appropriate level of detail.
- **open_core_strategy.md** — Clear boundary definition.
- **non_goals.md** — Concise and useful.
- **licensing.md** — Adequate.
- **self_hosting.md** — Adequate.
- **adr-0001-context-determinism.md** — Clear decision record.
- **adr-0002-open-core-split.md** — Clear decision record.
- **adr-0003-mcp-first.md** — Clear decision record.
- **context_observability.md** — Paid component, out of v0 scope.
- **metrics_and_signals.md** — Paid component, out of v0 scope.
- **policy_engine.md** — Paid component, out of v0 scope.

---

## Priority Summary

### Must fix for v0 correctness — ALL RESOLVED

| # | Issue | Type | Resolution |
|---|---|---|---|
| S4.1 | `index.json` format doesn't match spec (wrapped in `entries` key) | Code bug | Added `#[serde(transparent)]` to `CacheIndex` |
| S5.1 | Three inconsistent output schemas across specs | Spec fix | Aligned `context_selection.md` output/explainability sections to match `context.resolve.md` |
| S5.3 | Empty query: allowed in CLI spec, rejected by MCP server | Code fix | Removed empty query rejection from MCP server `resolve_context.rs` |
| S2.1 | milestone_zero output contract contradicts normative specs | Spec fix | Replaced `"metadata"` with `"selection"`, aligned all fields to `context.resolve.md` |
| S2.2 | "Provenance" claim in milestone_zero contradicts v0 exclusion | Spec fix | Changed to "version and scoring explanation" |

### Should fix for v0 completeness — ALL RESOLVED

| # | Issue | Type | Resolution |
|---|---|---|---|
| S4.2 | Duplicate document ID detection missing | Code | Added `DuplicateDocumentId` error + adjacent-pair check after sort in `CacheBuilder::build()` |
| S4.6 | Document version not verified on load | Code | Added version recomputation and comparison in `ContextCache::load_documents()` |
| S3.1 | Metadata extraction not version-scoped | Spec fix | Scoped "Extracted Metadata" to post-v0 in `document_model.md` |
| S3.2 | Frontmatter parsing not version-scoped | Spec fix | Scoped "Provided Metadata" frontmatter parsing to post-v0 in `document_model.md` |
| S9.1 | `list_caches` and `inspect_cache` have no spec | New spec | Added full specs (inputs, outputs, errors, behavior) to `mcp_interface.md` |
| S10.1 | CLI spec is too thin | Spec expansion | Rewrote `cli_spec.md`: added `resolve`, argument specs for all commands, exit code 1, merged `ingest`/`build`, error output format |
| S12.1 | Security model missing input validation | Spec expansion | Added "Input Validation" and "Resource Limits" sections to `security_model.md` |

### Invariant additions — RESOLVED

| # | Invariant | Spec | Change |
|---|---|---|---|
| I1 | Document ID uniqueness within a cache | `context_cache.md` | Added invariant #6, added to Build rules, added to Forbids |
| I2 | Build idempotency clarification (`created_at` excluded) | `context_cache.md` | Expanded Cache Version section with `created_at` exclusion note |
| I3 | Index-manifest consistency | `context_cache.md` | Added invariant #7, added requirement to Index section |
| I4 | Scoring purity (elevated to invariant) | `context_selection.md` | Added invariant #6 |
| I5 | Token counting purity | `context_selection.md` | Added invariant #7 |
| I6 | Query normalization determinism | `context_selection.md` | Added invariant #8 |
| I7 | Empty document set validity | `context_cache.md` | Added to Build rules |

### Implementation completions — RESOLVED

| # | Issue | Resolution |
|---|---|---|
| S8.1 | CLI doesn't exist | Built `context-cli` crate: `context build`, `context resolve`, `context inspect`. Binary name `context`. Frozen exit codes 0-7. 12 integration tests (5 determinism, 7 exit codes). All passing. |
| S6.2 | CLI parity claim references nonexistent CLI | CLI now exists. Both CLI (`context resolve`) and MCP (`context.resolve`) serialize `context-core::SelectionResult` via `serde_json` — parity by construction. |
| S9.1+ | `list_caches`/`inspect_cache` stubs | Both tools fully implemented in `mcp-context-server`. `context.list_caches`: enumerates cache root subdirs, checks manifest existence, sorted by path. `context.inspect_cache`: path traversal protection, manifest parsing, total_bytes, valid flag. 12 handler tests. `tools/list` advertises all 3 tools. |

### Post-v0

| # | Issue | Type | Effort |
|---|---|---|---|
| S11.1 | Summarization spec needs version scoping and model | Spec rewrite | Medium |
| S1.1 | Floating point language in INVARIANT.md | Spec clarification | Small |
