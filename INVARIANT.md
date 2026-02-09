# The Invariant

> **Given the same documents and configuration, context resolution produces identical output. Always.**

This is not a feature. This is the foundation.

---

## The Commitment

Every operation in this system upholds one guarantee:

```
f(documents, config, query) → deterministic output
```

- Same documents
- Same versions
- Same configuration
- Same query

**Always the same result.**

No randomness. No drift. No "it depends."

---

## Why This Matters

| Problem | How determinism solves it |
|---------|---------------------------|
| "Why did the agent say that?" | Replay the exact context it received |
| "Did something change?" | Hash the output, compare |
| "Can we audit this?" | Yes. Every response is reproducible |
| "Will it work air-gapped?" | No external calls means no variance |

---

## What This Forbids

- Random sampling in selection
- Time-based ordering without explicit config
- External API calls during resolution
- Non-deterministic tie-breaking
- Floating point comparison in ranking
- Undefined iteration order over collections

---

## What This Enables

- **Debugging**: Reproduce any context, anytime
- **Compliance**: Prove what context was used
- **Testing**: Assert exact outputs, not approximations
- **Caching**: If inputs match, output matches
- **Trust**: The system does what it says

---

## The Test

```bash
# Run twice, diff must be empty
context resolve --cache ./cache --query "topic" > a.txt
context resolve --cache ./cache --query "topic" > b.txt
diff a.txt b.txt
```

If this ever fails without config or document changes, we have a bug. Not a quirk. A bug.

---

## Enforcement

1. **No randomness in core paths** — random seeds are bugs
2. **Sorted iteration everywhere** — maps, sets, file listings
3. **Hash-based versioning** — content determines identity
4. **Reproducible builds** — same source → same cache
5. **Test on every commit** — determinism regression = broken build

---

## This Is The Moat

Most systems optimize for "good enough."

We optimize for "exactly the same."

That's the product.
