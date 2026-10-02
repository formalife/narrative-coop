# Architecture Decision Records

This directory is the canonical registry for structural technical decisions.

## Status values

- **PROPOSED** — under discussion; not binding.
- **ACCEPTED** — binding current decision.
- **SUPERSEDED** — replaced by a later ADR.
- **DEPRECATED** — retained historically but no longer recommended/current.
- **REJECTED** — considered and explicitly not selected.

A proposal discussed in ChatGPT is **not** an accepted decision until its ADR is accepted and the active architecture baseline/project state are updated.

## Required ADR structure

Use [ADR-TEMPLATE.md](ADR-TEMPLATE.md).

Each ADR should cover:

1. Context
2. Decision
3. Alternatives considered
4. Why rejected
5. Consequences
6. Risks
7. Revisit conditions

## Candidate ADRs from Architecture Baseline v0.1

The following are **candidates only**. They are not accepted merely because they appear here.

| ID | Topic | Status |
|---|---|---|
| ADR-001 | Event Sourcing scope | PROPOSED |
| ADR-002 | Session aggregate and single canonical event stream | PROPOSED |
| ADR-003 | Primary persistence / PostgreSQL | PROPOSED |
| ADR-004 | Componentized Entity Model | PROPOSED |
| ADR-005 | Semantic Action Model | PROPOSED |
| ADR-006 | Claim-based conflict resolution | PROPOSED |
| ADR-007 | LLM boundaries | PROPOSED |
| ADR-008 | Fact / Observation / Knowledge / Belief separation | PROPOSED |
| ADR-009 | Temporal model | PROPOSED |
| ADR-010 | Scenario source and compiled bundle | PROPOSED |
| ADR-011 | Version pinning | PROPOSED |
| ADR-012 | Replay and state hashing | PROPOSED |
| ADR-013 | Backend/database baseline | PROPOSED |
| ADR-014 | Logical CQRS | PROPOSED |
| ADR-015 | Guest/anonymous participation model | PROPOSED |
| ADR-016 | Simulation / Director / Realization separation | PROPOSED |

Do not create all ADRs mechanically. Create an ADR when the decision has been sufficiently analyzed to be accepted, rejected, or deliberately recorded.
