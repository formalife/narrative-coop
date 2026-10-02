# ADR-003 — PostgreSQL Persistence Baseline

**Status:** ACCEPTED  
**Date:** 2026-10-02

## Context

The MVP needs transactions, relational metadata, append-only canonical history, synchronous current-state projection, idempotency records and a transactional outbox. Introducing a specialized event store or multiple stateful databases now would add infrastructure before demonstrated need.

## Decision

Use PostgreSQL as the primary persistence baseline for the MVP architecture.

PostgreSQL will store:
- Session metadata;
- canonical append-only Session events;
- current synchronous SessionState projection;
- command/idempotency records;
- ActionSubmissions;
- ResolutionRecords;
- scenario/version metadata;
- transactional outbox;
- other ordinary relational product data as appropriate.

Schema-validated JSON/JSONB may be used where component/scenario flexibility materially helps, but not as an excuse for untyped arbitrary state.

No specialized event-store product is required initially.

## Alternatives Considered

1. Specialized event store.
2. Multiple databases by bounded context.
3. PostgreSQL primary store.

## Why Rejected

The alternatives increase infrastructure/operational cost before the two-player Session workload demonstrates a need.

## Consequences

Atomic commit of event batch + critical projection + outbox can remain in one database transaction.

The resolver/domain model must remain independent of PostgreSQL-specific APIs.

## Risks

Poor table/index design could make replay or append workloads inefficient.

## Revisit Conditions

Revisit if measured throughput, retention, replay/query requirements or operational complexity show a specialized event store or additional datastore would materially simplify the system.
