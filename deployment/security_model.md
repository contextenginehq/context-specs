# Security Model

## Principles

- **No outbound calls by default** — System operates without external network access
- **Explicit trust boundaries** — All trust relationships are configured, never inferred
- **Read-only runtime** — Runtime does not modify source documents or configuration

## Deployment Modes

| Mode        | Description                              |
|-------------|------------------------------------------|
| Local       | Single-machine deployment                |
| On-prem     | Organization-managed infrastructure      |
| Air-gapped  | No network connectivity                  |

## Trust Boundaries

- Document sources are explicitly trusted
- Agent connections are authenticated at the transport level
  - **stdio (v0):** Authentication is delegated to the operating system process boundary. The calling process owns stdin/stdout and is implicitly trusted. No application-level auth is required.
  - **HTTP/SSE (future):** Will require explicit authentication. Out of scope for v0.
- No implicit data sharing

## Input Validation

All external inputs MUST be validated before processing:

- **Cache identifiers** MUST be validated against path traversal attacks. Identifiers containing `..`, absolute paths, or symlinks that escape the cache root MUST be rejected.
- **Query strings** are passed through to the selection algorithm without length restriction in v0. Implementations SHOULD consider a maximum query length in future versions.
- **Budget values** MUST be validated as non-negative integers. Negative values MUST be rejected with `invalid_budget`.
- **JSON-RPC messages** MUST be bounded in size to prevent resource exhaustion. Implementations SHOULD reject messages exceeding a configurable maximum (e.g., 1 MB).

## Resource Limits

Implementations MUST enforce operational limits to prevent denial of service:

- **Operation timeouts**: Resolution operations MUST have a configurable timeout. The default SHOULD be reasonable for the expected workload (e.g., 30 seconds).
- **Message size**: Incoming messages MUST be bounded. Messages exceeding the limit SHOULD be rejected at the transport level with a JSON-RPC parse error.
- **Concurrent operations**: Implementations SHOULD limit concurrent resolution operations to prevent resource exhaustion.

## Data Handling

- Documents are stored locally
- No telemetry by default
- Audit logs are local-only unless configured otherwise
