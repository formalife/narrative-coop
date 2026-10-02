# ADR-017 — Persistent API / Worker Runtime and Durable Outbox Processing

**Status:** ACCEPTED  
**Date:** 2026-10-02

## Context

The engine needs:

- a public authoritative HTTP API;
- exact PostgreSQL transactions/row locks;
- a continuously available worker for post-commit presentation/realtime work;
- process-level least privilege matching accepted database roles.

The durable work authority is already the accepted PostgreSQL `engine.outbox`.

Hosted Supabase Edge Functions are not selected because their environment receives broader project credentials than each engine service requires.

## Decision

Use two persistent TypeScript services.

### engine-api

Responsibilities:
- verify Supabase user JWT;
- resolve AuthSubject -> ParticipantRef;
- create server-bound EngineCommand Principal;
- run authoritative engine/access PostgreSQL transactions;
- return authorized PlayerInteractionViews.

Credentials:
- `engine_runtime_login` database credential;
- public Supabase JWKS/project metadata.

Normal JWT verification uses public signing metadata and does not require Supabase secret/admin credentials.

No:
- worker DB credential;
- LLM/provider secret;
- Supabase secret/service-role key.

### engine-worker

Responsibilities:
- continuously claim `engine.outbox`;
- BUILD/DELIVER Presentation work;
- emit minimal Realtime invalidations;
- other explicitly noncanonical outbox tasks.

Credentials:
- `engine_worker_login`;
- worker-only LLM/media provider secrets.

No canonical Session event/runtime update privilege.

### Compute provider

Use Railway as the initial persistent compute host.

API and worker are separate Railway services from the same repository with service-scoped secrets.

### PostgreSQL connectivity

Default:
- Supabase shared session pooler (port 5432);
- custom LOGIN role per service;
- small application-side pools;
- Supabase DB SSL enforcement enabled;
- certificate/hostname verification (`verify-full` or driver equivalent).

Direct connection remains an allowed tested optimization.

### Outbox

Worker polls/claims accepted PostgreSQL Outbox continuously.

No Redis, Kafka, pgmq or second durable queue.

Railway restart policy improves availability but does not provide work durability.

### Realtime

Worker emits invalidation through a narrow PostgreSQL Supabase-specific wrapper around `realtime.send`.

No Supabase secret key is required by API/worker for normal engine operation.

## Alternatives Considered

1. Railway persistent API + worker.
2. Supabase Edge API/worker.
3. one combined persistent API/worker service.
4. persistent API plus Redis/queue worker.
5. fire-and-forget/background task runtime.

## Why Rejected

### Supabase Edge

Default broad project credentials weaken process-level least privilege; CPU/pool constraints are unnecessary.

### One combined service

Would expose API process to worker/LLM secrets and couple worker failures to public API.

### Redis/queue

Duplicates the accepted durable PostgreSQL Outbox.

### Fire-and-forget

Not durable.

## Consequences

Benefits:
- service-level secret isolation;
- direct mapping to engine_runtime/engine_worker roles;
- conventional long-running PostgreSQL transaction behavior;
- continuous worker without cron wakeup;
- no Edge CPU ceiling;
- runtime host is replaceable.

Costs:
- second provider;
- another deployment/observability boundary;
- cross-provider network path.

## Risks

- Railway outage;
- cross-provider latency/egress;
- DB connection-pool misconfiguration;
- service-secret misconfiguration;
- worker polling inefficiency;
- accidental API/worker credential sharing.

## Revisit Conditions

Revisit compute provider/runtime if:
- another host offers materially simpler/safer persistent services;
- Railway reliability/cost becomes problematic;
- self-hosting/compliance requirements arise;
- workload justifies a broker or continuously scalable worker fleet;
- measured connection/network behavior favors another deployment topology.

A runtime-host change must not alter accepted canonical contracts/persistence semantics.


---

## Acceptance record

Explicitly accepted on 2026-10-02 as part of Implementation Platform Baseline v0.3.
