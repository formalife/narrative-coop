# Architecture Decision Records

This directory is the canonical registry for structural technical decisions.

## Status values

- **PROPOSED** — under discussion; not binding.
- **ACCEPTED** — binding current decision.
- **SUPERSEDED** — replaced by a later ADR.
- **DEPRECATED** — historical but no longer recommended/current.
- **REJECTED** — explicitly considered and not selected.

A proposal in ChatGPT or an Architecture Baseline does not become binding until its decision is explicitly accepted and recorded.

## Required ADR structure

Use [ADR-TEMPLATE.md](ADR-TEMPLATE.md).

Each ADR covers:

1. Context
2. Decision
3. Alternatives considered
4. Why rejected
5. Consequences
6. Risks
7. Revisit conditions

## Current candidate ADR set — Baseline v0.3

All candidates remain **PROPOSED**.

| ID | Topic | Status |
|---|---|---|
| ADR-001 | Selective Event Sourcing scope | PROPOSED |
| ADR-002 | Session stream and Single Canonical Frontier | PROPOSED |
| ADR-003 | PostgreSQL persistence baseline | PROPOSED |
| ADR-004 | Componentized Entity Model | PROPOSED |
| ADR-005 | EngineCommand and ActionSubmission authority boundary | PROPOSED |
| ADR-006 | Claims, Interaction Graph and deterministic resolver | PROPOSED |
| ADR-007 | Semantic Domain Event and State Mutation IR | PROPOSED |
| ADR-008 | Epistemic model | PROPOSED |
| ADR-009 | Temporal and Scheduler model | PROPOSED |
| ADR-010 | Compiled Scenario Bundle and pure rules | PROPOSED |
| ADR-011 | Versioning, canonical serialization, hashing and replay | PROPOSED |
| ADR-012 | Scenario Progression vs Narrative Direction/Realization | PROPOSED |
| ADR-013 | Logical CQRS, critical projection and Transactional Outbox | PROPOSED |
| ADR-014 | LLM runtime boundaries | PROPOSED |
| ADR-015 | Guest identity / participation baseline | PROPOSED |

Do not create/accept all ADRs mechanically.

Create the individual ADR when its decision is being explicitly accepted, rejected or superseded.
