# ADR-016 — Managed Supabase / PostgreSQL 17 Platform Baseline

**Status:** PROPOSED  
**Date:** 2026-10-02

## Context

The accepted architecture requires PostgreSQL, guest authentication, reliable private realtime invalidation and server-side TypeScript execution.

The project should minimize infrastructure during MVP without coupling canonical mechanics to a vendor SDK.

## Decision

Use managed Supabase as the MVP platform baseline for:

- PostgreSQL;
- Auth;
- Realtime;
- Edge Functions;
- managed pooler;
- cron/Vault integration where needed.

Target PostgreSQL major: **17**.

Before executable migrations, record and test the project's exact deployed server version.

### Internal schema

Keep authoritative `engine` schema out of Supabase Data API exposed schemas.

Browser roles receive no direct USAGE/table grants.

### Realtime

Use private Realtime Broadcast only as a notification/invalidation channel.

Do not expose canonical engine tables through Postgres Changes as the gameplay read model.

### Portability

Domain/resolver packages remain Supabase-independent.

Provider-specific code is isolated in platform adapters.

Provider-neutral SQL remains canonical under `db/migrations/`.

## Alternatives Considered

1. Supabase managed platform.
2. Separate managed PostgreSQL + auth provider + Node service.
3. Cloudflare-centric stateful architecture.
4. Self-host Supabase.

## Why Rejected

### Separate providers

Adds operations/integration complexity before demonstrated need.

### Cloudflare-centric state

Would compete with the accepted PostgreSQL Single Canonical Frontier and risks premature Durable Objects.

### Self-host Supabase

Adds database/auth/realtime operations burden without current compliance/control requirement.

## Consequences

Benefits:
- fewer providers;
- native anonymous auth;
- private realtime;
- managed PostgreSQL 17;
- local CLI/dev environment.

Costs:
- platform-specific auth/realtime glue;
- serverless runtime/database pool constraints;
- provider upgrade behavior must be monitored.

## Risks

- vendor/platform outage;
- Edge runtime limits;
- provider-specific schema/role restrictions;
- Postgres major upgrades;
- accidental Data API exposure.

## Revisit Conditions

Revisit if:
- Supabase runtime limits become material;
- compliance/data-residency requirements change;
- database/realtime cost/scale materially diverges;
- platform constraints conflict with accepted PostgreSQL schema;
- persistent backend becomes operationally simpler.
