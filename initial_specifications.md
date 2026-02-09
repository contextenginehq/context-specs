# Product Vision

## Vision

Build a deterministic, auditable, and scalable **Context Layer** for AI systems that allows organizations to turn documentation and structured knowledge into reliable, bounded context for local and remote agents.

The platform is designed to:

* Reduce hallucinations
* Enforce knowledge boundaries
* Enable on‑prem and air‑gapped deployments
* Integrate natively with agent systems via MCP

## Target Users

* Engineering teams deploying LLMs internally
* Regulated industries (EU-first)
* Platform / infra teams

## Non‑Goals

* We are not a chatbot
* We are not a vector database replacement
* We do not train foundation models
* We do not optimize prompts dynamically at runtime

---

# Open‑Core Strategy

## Open Components

* Context ingestion
* Deterministic context selection
* Context cache lifecycle
* MCP server
* CLI and SDKs

## Paid Components

* Context observability
* Drift detection
* Cost attribution
* Usage analytics
* Policy intelligence

The open core must be fully functional without paid components.

---

# Core Concepts

## Document Model

A **Document** is an immutable, versioned unit of knowledge.

### Properties

* id (stable)
* version (hash-based)
* source
* content
* metadata

Documents are never mutated; updates create new versions.

---

## Context Cache

A **Context Cache** is a compiled, queryable snapshot of documents optimized for agent consumption.

### Guarantees

* Deterministic output
* Explicit invalidation
* Rebuild reproducibility

### Lifecycle

1. Ingest documents
2. Normalize and parse
3. Build cache
4. Serve via MCP

---

## Context Selection

Context selection determines which documents are included in a response.

### Constraints

* Token budget
* Relevance score
* Policy filters

Selection must be:

* Deterministic
* Explainable
* Replayable

---

## Summarization & Compression

Summarization is optional and offline.

Rules:

* Never lossy by default
* Source references must be preserved
* Output must be traceable to original documents

---

# MCP Interface Specification

## Purpose

Expose context resolution to agents via the Model Context Protocol.

## Core Methods

* list_contexts
* resolve_context
* inspect_cache

## resolve_context

Inputs:

* context_id
* query
* budget

Outputs:

* ordered document bundle
* metadata
* provenance

---

# CLI Specification

## Commands

* ingest
* build
* serve
* inspect

CLI must:

* Be scriptable
* Emit JSON by default
* Support CI usage

---

# Security Model

## Principles

* No outbound calls by default
* Explicit trust boundaries
* Read-only runtime

## Deployment Modes

* Local
* On‑prem
* Air‑gapped

---

# Determinism Guarantees

Given:

* Same documents
* Same versions
* Same configuration

The system MUST produce identical context outputs.

---

# Observability (Paid)

## Signals

* Context hit rate
* Cache freshness
* Token utilization

## Constraints

Observability must not alter context selection behavior.

---

# Licensing

* Open core licensed under Apache 2.0 or MPL
* Paid components proprietary
* Specs publicly readable

---

# Architectural Decision Records

## ADR‑0001: Determinism First

All context operations prioritize determinism over adaptability.

## ADR‑0002: MCP as Primary Interface

MCP is the first‑class integration surface for agents.

## ADR‑0003: Open Core Boundary

No paid logic may be required for correctness.
