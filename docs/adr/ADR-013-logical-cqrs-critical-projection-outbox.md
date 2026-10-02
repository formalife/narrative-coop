# ADR-013 — Logical CQRS, Critical Projection and Transactional Outbox

**Status:** ACCEPTED  
**Date:** 2026-10-02

## Context

Gameplay needs a current state for efficient command validation/resolution, while canonical history remains event-sourced. Post-commit work such as presentation delivery/realtime notification must not be lost if the process crashes after committing gameplay state.

## Decision

Use CQRS as a logical separation, initially within PostgreSQL/application modules rather than distributed services.

### Synchronous critical projection

Update atomically with canonical event append:
- current SessionState;
- stream revision;
- state hash;
- critical scheduler/progression state.

### Secondary projections

May lag/rebuild:
- recap;
- analytics;
- aggregate admin/reporting;
- search/reporting views.

### Transactional Outbox

Insert required post-commit work into an outbox in the same transaction as canonical commit.

Initial workers may poll PostgreSQL.

Consumers must be idempotent because delivery is at-least-once in practice.

No Kafka/Redis/message broker is implied.

## Alternatives Considered

1. Fully synchronous everything inside gameplay request.
2. Fire-and-forget post-commit callbacks.
3. Distributed CQRS with broker/read database.
4. Logical CQRS + transactional outbox.

## Why Rejected

Fire-and-forget can lose required continuation work.

Distributed CQRS/brokers add premature infrastructure.

## Consequences

Canonical commit remains atomic while presentation/notifications can retry safely.

Outbox lifecycle/cleanup/observability must be designed.

## Risks

Poor outbox consumer idempotency can create duplicate external side effects.

## Revisit Conditions

Revisit transport architecture only if measured scale/latency/reliability requirements justify a broker or independent read infrastructure.
