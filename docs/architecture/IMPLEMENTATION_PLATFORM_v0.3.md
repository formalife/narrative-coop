# IMPLEMENTATION PLATFORM BASELINE v0.3

**Status:** PROPOSED  
**Date:** 2026-10-02  
**Supersedes as working proposal:** IMPLEMENTATION_PLATFORM_v0.2  
**Governing architecture:** Architecture Baseline v0.3 — ACCEPTED  
**Governing persistence:** Persistence Data Model v0.3 — ACCEPTED  
**Governing schema:** PostgreSQL Schema v0.5 — ACCEPTED

---

# 1. Purpose

Freeze the minimum implementation platform needed before executable migrations and repository implementation.

The platform must preserve:

- one PostgreSQL canonical authority;
- accepted DB least privilege;
- AuthSubject / ParticipantRef / Character separation;
- deterministic transaction/locking semantics;
- project Outbox as durable continuation authority;
- private asymmetric player views;
- provider-independent domain/resolver packages.

---

# 2. Proposed platform split

## Supabase

Use managed Supabase for:

- PostgreSQL;
- Supabase Auth;
- Supabase Realtime;
- provider-specific Realtime authorization/broadcast helpers.

Target PostgreSQL major:
- 17.

## Railway

Use Railway for two persistent TypeScript services:

- `engine-api`;
- `engine-worker`.

Railway is compute only.

It does not own:
- canonical state;
- queue truth;
- authentication records;
- Realtime state.

---

# 3. Why this split

Supabase remains a good fit for managed PostgreSQL/Auth/Realtime.

Supabase Edge Functions are not selected as authoritative engine runtime because hosted functions currently receive broad default project credentials including DB URL and RLS-bypassing secret keys.

This weakens the accepted service-specific database privilege model.

A separate persistent runtime permits each process to receive only the credentials it requires.

The additional provider is accepted as a smaller risk than broad canonical-state credential exposure.

---

# 4. PostgreSQL target and connectivity

Use Supabase PostgreSQL 17.

Before executable migration sign-off:

```sql
SHOW server_version;
```

Record actual deployed version.

## Connection mode

Default for Railway services:

- Supabase shared pooler;
- **session mode** on port 5432;
- custom PostgreSQL LOGIN role per service;
- small bounded application-side connection pool.

Reasons:

- persistent services do not need serverless transaction pooling;
- session mode is IPv4-compatible;
- behaves closer to a normal long-lived PostgreSQL connection;
- supports prepared statements/session-level connection behavior if later useful.

Direct PostgreSQL connection remains an allowed optimization if deployed network connectivity supports it and integration tests prefer it.

Do not use transaction-pooler mode by default for these persistent services.

---

# 5. PostgreSQL roles and credentials

Provider-neutral group roles remain:

```
engine_runtime      NOLOGIN
engine_worker       NOLOGIN
engine_publisher    NOLOGIN
engine_ops_readonly NOLOGIN
```

Deployment-specific LOGIN roles:

```
engine_runtime_login LOGIN
  IN ROLE engine_runtime

engine_worker_login LOGIN
  IN ROLE engine_worker
```

Migration owner is separate.

## Secret scope

### engine-api receives

- runtime DB connection string for `engine_runtime_login`;
- Supabase project URL / issuer / JWKS URL;
- public/publishable Supabase metadata if needed.

It does NOT receive:

- worker DB password;
- LLM/media provider secret;
- Supabase secret/service-role key;
- migration-owner credential.

### engine-worker receives

- worker DB connection string for `engine_worker_login`;
- LLM/media provider secrets required by worker tasks.

It does NOT receive:

- runtime DB password;
- migration-owner credential;
- Supabase secret/service-role key.

Railway service variables are scoped independently per service.

Use sealed variables for production secrets where available.

---

# 6. API runtime

`engine-api` is a long-running Node.js/TypeScript HTTP service.

Exact framework remains implementation-local; preferred minimal choices:
- Fastify or Hono/Node-compatible equivalent;
- no heavyweight application framework required.

Exact Node version is not frozen here.

Repository bootstrap pins one supported active-LTS runtime.

## Responsibilities

- HTTPS gameplay API;
- JWT verification;
- AuthSubject -> ParticipantRef lookup;
- invite/session access operations;
- EngineCommand Principal construction;
- canonical/pending-input transactions;
- PlayerInteractionView reads;
- minimal resume metadata.

