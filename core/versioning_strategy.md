# Context Versioning Strategy

v0 to v1 transition rules.

This governs versioning for:

- cache format
- selection behavior
- CLI machine interface
- MCP protocol
- error schema

Human docs can evolve freely — machine contracts cannot.

---

## Version Dimensions

Context has four versioned surfaces. They move independently.

| Surface | Where version lives | Compatibility rule |
|---------|--------------------|--------------------|
| Cache format | `manifest.json` `build_config.version` | Reader must support version |
| Selection semantics | selection spec version | Must match runtime |
| MCP protocol | protocol version field | Negotiated |
| CLI machine interface | binary version | Must preserve v0 guarantees |

These do NOT need to change at the same time.

---

## Semantic Versioning Model

Two layers of versioning:

### Product Version (Binary Release)

Standard semver:

```
MAJOR.MINOR.PATCH
```

- `PATCH` — bug fixes, no behavior change
- `MINOR` — additive features
- `MAJOR` — breaking change

In v0 phase (`0.MINOR.PATCH`): MINOR may still break behavior internally, but must not violate the v0 compatibility policy.

### Protocol & Format Versions (Frozen Contracts)

These are explicit and machine-readable.

**Cache Format Version**

Stored in manifest:

```json
{
  "build_config": {
    "version": "1"
  }
}
```

Rules:

- Reader must reject unsupported versions
- Writer must emit only one version per release
- Format version changes only on breaking change

**MCP Protocol Version**

Included in initialization:

```json
{
  "protocolVersion": "2024-11-05"
}
```

Rules:

- Major change increments protocol version
- Server may support multiple versions
- Client must declare version

**Selection Spec Version**

Not serialized, but locked to runtime.

Rule: if output changes, selection spec version increments.

---

## Compatibility Matrix

### v0 Runtime Must

- Read all v0 caches
- Produce identical results for identical inputs
- Accept v0 protocol clients
- Preserve CLI JSON format

### v0 Runtime Must NOT

- Read future cache versions
- Silently change selection semantics
- Change error schema
- Introduce nondeterminism

Fail fast > degrade silently.

---

## What Triggers v1

Move to v1 when ANY of these occur:

**Output Behavior Changes**

- ranking algorithm produces different results
- token counting changes
- explainability changes
- ordering changes

**Schema Changes**

- cache manifest changes
- MCP request/response shape changes
- error object schema changes
- CLI JSON changes

**Determinism Contract Changes**

- output no longer byte-stable
- platform independence relaxed

**Architecture Changes**

- cache no longer fully self-contained
- selection depends on external state
- partial document inclusion allowed

If machines must adapt, it is v1.

---

## Migration Model (v0 to v1)

You do NOT upgrade in place. You support parallel versions.

### Cache Handling

Runtime behavior:

```
if cache.version == supported_version:
    proceed
else:
    error: unsupported_cache_version
```

Optional helper:

```bash
context migrate-cache --from v0 --to v1
```

Migration is explicit and offline.

### MCP Version Negotiation

Server behavior:

- if supported: proceed
- else: error `unsupported_protocol_version`

---

## Compatibility Enforcement Mechanism

Every release must pass:

**Golden Selection Tests** — Old binary vs new binary must match output byte-for-byte.

**Cache Roundtrip Test** — v0 cache built previously must load successfully.

**Protocol Schema Validation** — Responses must validate against frozen JSON Schema.

If any fail, it is not a v0-compatible release.

---

## Deprecation Model

v0 does not deprecate machine behavior.

You may deprecate:

- CLI flags
- human-readable output
- documentation patterns

Machine contracts remain stable until v1.

---

## Version Constants in Code

### context-core

```rust
pub const CACHE_FORMAT_VERSION: &str = "1";
pub const SELECTION_SPEC_VERSION: &str = "0";
```

### mcp-context-server

```rust
"protocolVersion": "2024-11-05"
```

---

## Practical Decision Tree

When making a change:

1. Does output change? If yes: v1.
2. Does schema change? If yes: v1.
3. Does only performance change? Safe for v0.
4. Does behavior improve but output is identical? Safe for v0.

When in doubt: run golden tests.

---

## Guiding Principle

Keep v0 stable longer than feels comfortable. Batch all behavior changes into a single v1. Treat v1 as the algorithmic evolution release.

- v0 = trust + adoption
- v1 = intelligence upgrade
