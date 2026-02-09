# Context v0 Compatibility & Breaking Change Policy

## Version Scope

This policy applies to all releases 0.x.y where:

- protocol version = v0
- selection spec = v0
- cache format = v0

v0 guarantees deterministic behavior and stable machine contracts.

---

## What Counts as a Breaking Change

If any item below changes, the release must not be a compatible v0 update.

### 1. Selection Output Contract

Any change to the structure, ordering, or semantics of selection output is breaking.

**Structure changes**

Breaking:

- adding required fields
- removing fields
- renaming fields
- changing field types
- changing nesting structure

Not breaking:

- adding optional fields (must not affect determinism)

**Ordering changes**

Breaking:

- changing document ordering rules
- changing JSON field order
- changing tie-break behavior
- nondeterministic ordering

**Semantics changes**

Breaking:

- different documents selected for same input
- different scores for same input
- different token counts
- different explainability values

Rule: same cache + query + budget must produce identical output.

### 2. Determinism Guarantee

Any change that can cause byte-level output differences is breaking.

Breaking examples:

- timestamp added to output
- float serialization changes
- randomization
- filesystem order dependence
- locale-dependent behavior
- platform-dependent output
- nondeterministic hashing
- unstable JSON formatting

Determinism is a core contract, not an implementation detail.

### 3. MCP Protocol Contract

Breaking:

- removing a method
- renaming a method
- changing parameters
- changing error codes
- changing error object schema
- changing success response structure
- altering transport behavior for stdio mode

Not breaking:

- adding new optional methods
- adding optional response fields

### 4. Cache Format Compatibility

Breaking:

- existing caches cannot be read
- manifest schema changes
- index schema changes
- document addressing changes
- hash interpretation changes

Not breaking:

- additional optional manifest metadata
- new tooling that produces equivalent cache

Rule: a v0 cache must remain readable by all future v0 versions.

### 5. CLI Machine Interface

Breaking:

- JSON output format changes
- exit code semantics change
- stdout/stderr contract changes
- command parameter meaning changes

Not breaking:

- new CLI commands
- new optional flags
- improved human-readable stderr

CLI is treated as a machine interface.

### 6. Error Model

Breaking:

- removing error codes
- renaming error codes
- changing error schema
- changing when a specific error type is emitted

Not breaking:

- improved error messages
- more precise classification within same category

---

## What Is Allowed in v0

These changes are explicitly compatible:

**Internal Improvements**

- performance optimizations
- refactoring
- memory usage improvements
- code cleanup
- better error messages

**Additive Features**

- new MCP tools
- new CLI commands
- optional fields
- new cache metadata
- documentation updates

**Implementation Freedom**

You may change:

- internal data structures
- file layout (if external format identical)
- scoring implementation (only if output identical)

---

## Compatibility Test Rule

A release is compatible with v0 only if:

```
old_binary resolve(...) == new_binary resolve(...)
```

for all test fixtures.

Golden tests are the authority. If golden output changes, it is a breaking change.

---

## Frozen Surfaces in v0

The following are locked:

- selection algorithm semantics
- token counting formula
- ordering rules
- JSON serialization layout
- MCP error schema
- cache manifest structure
- determinism guarantee

These may evolve only in v1.

---

## What Requires v1

Move to v1 when you want to change:

- ranking algorithm behavior
- token counting model
- output schema
- cache format
- protocol shape
- determinism guarantees
- transport model

v1 is where innovation happens. v0 is where trust is built.

---

## Practical Decision Rule

When evaluating a change:

> Could an automated system detect a difference in output?

If yes, it is a breaking change.