No LLM narrative generation runs synchronously in normal canonical command transactions.

---

# 7. Supabase JWT verification

Normal API authentication does not require a Supabase secret key.

For a project using asymmetric Auth signing keys:

1. read bearer access token;
2. verify signature against Supabase project JWKS;
3. verify issuer;
4. verify expiry / expected token semantics;
5. extract `sub` as AuthSubject;
6. resolve ACTIVE access binding in PostgreSQL.

Use a maintained JWT library rather than custom cryptography.

## Access revocation

Immediate game-access revocation is controlled by:

`access.session_principal_bindings.status`.

Therefore engine authorization does not require Auth-admin API calls in the command hot path.

Auth JWT expiry/revocation behavior remains an upstream authentication concern.

---

# 8. Guest identity

Preserve proposed ADR-015:

```
AuthSubject != ParticipantRef != CharacterEntityId
```

Guest authentication:
- Supabase anonymous Auth;
- CAPTCHA/Turnstile;
- no mandatory PII.

Operational authorization:
- `access.session_principal_bindings`.

Invites:
- `access.session_invites`.

Neither access table is canonical world state.

---

# 9. Creator/session transaction

Preferred creator transaction:

1. verify AuthSubject;
2. create Session;
3. create SessionGenesis;
4. insert SessionRuntime revision 0;
5. create ParticipantRef;
6. insert ACTIVE access binding;
7. build/commit canonical ParticipantBound transition revision 1+;
8. optionally create invite;
9. materialize required PlayerView/Presentation/Outbox;
10. commit.

No AuthSubject enters canonical state or event hash.

---

# 10. Invite contract

Use one-time high-entropy bearer token.

Requirements:

- 256 random bits;
- URL-safe encoding;
- only cryptographic hash stored;
- token hash unique;
- expiration required;
- one ACTIVE invite per target slot by default;
- terminal CLAIMED / REVOKED / EXPIRED;
- plaintext never logged or sent to analytics.

Prefer raw invite token in SPA URL fragment and scrub immediately after reading.

Claim transaction atomically commits:
- access binding;
- invite terminal state;
- canonical ParticipantBound transition.

---

# 11. Access schema

Internal schemas:

```
engine
access
```

Neither is exposed through browser Data API.

Browser roles receive no table access.

`access` owns operational security state only.

No hard FK to Supabase `auth.users`.

Authorization always begins from a valid verified JWT.

---

# 12. Binding uniqueness

MVP default:

- one ACTIVE AuthSubject -> at most one ParticipantRef per Session;
- one ACTIVE ParticipantRef -> at most one AuthSubject per Session.

These invariants are enforced by partial UNIQUE indexes in the future additive access-schema migration.

A future multi-device/shared-controller mode requires explicit structural review.

Synthetic/internal test players use internal principals rather than normal guest access bindings.

---

# 13. Anonymous account cleanup

Supabase currently has no automatic anonymous-user cleanup.

Cleanup remains deferred until retention is accepted.

Required rule:

> an anonymous AuthSubject with ACTIVE binding to a nonterminal or resumable Session is not cleanup-eligible.

Cleanup affects operational Auth data only.

Canonical Session history is never deleted as a side effect of Auth cleanup.

---

# 14. Worker runtime

`engine-worker` is a continuously running process.

It repeatedly:

1. claims ready Outbox items with accepted `FOR UPDATE SKIP LOCKED` protocol;
2. commits claim/lease generation;
3. performs external work;
4. fences result persistence by expected lease generation;
5. repeats immediately while backlog exists;
6. sleeps/backs off briefly when no work exists.

Exact idle polling/backoff is operational configuration.

No Redis/Kafka/pgmq is introduced.

---

# 15. Worker durability

Railway process uptime is not work durability.

The durable authority remains:

`engine.outbox`.

If worker:
- crashes;
- restarts;
- deploys;
- loses network;

Outbox leases expire and work becomes claimable again.

Restart policies/health monitoring improve availability only.

---

# 16. API/worker service topology

Initial deployment:

- one `engine-api` service instance;
- one `engine-worker` service instance.

No sticky Session state exists in process memory.

Scaling later is already supported by:
- SessionRuntime row locking;
- command fencing;
- Outbox fencing.

Do not autoscale before measuring connection/pool behavior.

