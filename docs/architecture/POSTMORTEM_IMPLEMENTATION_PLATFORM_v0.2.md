# POSTMORTEM — Implementation Platform Baseline v0.2

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/architecture/IMPLEMENTATION_PLATFORM_v0.2.md`  
**Reviewed ADRs:** ADR-015, ADR-016, ADR-017 — PROPOSED  
**Verdict:** MODIFY

## Executive verdict

v0.2 fixed the identity/access/wakeup issues found in v0.1, but current Supabase Edge Function security defaults reveal a structural conflict with the accepted PostgreSQL Schema v0.5 least-privilege model.

The problem is not Supabase PostgreSQL, Auth or Realtime.

The problem is using Supabase Edge Functions as the authoritative engine API/worker runtime.

Current Supabase documentation states that hosted Edge Functions receive by default:

- `SUPABASE_DB_URL`;
- `SUPABASE_SECRET_KEYS`;
- legacy `SUPABASE_SERVICE_ROLE_KEY`;
- project JWKS/publishable values.

Supabase secret keys bypass RLS.

Production custom secrets cannot use the reserved `SUPABASE_` prefix, and current public documentation does not expose a per-function mechanism to suppress these injected default project credentials.

Sources:
- https://supabase.com/docs/guides/functions/secrets
- https://supabase.com/docs/guides/getting-started/migrating-to-new-api-keys
- https://supabase.com/docs/guides/getting-started/api-keys

This means that even if application code voluntarily uses a restricted `engine_runtime_login`, the function process still has access to broader Supabase project credentials.

That materially weakens the accepted privilege separation:

- engine_runtime;
- engine_worker;
- migration owner;
- provider-admin authority.

**Decision from postmortem:** preserve Supabase for PostgreSQL/Auth/Realtime, but move authoritative API and Outbox worker to a persistent external runtime where secrets are scoped per service.

Railway is the current proposed runtime host because its services support service-scoped/sealed variables and persistent API/worker processes. This remains a PROPOSAL until v0.3 red-team/acceptance.

No accepted ADR-001..014, Domain Contract, Domain Model, Persistence Model or PostgreSQL Schema needs supersession.

---

# 1. Finding — Edge default secret blast radius

## v0.2 intent

- `api` gets runtime DB credential only;
- `outbox-drain` gets worker DB credential + provider secrets;
- wake credential remains narrow.

## Actual hosted Edge environment

Current Supabase Edge docs list injected defaults including:

```
SUPABASE_DB_URL
SUPABASE_PUBLISHABLE_KEYS
SUPABASE_SECRET_KEYS
SUPABASE_JWKS
```

with secret keys documented as RLS-bypassing admin credentials.

## Consequence

The intended service-level least privilege is not actually represented by the process environment.

A code-execution bug or compromised dependency in either Edge Function can potentially access broader project credentials than its declared DB role.

## Correction

Do not use Supabase Edge Functions for authoritative engine API/worker.

Use a host where:

- API secret scope contains runtime DB credential only;
- worker secret scope contains worker DB credential + presentation provider keys only;
- no platform automatically injects Supabase admin/database-superuser credentials.

---

# 2. Supabase remains the data/auth/realtime platform

Keep:

- managed Supabase PostgreSQL 17;
- Supabase Auth;
- anonymous guest auth;
- publishable browser key;
- Supabase Realtime private Broadcast;
- provider-specific Realtime authorization policy.

Do not use:

- Edge Functions for canonical API/worker;
- Supabase secret/service-role key in normal runtime;
- Data API for canonical multi-row transactions.

This is a narrower Supabase dependency, not a provider abandonment.

---

# 3. Proposed persistent runtime

Use two persistent services from the same monorepo:

## `engine-api`

Responsibilities:
- public HTTPS API;
- verify Supabase user JWT;
- resolve AuthSubject -> ParticipantRef;
- build server-bound EngineCommand Principal;
- run authoritative PostgreSQL transactions;
- return PlayerInteractionView/presentation-safe data.

Secrets:
- `engine_runtime_login` database URL only;
- public Supabase project/JWKS metadata;
- no worker database password;
- no Supabase secret/admin key;
- no LLM provider key by default.

## `engine-worker`

Responsibilities:
- continuously claim `engine.outbox`;
- BUILD_PRESENTATION;
- DELIVER_PRESENTATION;
- emit private Realtime invalidations;
- secondary noncanonical work.

Secrets:
- `engine_worker_login` database URL;
- LLM/media provider keys needed by worker.

No canonical Session mutation privilege.

---

# 4. Railway fit

Current Railway documentation supports:

- long-running API services;
- separate long-running worker services from the same repository;
- service-scoped variables;
- sealed variables whose values cannot be retrieved through UI/API;
- custom start commands;
- restart policies;
- healthcheck-gated API deploys.

Sources:
- https://docs.railway.com/overview/the-basics
- https://docs.railway.com/variables
- https://docs.railway.com/guides/cron-workers-queues
- https://docs.railway.com/deployments/start-command
- https://docs.railway.com/guides/roll-back-bad-deploy

Railway cron is not relevant because the worker can run continuously and the project Outbox remains durable.

---

# 5. PostgreSQL connectivity improves with persistent runtime

Supabase documentation recommends direct connections for long-running containers/VMs and shared session pooler as the IPv4-compatible alternative.

Custom LOGIN roles work through direct connections and shared Supavisor pooler.

Sources:
- https://supabase.com/docs/guides/database/connecting-to-postgres
- https://supabase.com/docs/guides/troubleshooting/fatal-password-authentication-failed

## Proposed connection mode

Default integration target:

- Supabase shared pooler **session mode** on port 5432;
- one custom LOGIN role per service;
- bounded application-side pool per Railway service.

Why session mode:
- Railway runtime connectivity need not rely on Supabase direct IPv6 availability/add-on;
- session mode behaves much closer to a normal persistent PostgreSQL connection;
- prepared statements/session-level features are supported.

Direct connection may replace it if deployed networking supports it and integration tests show a material benefit.

Do not use transaction-mode pooler by default for persistent Railway services.

---

# 6. JWT verification no longer needs Supabase server secret

Supabase Auth exposes public JWKS for asymmetric JWT verification.

Current docs recommend verified claims/JWKS instead of trusting unverified session data.

Sources:
- https://supabase.com/docs/guides/auth/jwts
- https://supabase.com/docs/reference/javascript/auth-getclaims
- https://supabase.com/docs/guides/auth/server-side/creating-a-client

## Proposed API auth path

For a new Supabase project:

- use asymmetric signing keys;
- API verifies access-token signature/issuer/audience/expiry using project JWKS;
- extract `sub` as AuthSubject;
- then resolve access binding in PostgreSQL.

No Supabase secret key is required for normal authenticated engine commands.

If live Auth-server verification is later required for a specific high-risk operation, it can use user JWT + publishable key, not a secret/admin key.

---

# 7. Realtime invalidation without Supabase admin secret

Current Supabase Realtime supports database-originated private Broadcast through:

`realtime.send(...)`.

Source:
https://supabase.com/docs/guides/realtime/broadcast

Therefore worker does not need a Supabase secret key simply to send invalidations.

## Proposed provider-specific wrapper

Create a narrow Supabase-specific PostgreSQL function, for example:

```
platform.notify_session_view_changed(session_id uuid)
```

It:
- constructs only minimal invalidation payload;
- calls `realtime.send`;
- always uses private channel;
- has empty search_path / fully qualified refs;
- PUBLIC execute revoked;
- EXECUTE granted only to `engine_worker`.

This function belongs in:
`platform/supabase/sql/`

not provider-neutral `db/migrations/`.

The worker's only Realtime capability is calling this narrow database function.

---

# 8. Worker no longer needs a wake endpoint

A persistent worker continuously claims the accepted Outbox.

Consequences:

Remove from v0.2:
- worker wake token;
- `EdgeRuntime.waitUntil`;
- pg_cron/pg_net worker wakeup;
- Edge worker HTTP endpoint.

Outbox authority remains unchanged.

## Poll strategy

Keep operationally configurable.

Suggested implementation:
- immediately claim next work while backlog exists;
- short idle polling/backoff when empty;
- no correctness dependency on exact interval.

Optional future:
- PostgreSQL LISTEN/NOTIFY as best-effort wake optimization after connection-mode spike.

Do not make LISTEN/NOTIFY durable authority.

---

# 9. Provider count trade-off

v0.2 optimized for one provider.

v0.3 would use:

- Supabase — data/auth/realtime;
- Railway — API/worker compute.

This adds:
- one deployment provider;
- one additional billing/observability boundary.

It removes:
- Edge 2s CPU constraint from authoritative engine work;
- automatic broad Supabase secret injection into runtime;
- cron/wakeup glue;
- transaction-pooler/serverless constraints;
- need to put presentation-provider secrets in Supabase function environment.

For this engine, least privilege + predictable transaction runtime outweigh the modest extra provider count.

---

# 10. Edge CPU limit becomes irrelevant to engine authority

Current Supabase Edge CPU/wall limits remain factual but no longer constrain canonical resolver runtime.

The resolver still must remain bounded and performance-tested.

Do not use removal of the Edge 2s cap as permission for unbounded simulation/LLM work inside canonical commands.

Canonical mechanics remain deterministic and small.

---

# 11. Railway worker durability

A process can crash/restart.

Correctness still comes from:
- PostgreSQL Outbox;
- row claims;
- leases;
- lease-generation fencing;
- idempotent external delivery.

Railway restart policy improves availability but is not a work queue.

This exactly matches accepted persistence semantics.

---

# 12. Service replication

Do not start with complex autoscaling assumptions.

MVP:
- one API service instance;
- one worker service instance.

The accepted database concurrency/fencing model already supports multiple instances later.

Before scaling API replicas, confirm:
- no in-memory Session authority;
- no sticky-session dependency;
- DB connection pool sizing.

Before worker replicas, confirm Outbox contention metrics.

---

# 13. Deployment region

Choose Railway API/worker region near the Supabase database region.

Do not attempt active-active multi-region authoritative API initially.

PostgreSQL is still one canonical persistence region/frontier.

Cross-region deployment before evidence adds latency and complexity without improving the accepted Session model.

---

# 14. Access schema findings from v0.1 remain valid

Preserve v0.2 fixes:

- internal `access.session_principal_bindings`;
- internal `access.session_invites`;
- AuthSubject != ParticipantRef != CharacterEntityId;
- one active AuthSubject <-> one ParticipantRef per Session;
- creator binding + ParticipantBound transition atomic;
- invite one-time hashed token;
- cleanup aware of active bindings;
- no normal browser access to `access`.

---

# 15. Browser model

Browser receives only:

- Supabase project URL;
- Supabase publishable key;
- public engine API URL.

Browser:
- signs in anonymously or permanently through Supabase Auth;
- subscribes to authorized private Realtime topic;
- sends Supabase access token to engine API;
- never receives DB credentials or Supabase secret keys.

---

# 16. Provider-specific migration boundary

Keep:

```
db/migrations/
```

for provider-neutral engine/access PostgreSQL structures.

Keep:

```
platform/supabase/sql/
```

for:
- Realtime authorization helper/policies;
- narrow Realtime-send wrapper;
- Supabase-specific grants/hooks.

Railway deployment config lives under:
```
platform/railway/
```

No runtime provider owns canonical schema semantics.

---

# 17. Runtime repository boundary

Proposed:

```
apps/
  player-web/
  engine-api/
  engine-worker/

