# ADR-0002: Open Core Boundary

## Status

Accepted

## Context

The platform follows an open-core business model. Clear boundaries between open and paid components are essential to:

- Maintain trust with the open-source community
- Prevent accidental exposure of proprietary logic
- Ensure the open core is genuinely useful

## Decision

No paid logic may be required for correctness.

## Boundary Definition

### Open (Required for Correctness)

- Document ingestion
- Context selection algorithm
- Cache lifecycle
- MCP interface
- CLI

### Paid (Operational Enhancement)

- Observability and metrics
- Drift detection
- Cost attribution
- Usage analytics
- Policy intelligence

## Consequences

### Positive

- Open core is fully functional standalone
- Clear value proposition for paid features
- Community can contribute to core without IP concerns

### Negative

- Some features must be explicitly excluded from open core
- Requires discipline to maintain boundary

## Enforcement

- Specs explicitly mark features as open or paid
- Code reviews verify boundary compliance
- Automated checks prevent paid imports in open modules
