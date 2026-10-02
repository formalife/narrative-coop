# ADR-016 — Supabase PostgreSQL / Auth / Realtime Platform Baseline

**Status:** PROPOSED  
**Date:** 2026-10-02

## Context

The accepted architecture needs managed PostgreSQL, low-friction guest authentication and private realtime invalidation.

The runtime itself must preserve the accepted database least-privilege model.

Current Supabase hosted Edge Functions receive broad project credentials by default, including database URL and RLS-bypassing secret keys. That makes them a poor fit for the authoritative engine runtime even though Supabase remains a strong fit for data/auth/realtime.

## Decision

Use managed Supabase for:

- PostgreSQL;
- Supabase Auth;
- Supabase Realtime.

Target PostgreSQL major:
- **17**.

Do NOT use Supabase Edge Functions as the authoritative engine API/worker baseline.

### Database

Canonical SQL remains ordinary PostgreSQL under:

`db/migrations/`.

Before migration execution:
- record actual `SHOW server_version`;
- run accepted DDL/integration tests on the target project.

### Authentication

Use Supabase Auth:
- anonymous guest sign-in;
- CAPTCHA/Turnstile;
- later identity linking.

Normal engine API JWT verification uses public Supabase Auth signing metadata/JWKS and does not require a Supabase secret/admin key.

### Realtime

Use private Supabase Broadcast as invalidation only.

Realtime server emission should use a narrow database-side provider wrapper rather than broad Supabase API admin credentials.

### Internal schemas

Do not expose:
- `engine`;
- `access`.

Browser roles receive no direct table privileges.

### Provider-specific glue

Supabase-specific SQL/config lives under:

`platform/supabase/`

and may include:
- Realtime RLS;
- Realtime helper functions;
- provider-specific grants.

It may not redefine canonical domain/persistence semantics.

## Alternatives Considered

1. Supabase PostgreSQL/Auth/Realtime + external compute.
2. Supabase including Edge Functions as authoritative runtime.
3. Separate PostgreSQL/Auth/Realtime providers.
4. self-hosted Supabase.
5. Cloudflare-centric stateful backend.

## Why Rejected

### Supabase Edge authoritative runtime

Hosted function environments receive broader Supabase project credentials than required by the accepted engine runtime/worker DB roles.

The Edge CPU/pooling model also adds constraints without being necessary once persistent compute is used.

### Separate providers for DB/Auth/Realtime

Adds integration/operations without current benefit.

### Self-host

Adds operations burden before demonstrated requirement.

### Cloudflare-centric state

Would add another state/coordination model before PostgreSQL proves insufficient.

## Consequences

Benefits:
- managed PostgreSQL/Auth/Realtime;
- anonymous Auth;
- private realtime;
- canonical SQL portability;
- runtime does not need Supabase admin secret.

Costs:
- compute is a second provider;
- provider-specific Realtime/Auth glue remains;
- cross-provider network latency must be managed by region selection.

## Risks

- Supabase outage affects DB/Auth/Realtime;
- provider schema/role behavior changes;
- Realtime policy mistakes;
- Postgres major/platform upgrades;
- accidental internal-schema exposure.

## Revisit Conditions

Revisit if:
- compliance/data-residency requirements change;
- Supabase costs/limits materially diverge;
- another managed PostgreSQL/Auth platform materially simplifies the stack;
- Supabase constraints conflict with accepted PostgreSQL semantics.
