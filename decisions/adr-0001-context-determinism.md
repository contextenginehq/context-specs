# ADR-0001: Context Determinism

## Status

Accepted

## Context

AI systems require predictable, reproducible context to enable debugging, auditing, and compliance. Non-deterministic context selection makes it impossible to explain or reproduce agent behavior.

## Decision

All context operations prioritize determinism over adaptability.

## Determinism Guarantees

Given:

- Same documents
- Same versions
- Same configuration

The system **MUST** produce identical context outputs.

## Consequences

### Positive

- Context selection is auditable
- Agent behavior is reproducible
- Debugging is tractable

### Negative

- No runtime adaptation to improve relevance
- Configuration changes required for behavior changes

## Compliance

Any feature that would introduce non-determinism must be:

1. Explicitly opt-in
2. Clearly documented
3. Isolated from core selection logic
