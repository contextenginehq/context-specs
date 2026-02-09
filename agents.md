# Agent Instructions for context-specs

## 🎯 Repository Purpose

This repository is the **canonical source of truth** for the Context Platform architecture and specifications. It is sacred and must be treated with extreme care.

## ✅ What IS Allowed

- **Specification documents only** — architectural decisions, API contracts, behavioral specs
- **Markdown files** — human-readable documentation
- **Versioning** — specs evolve, but changes are deliberate and tracked
- **Public visibility** — even paid features are documented here (implementation remains proprietary)
- **Discussion via PRs** — debate specs before implementation
- **Architecture Decision Records (ADRs)** — document significant decisions

## 🚫 What is NOT Allowed

- **No runtime code** — no `.ts`, `.js`, `.py`, `.go`, etc.
- **No implementation details** — describe *what* and *why*, not *how*
- **No build artifacts** — no compiled files, binaries, or generated code
- **No environment configs** — no `.env`, `docker-compose.yml`, etc.
- **No package managers** — no `package.json`, `requirements.txt`, `go.mod`, etc.

## 📁 Directory Structure

```
context-specs/
├── product/          # Vision, strategy, and product decisions
├── core/             # Core system architecture and behavior
├── interfaces/       # External API contracts (MCP, HTTP, CLI)
├── ops/              # Observability, metrics, and operational concerns
├── deployment/       # Security, hosting, and licensing model
└── decisions/        # Architecture Decision Records (ADRs)
```

## 🔒 Integrity Rules

1. **Specs are contracts** — Once published, breaking changes require versioning
2. **Open vs Paid is explicit** — Clearly mark which features are OSS vs proprietary
3. **No accidental leaks** — Proprietary implementation logic stays out of specs
4. **Reference, don't duplicate** — Other repos reference these specs, they don't copy them
5. **Debate before build** — Specs are discussed and approved before implementation

## 📝 When Editing Specs

- **Ask "Is this a spec or implementation?"** — If implementation, it doesn't belong here
- **Use ADRs for decisions** — Major architectural choices go in `/decisions`
- **Version breaking changes** — Don't silently change published contracts
- **Keep specs minimal** — Describe behavior, not implementation
- **Think: Would this leak proprietary info?** — If yes, rephrase or remove

## 🤝 For AI Agents

When working in this repository:

- **Never add code files** — Only markdown specifications
- **Never add tooling** — No linters, formatters, or build scripts
- **Ask before breaking changes** — Existing specs may be referenced elsewhere
- **Preserve structure** — Don't reorganize without discussion
- **Keep it declarative** — Focus on *what*, not *how*

## 🎯 This Repo's Job

To ensure that every engineer, contributor, and AI agent knows:

- What Context Platform does
- How its interfaces behave
- Why architectural decisions were made
- Which features are open vs proprietary

**This is the single source of truth. Treat it accordingly.**