---

# 17. Railway deployment properties

Use service-scoped configuration.

API:
- public domain;
- deploy healthcheck endpoint;
- restart on failure.

Worker:
- no public domain required;
- restart on failure;
- service-specific variables.

Keep production/staging environments separate.

Prefer Railway region near Supabase database region.

Do not deploy authoritative engine active-active across regions in MVP.

---

# 18. Realtime invalidation

Use Supabase private Broadcast for notification only.

Payload:

```
{
  "type": "session_view_changed",
  "session_id": "..."
}
```

No StreamRevision is required because PlayerInteractionView can change while canonical revision stays constant.

Never broadcast:
- private view data;
- pending action;
- secret;
- canonical world state;
- narrative output.

Client always refetches current authorized view through `engine-api`.

---

# 19. Realtime send path

The worker does not hold a Supabase secret key.

Use a narrow provider-specific PostgreSQL wrapper around Supabase:

`realtime.send(...)`.

Conceptual function:

```
platform.notify_session_view_changed(session_id uuid)
```

Properties:
- provider-specific Supabase SQL;
- minimal fixed payload;
- private channel only;
- `SECURITY DEFINER`;
- `search_path = ''`;
- fully qualified names;
- PUBLIC EXECUTE revoked;
- EXECUTE granted only to `engine_worker`.

The worker can therefore emit invalidation using its database role without broad Supabase API admin credentials.

---

# 20. Realtime receive authorization

Private topic:

`session:<session_id>`.

Realtime RLS policy on `realtime.messages` uses a narrow access helper based on verified Supabase Auth JWT context.

Conceptual helper:

`access.can_receive_session_topic(auth_subject, session_id) -> boolean`.

Requirements:
- SECURITY DEFINER;
- empty search_path;
- PUBLIC execute revoked;
- boolean result only;
- browser role gets no direct SELECT on access tables.

Normal gameplay clients receive broadcasts; they do not need broadcast-send permission.

---

# 21. Supabase anonymous Auth/browser configuration

Browser receives only:

- Supabase URL;
- Supabase publishable key;
- engine API public URL.

Browser never receives:
- DB credentials;
- Supabase secret key;
- Railway service variables;
- LLM/provider secrets.

Use current publishable key model rather than legacy service-role patterns.

---

# 22. Provider-neutral versus provider-specific SQL

Canonical portable schema:

```
db/migrations/
  engine schema
  access schema
  group roles/grants where portable
```

Supabase glue:

```
platform/supabase/sql/
  realtime authorization policies
  realtime helper functions
  Supabase-specific grants
```

Railway config:

```
platform/railway/
  deployment/service configuration
```

Provider glue cannot redefine canonical domain/persistence semantics.

---

# 23. Repository runtime layout

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
  actions/
  resolver/
  temporal/
  epistemics/
  projections/
  narrative-director/
  narrative-realizer/
  persistence-postgres/
  auth-supabase/
  replay/
  validation/
  testkit/

db/
  migrations/

platform/
  supabase/
  railway/
