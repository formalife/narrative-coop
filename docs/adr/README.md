# Architecture Decision Records

This directory is the canonical registry for structural technical decisions.

## Status values

- **PROPOSED** — under discussion; not binding.
- **ACCEPTED** — binding current decision.
- **SUPERSEDED** — replaced by a later ADR.
- **DEPRECATED** — historical but no longer recommended/current.
- **REJECTED** — explicitly considered and not selected.

## Accepted Phase-0 decisions

| ID | Topic | Status |
|---|---|---|
| [ADR-001](ADR-001-selective-event-sourcing-scope.md) | Selective Event Sourcing scope | ACCEPTED |
| [ADR-002](ADR-002-session-stream-single-canonical-frontier.md) | Session stream and Single Canonical Frontier | ACCEPTED |
| [ADR-003](ADR-003-postgresql-persistence-baseline.md) | PostgreSQL persistence baseline | ACCEPTED |
| [ADR-004](ADR-004-componentized-entity-model.md) | Componentized Entity Model | ACCEPTED |
| [ADR-005](ADR-005-command-action-authority-boundary.md) | EngineCommand and ActionSubmission authority boundary | ACCEPTED |
| [ADR-006](ADR-006-claims-interaction-deterministic-resolver.md) | Claims, Interaction Graph and deterministic resolver | ACCEPTED |
| [ADR-007](ADR-007-semantic-events-mutation-ir.md) | Semantic Domain Event and State Mutation IR | ACCEPTED |
| [ADR-008](ADR-008-epistemic-model.md) | Epistemic model | ACCEPTED |
| [ADR-009](ADR-009-temporal-scheduler-model.md) | Temporal and Scheduler model | ACCEPTED |
| [ADR-010](ADR-010-compiled-scenario-bundle-pure-rules.md) | Compiled Scenario Bundle and pure rules | ACCEPTED |
| [ADR-011](ADR-011-versioning-hashing-replay.md) | Versioning, canonical serialization, hashing and replay | ACCEPTED |
| [ADR-012](ADR-012-scenario-progression-narrative-separation.md) | Scenario Progression vs Narrative Direction/Realization | ACCEPTED |
| [ADR-013](ADR-013-logical-cqrs-critical-projection-outbox.md) | Logical CQRS, critical projection and Transactional Outbox | ACCEPTED |
| [ADR-014](ADR-014-llm-runtime-boundaries.md) | LLM runtime boundaries | ACCEPTED |
| [ADR-015](ADR-015-guest-identity-session-participation.md) | Guest identity and Session participation | PROPOSED |
| [ADR-016](ADR-016-managed-supabase-platform-baseline.md) | Supabase PostgreSQL / Auth / Realtime platform baseline | PROPOSED |
| [ADR-017](ADR-017-persistent-api-worker-outbox.md) | Persistent API / Worker runtime and durable Outbox | PROPOSED |

## Governing baseline

Accepted Phase-0 architecture:
- `../architecture/ARCHITECTURE_BASELINE_v0.3.md`

Acceptance record:
- `../architecture/PHASE_0_ACCEPTANCE.md`

## Required ADR structure

Use [ADR-TEMPLATE.md](ADR-TEMPLATE.md).

Every structural change to an ACCEPTED decision requires:
1. an explicit proposed ADR;
2. alternatives/consequences/risks;
3. explicit acceptance;
4. superseding status on the old ADR where applicable;
5. update of the architecture baseline and PROJECT_STATE.

Do not silently reinterpret accepted ADRs during implementation.
