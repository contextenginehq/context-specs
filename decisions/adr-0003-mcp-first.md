# ADR-0003: MCP as Primary Interface

## Status

Accepted

## Context

Multiple interface options exist for exposing context to agents:

- Custom HTTP APIs
- gRPC
- Model Context Protocol (MCP)
- Direct library integration

## Decision

MCP is the first-class integration surface for agents.

## Rationale

- MCP is an emerging standard for agent-context interaction
- Native support in major agent frameworks
- Protocol-level guarantees for context handling
- Reduces integration burden for adopters

## Consequences

### Positive

- Immediate compatibility with MCP-aware agents
- Protocol handles transport concerns
- Community alignment with emerging standards

### Negative

- Dependency on MCP ecosystem evolution
- May need fallback for non-MCP systems

## Implementation

- MCP server is part of open core
- HTTP API provided as secondary interface
- CLI wraps MCP operations for scripting
