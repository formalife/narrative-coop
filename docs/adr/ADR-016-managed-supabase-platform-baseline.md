# ADR-016 — Managed Supabase / PostgreSQL 17 Platform Baseline

**Status:** PROPOSED  
**Date:** 2026-10-02

## Context

The accepted architecture requires PostgreSQL, minimal guest authentication, private realtime invalidation and a small server-side TypeScript runtime.

The MVP should minimize infrastructure while keeping canonical mechanics/provider-independent SQL portable.

## Decision

Use managed Supabase as the MVP platform baseline for:

- PostgreSQL;
- Auth;
- Realtime;
- Edge Functions;
- managed database pooling;
- Supabase-specific cron/Vault glue where needed.

Target PostgreSQL major:
- **17**.

Before executable migration sign-off:
- query and record actual `show server_version`;
- run the accepted DDL/integration suite on the actual project.

### Repository boundary

Provider-neutral:

```
db/migrations/
```

Supabase-specific:

```
platform/supabase/
```

Supabase-specific SQL/config may cover:
- Realtime policies;
- pg_cron/pg_net wakeups;
- Vault integration;
- provider role/bootstrap glue.

It may not redefine engine canonical semantics.

### Internal schemas

Do not expose:
- `engine`;
- `access`.

Browser roles get no direct USAGE/table grants.

### Realtime

Use private Broadcast for invalidation only.

Do not use engine table Postgres Changes as the gameplay read model.

### Portability

Domain/resolver packages do not import Supabase SDKs.

Database schema is ordinary PostgreSQL.

## Alternatives Considered

1. Managed Supabase.
2. Separate PostgreSQL + auth + persistent Node service.
3. Cloudflare-centric stateful backend.
4. Self-hosted Supabase.

## Why Rejected

### Separate providers

Adds integration/secrets/operations before measured need.

### Cloudflare-centric state

Would introduce a second coordination/state model before PostgreSQL proves insufficient.

### Self-host

Adds operational burden without present requirement.

## Consequences

Benefits:
- one operational platform for DB/Auth/Realtime/API;
- anonymous Auth support;
- private realtime authorization;
- PG17 target;
- local/dev tooling.

Costs:
- provider glue;
- Edge/pooler constraints;
- platform upgrade monitoring.

## Risks

- vendor outage;
- accidental schema exposure;
- Edge runtime limits;
- provider role limitations;
- major upgrade behavior.

## Revisit Conditions

Revisit if:
- Supabase limits block representative scenarios;
- compliance/data-residency needs change;
- persistent backend becomes simpler;
- costs materially diverge;
- Supabase constraints conflict with accepted PostgreSQL invariants.
