# Context Selection

## What Is Selection?

Selection answers: which documents, in what order, within what budget?

```
selection = select(cache, query, budget)
```

Output is an ordered list of documents that fit within the token budget.

---

## Inputs

| Input | Type | Description |
|-------|------|-------------|
| `cache` | Cache | The compiled document cache |
| `query` | string | The search query |
| `budget` | integer | Maximum tokens allowed |

---

## Output

```json
{
  "documents": [
    {
      "id": "docs/deployment.md",
      "version": "sha256:...",
      "content": "...",
      "score": 0.92,
      "tokens": 847,
      "why": {
        "query_terms": ["deployment"],
        "term_matches": 12,
        "total_words": 156
      }
    }
  ],
  "selection": {
    "query": "deployment",
    "budget": 4000,
    "tokens_used": 3241,
    "documents_considered": 42,
    "documents_selected": 3,
    "documents_excluded_by_budget": 9
  }
}
```

## Execution Model

Selection proceeds in three strict phases:

1. **Scoring Phase**: Every document is scored independently against the query.
2. **Ordering Phase**: Documents are globally sorted by score (descending) and identity (ascending).
3. **Budgeting Phase**: Documents are greedily selected from the sorted list until budget is exhausted.

---

## Relevance: How Is It Computed?

### v0: Term Frequency

v0 uses simple term matching. No embeddings, no ML.

```
score(doc, query) := term_frequency(query_terms, doc.content)
```

Where:

```
query_terms := lowercase(split(query, whitespace))
total_words := len(split(lowercase(content), whitespace))
term_frequency := count(occurrences of any query term) / total_words
```

If `query_terms` is empty after normalization, all documents receive score 0.0.

This is intentionally naive. The algorithm will improve, but the interface won't change.

### Scoring Rules

1. Score is a float between 0.0 and 1.0
2. Higher score = more relevant
3. Score of 0.0 = no query terms found
4. Scoring function is pure (no side effects, no randomness)

---

## Ordering: How Is Order Decided?

Documents are ordered by:

1. **Score descending** (highest first)
2. **Document ID ascending** (alphabetical tie-breaker)

The tie-breaker is critical for determinism. Without it, equal-scored documents could appear in any order.

```python
sorted(documents, key=lambda d: (-d.score, d.id))
```

---

## Budget: How Is It Enforced?

### Token Counting

v0 uses a simple approximation:

```
tokens(content) := ceil(len(content) / 4)
```

Token count depends only on content, not metadata or ID.
This approximates GPT-style tokenization. Exact tokenization is deferred.

### Filling the Budget

Documents are added in order until the next document would exceed budget:

```python
selected = []
used = 0
for doc in ordered_documents:
    doc_tokens = tokens(doc.content)
    if used + doc_tokens <= budget:
        selected.append(doc)
        used += doc_tokens
    # Documents that don't fit are skipped, not truncated
```

### Budget Rules

1. Documents are never truncated
2. A document either fits entirely or is excluded
3. Remaining budget may be unused (no padding)
4. Budget of 0 returns empty selection
5. Documents with score 0.0 MAY be selected if budget allows

---

## Explainability: How Do We Know Why?

Every selected document includes enough information to explain itself:

- `score`: the computed relevance score
- `tokens`: the token count used for budgeting
- `why`: the scoring components (`query_terms`, `term_matches`, `total_words`)

Selection metadata shows aggregate counts: `documents_considered`, `documents_selected`, `documents_excluded_by_budget`.

### Explainability Rules

1. Every selected document includes its score and token count
2. The `why` field shows how the score was computed
3. Selection metadata shows what was excluded and why
4. No black boxes: the selection is fully reconstructible from the output

Future versions MAY add `documents_excluded_by_score` when score-based filtering is introduced. v0 does not exclude by score.

---

## Determinism: The Test

```bash
# Run twice
context resolve --cache ./cache --query "deployment" --budget 4000 > a.json
context resolve --cache ./cache --query "deployment" --budget 4000 > b.json

# Must be byte-identical
diff a.json b.json  # Empty output
```

If the diff ever produces output, the implementation is broken.

---

## Invariants

1. **Deterministic**: Same cache + query + budget → same result
2. **Ordered**: Output order is fully specified
3. **Bounded**: Token usage never exceeds budget
4. **Explainable**: Selection can be reconstructed from output
5. **Complete**: All selected documents included in full
6. **Pure scoring**: Score is a pure function of document content and query. No side effects, no randomness, no external state.
7. **Pure token counting**: Token count is a pure function of document content. Not affected by metadata, document ID, or query.
8. **Deterministic normalization**: Query normalization (lowercase + whitespace split) produces identical term lists for identical input strings.

---

## What This Spec Forbids

- Random sampling
- Probabilistic scoring
- Document truncation
- Undefined ordering
- Hidden relevance factors
- Budget approximation that can exceed limit
- Selection that depends on time or external state