```

Rules:
- provider SDKs do not enter domain/resolver;
- API and worker share application packages but not deployment secrets;
- worker-only presentation providers remain worker adapters.

---

# 24. PostgreSQL client

Initial preference for persistent Node runtime:

- `pg` / node-postgres;
- small explicit Pool per service;
- Kysely optional above it.

Do not freeze ORM.

Required spike before implementation acceptance:

- transaction commit/rollback;
- `SELECT ... FOR UPDATE`;
- `SKIP LOCKED`;
- DEFERRABLE constraints;
- custom LOGIN role through Supabase session pooler;
- command/outbox fencing;
- reconnect after database/network interruption;
- password rotation behavior;
- migration-owner separation.

If direct connection is later used, rerun the same suite.

---

# 25. Connection-pool sizing

Do not copy serverless assumptions.

Persistent services maintain a small pool.

Initial pool size is configuration, not architecture.

Measure:
- connection saturation;
- pool wait time;
- command p95/p99 latency;
- worker claim latency.

Scale connections conservatively because each distinct role/mode combination can consume pooler/database resources.

---

# 26. Runtime failure boundaries

## Railway API down

New commands/views unavailable.

Existing canonical state remains safe in PostgreSQL.

## Railway worker down

Outbox accumulates.

Canonical state remains committed.

Presentation/realtime delivery resumes on worker recovery.

## Supabase Auth down

New/refresh auth flows may fail.

Already-issued token behavior depends on JWT validity; engine still verifies accepted tokens under configured rules.

## Supabase Postgres down

Canonical gameplay writes stop.

No fail-open path.

## Realtime down

Gameplay may continue through API polling/refetch.

Realtime is convenience/latency layer, never authority.

---

# 27. Monitoring baseline

Before production playtests monitor:

API:
- request/error rate;
- command latency;
- DB pool wait/usage;
- resolver CPU duration;
- canonical commit failure codes.

Worker:
- oldest pending Outbox age;
- claim rate;
- failed retryable/terminal items;
- stale leases;
- presentation latency/provider errors.

Database:
- active connections;
- lock waits/deadlocks;
- event/session table growth;
- slow transactions.

Realtime:
- invalidation send failures;
- subscription authorization failures.

---

# 28. Security baseline

- API/worker use separate DB credentials.
- Migration owner separate.
- No Supabase secret/service-role key in engine runtime.
- Auth JWT verified before access binding lookup.
- Access schema not browser-exposed.
- Invite plaintext never persisted.
- No private world/free text in telemetry by default.
- Worker provider keys never available to player web/API unless explicitly necessary.
- Railway service variables scoped by service; seal production values where possible.

---

# 29. Alternatives

## Supabase Edge for API/worker

Rejected for authoritative runtime in this phase.

Reason:
- broad default project secrets are injected into hosted function environment;
- conflicts with accepted least-privilege DB service separation;
- serverless CPU/pool constraints add complexity.

## Single Railway service combining API + worker

Rejected initially.

Reason:
- mixes secret scopes;
- API would receive worker/LLM provider secrets;
- worker failure pressure can affect API.

Two services are a cheap security/reliability boundary.

## Redis/queue broker

Rejected.

PostgreSQL Outbox already provides durable work authority.

## Cloudflare/Durable Objects

Rejected until measurements demonstrate need.

---

# 30. Accepted-design compatibility

This platform does not change:

- Single Canonical Frontier;
- Event Sourcing scope;
- SessionRuntime locking;
- command/outbox fencing;
- LLM boundaries;
- PostgreSQL Schema v0.5 semantics;
- deterministic replay;
- PlayerInteractionView privacy.

Runtime services are replaceable adapters around accepted core.

---

# 31. Proposed ADR mapping

ADR-015:
- Guest Identity and Session Participation.

ADR-016:
- Supabase PostgreSQL/Auth/Realtime Platform Baseline.

ADR-017:
- Persistent API/Worker Runtime and Durable Outbox Processing.

All remain PROPOSED until explicit acceptance.

---

# 32. Red-team targets

Before acceptance test the architecture against:

1. Railway API credential leak;
2. Railway worker credential leak;
3. compromised API cannot mutate using worker role;
4. compromised worker cannot append canonical events;
5. API JWT signed with unknown/rotated key;
6. JWT valid but access binding revoked;
7. same AuthSubject attempts both roles;
8. two users claim one invite concurrently;
9. Auth user deleted but stale binding remains;
10. worker crashes before/after external LLM call;
11. two worker replicas claim same task;
12. Railway worker down for hours;
13. Realtime wrapper called with arbitrary topic/payload;
14. browser tries direct `engine`/`access` table access;
15. Supabase Realtime private-topic RLS mistake;
16. Supabase secret key accidentally added to Railway;
17. session pooler custom-role authentication/password rotation;
18. DB connection exhaustion;
19. Railway/Supabase regions far apart;
20. Railway outage;
21. Supabase outage;
22. provider migration away from Railway;
23. provider migration away from Supabase Auth;
24. canonical replay with both compute services absent.

---

# 33. Acceptance dependencies

Before accepting v0.3:

- red-team this baseline;
- align ADR-015..017;
- update PROJECT_STATE/glossary/references.

After acceptance, and before implementation code:
- create/select Supabase project;
- choose Supabase region;
- record server version;
- create Railway project/services configuration plan;
- choose nearby Railway region;
- perform DB custom-role/session-pooler spike;
- verify JWKS authentication;
- verify Realtime RLS/wrapper design;
- create additive access-schema design/DDL proposal and red-team it.

Only then generate executable migrations and implementation skeleton.
