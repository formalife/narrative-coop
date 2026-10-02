# ADR-017 — Edge Function API and Durable Outbox Wakeup Model

**Status:** PROPOSED  
**Date:** 2026-10-02

## Context

The MVP needs an authoritative HTTP command API and reliable execution of post-commit work such as presentation generation and realtime invalidation.

Canonical correctness already depends on PostgreSQL transactions and the accepted engine Outbox. Runtime infrastructure must not introduce a second queue/authority.

Supabase Edge Functions are attractive for a small TypeScript backend but impose CPU/wall-time constraints and are not a durable queue by themselves.

## Decision

### Authoritative API

Use one routed Supabase Edge Function `api` initially for authenticated command/view endpoints.

It:

- verifies Supabase JWT;
- resolves AuthSubject -> ParticipantRef server-side;
- creates EngineCommand Principal server-side;
- invokes provider-independent application/domain services;
- executes authoritative PostgreSQL transactions directly.

### Database driver

Use a direct PostgreSQL driver suitable for the Supabase transaction pooler.

Initial preference:
- Deno Postgres, optionally under Kysely.

Do not use PostgREST/supabase-js as the canonical multi-row transaction engine.

Do not freeze the query-builder until an integration spike proves explicit transactions, row locking, deferred constraints, accepted lock ordering, and fencing through the deployed transaction pooler.

### Outbox worker

Use a separate `outbox-drain` Edge Function.

The durable authority is `engine.outbox`.

Two wakeup paths are allowed:

1. best-effort immediate `EdgeRuntime.waitUntil` request to the separate worker after relevant API work;
2. periodic pg_cron/pg_net recovery sweep, initially proposed every 10 seconds.

Either wakeup may fail without losing work.

The API and worker use separate database roles. The API does not receive worker database credentials. Worker invocation uses a narrow internal invocation credential that is distinct from database credentials and broad platform-admin credentials.

### Concurrency

Multiple drains are safe because accepted Outbox lease-generation fencing selects authoritative completion.

### No second queue

Do not add Supabase Queues/pgmq for canonical continuation work in MVP.

## Alternatives Considered

1. Edge Functions + PostgreSQL Outbox.
2. Persistent Node worker/service.
3. Supabase Queues/pgmq replacing Outbox.
4. Fire-and-forget background task only.

## Why Rejected

### Persistent Node service

Adds another deployed service/provider before measured need.

### pgmq replacement

Duplicates accepted Outbox semantics and migration/retry state.

### Background task only

Not durable enough: runtime shutdown may interrupt work.

## Consequences

Benefits:
- minimal service count;
- TypeScript server runtime;
- correctness remains in PostgreSQL/outbox;
- worker failures are recoverable.

Costs:
- Edge CPU/wall limits;
- connection-pooler constraints;
- cron/wakeup monitoring required.

## Risks

- 2s CPU ceiling for Edge Functions;
- pooler/driver transaction incompatibility;
- missed cron wakeup causing latency;
- duplicate external provider calls after ambiguous failures;
- worker invocation authorization mistakes.

## Revisit Conditions

Move API/worker runtime to a persistent service if:
- representative p99 resolver CPU persistently exceeds 500ms;
- Edge CPU/wall limits materially constrain scenarios;
- DB connection/pooler instability persists;
- outbox latency/SLO requires a continuously running worker.

A runtime move must not change canonical contracts or schema semantics.