packages/
  contracts/
  domain/
  state/
  resolver/
  temporal/
  epistemics/
  projections/
  replay/
  validation/
  persistence-postgres/
  auth-supabase/

db/
  migrations/

platform/
  supabase/
  railway/
```

Rules:
- `domain/resolver` import no Supabase/Railway SDK;
- API/worker depend on application/provider adapters;
- worker and API may share compiled application packages, not secret scope.

---

# 18. Cost/complexity comparison

## Supabase Edge v0.2

Pros:
- one provider;
- simple deployment.

Cons:
- broad injected Supabase secrets in every Edge Function;
- Edge CPU cap;
- serverless DB/pooler restrictions;
- extra wake/cron design;
- worker/API credential separation weakened by environment defaults.

## Supabase + persistent Railway runtime

Pros:
- per-service scoped/sealed secrets;
- custom least-privilege DB logins;
- long-running transaction-friendly runtime;
- continuous Outbox worker;
- no Supabase admin key needed;
- no Edge CPU cap.

Cons:
- second provider;
- additional deployment/monitoring surface.

## Conclusion

For this project's accepted security/concurrency model, persistent separated runtime is the better fit.

---

# 19. Root cause

The v0.2 design optimized too strongly for provider consolidation.

The accepted database privilege model is not cosmetic: it is a security invariant around canonical state.

A runtime platform that automatically injects stronger credentials than the service requires undermines that design even if code promises not to read them.

---

# 20. Verdict

**IMPLEMENTATION_PLATFORM_v0.2 is NOT ready for acceptance.**

Create v0.3 with:

- Supabase PostgreSQL/Auth/Realtime retained;
- Supabase Edge removed from authoritative runtime;
- persistent API + worker services;
- Railway as proposed compute host;
- distinct custom PostgreSQL LOGIN credentials;
- JWKS-based user authentication without Supabase secret key;
- database-based narrow Realtime broadcast capability;
- continuous Outbox worker;
- no pg_cron/Edge wake path.

Update ADR-016/017 accordingly.

No executable migrations or implementation code yet.
