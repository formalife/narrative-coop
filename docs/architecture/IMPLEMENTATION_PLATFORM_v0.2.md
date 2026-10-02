# IMPLEMENTATION PLATFORM BASELINE v0.2

**Status:** PROPOSED  
**Date:** 2026-10-02  
**Supersedes as working proposal:** IMPLEMENTATION_PLATFORM_v0.1  
**Governing architecture:** Architecture Baseline v0.3 — ACCEPTED  
**Governing persistence:** Persistence Data Model v0.3 — ACCEPTED  
**Governing schema:** PostgreSQL Schema v0.5 — ACCEPTED

---

# 1. Proposed implementation-blocking decisions

If accepted:

1. managed Supabase is the MVP database/auth/realtime/edge platform;
2. PostgreSQL major baseline is 17;
3. guest play uses Supabase anonymous Auth + Turnstile/CAPTCHA;
4. AuthSubject, ParticipantRef and CharacterEntityId remain distinct;
5. operational authorization/invites live in internal `access` schema;
6. authoritative HTTP API runs initially on Supabase Edge Functions;
7. Edge runtime connects through PostgreSQL transaction pooler using a transaction-capable direct driver;
8. project PostgreSQL Outbox remains the only durable continuation-work authority;
9. Edge background tasks + pg_cron/pg_net only wake the Outbox worker;
10. Realtime private Broadcast carries invalidations only;
11. provider-neutral core SQL and Supabase-specific deployment SQL/config remain separated.

---

# 2. Supabase / PostgreSQL baseline

Use managed Supabase.

Target major:
- PostgreSQL 17.

Before executable migration sign-off:

```sql
show server_version;
```

must be run against the target project and recorded in deployment documentation.

Canonical engine SQL relies on PostgreSQL semantics, not Supabase Data API semantics.

No canonical mechanic depends on provider-specific extension behavior.

---

# 3. Core versus provider-specific repository layout

```
db/
  migrations/
    provider-neutral PostgreSQL engine/access schema

platform/
  supabase/
    config/
    sql/
      realtime authorization
      pg_cron/pg_net wakeup
      Vault integration
      Supabase-specific role/bootstrap glue
    functions/
      api/
      outbox-drain/
```

Both directories are canonical GitHub source.

Rules:

- `db/migrations` can be reasoned about as ordinary PostgreSQL;
- `platform/supabase` may depend on Supabase schemas/services;
- provider glue cannot redefine canonical state/event semantics.

---

# 4. Guest Auth

Use Supabase Auth anonymous sign-in.

Requirements:

- CAPTCHA/Turnstile enabled for anonymous sign-in;
- browser uses current Supabase publishable key, never server secret;
- no mandatory email/phone/name;
- application quotas on Session/invite operations;
- no raw IP retained by default for game-domain needs.

Anonymous identity upgrade path:

- link email/phone/OAuth identity to the current user;
- canonical Session identity is unchanged.

Signing into a different existing account and merging access is a future operation.

---

# 5. Identity invariants

```
AuthSubject
  external security identity

ParticipantRef
  internal gameplay participant identity

CharacterEntityId
  canonical world Character
```

Invariant:

```
AuthSubject != ParticipantRef != CharacterEntityId
```

AuthSubject:
- never appears in CanonicalStateContent;
- never contributes to gameplay state hash;
- never acts as EntityId.

ParticipantRef:
- canonical opaque participant reference;
- linked to role/character in ParticipationState.

---

# 6. Internal access schema

Additive persistence work after ADR-015 acceptance:

```
access.session_principal_bindings
access.session_invites
```

The `access` schema is:
- backend-only;
- not Data-API exposed;
- not directly readable by browser authenticated role.

It is operational security authority, not world-state authority.

---

# 7. session_principal_bindings logical contract

```
session_principal_bindings
  session_id
  participant_ref
  auth_subject

  status:
    ACTIVE
    REVOKED

  created_at
  optional revoked_at
```

MVP invariants:

1. one ACTIVE AuthSubject may control at most one ParticipantRef per Session;
2. one ParticipantRef has at most one ACTIVE AuthSubject;
3. authorization begins from a verified JWT, then binding lookup;
4. binding row alone never authenticates anyone;
5. AuthSubject is not FK-coupled to `auth.users`;
6. account deletion can leave stale binding evidence but cannot authenticate;
7. access rebind/recovery is explicit and revokes old binding.

Synthetic/test agents bypass guest bindings through internal test principals, not public access records.

---

# 8. Creator Session transaction

Preferred create-session transaction:

1. verify AuthSubject JWT;
2. create Session;
3. create Genesis;
4. create Runtime revision 0;
5. create ParticipantRef;
6. insert ACTIVE access binding;
7. build canonical ParticipantBound transition;
8. append transition/events and advance Runtime to revision 1+;
9. optionally create invite;
10. materialize required PlayerView/Presentation/Outbox;
11. commit.

Benefits:
- no visible orphan Session with no creator access;
- revision 0 remains deterministic/account-free;
- external identity never enters canonical event payload.

---

# 9. Invite contract

```
access.session_invites
  invite_id
  session_id
  target_participant_slot

  token_hash

  status:
    ACTIVE
    CLAIMED
    REVOKED
    EXPIRED

  created_by_participant_ref

  created_at
  expires_at
  optional claimed_at
```

Invariants:

- 256-bit cryptographically random raw token;
- cryptographic hash only in DB;
- token hash unique;
- at most one ACTIVE invite per target slot by default;
- one-time claim;
- expiry required;
- terminal invite states do not reopen;
- raw token excluded from logs/analytics/error telemetry.

Exact expiry duration remains configuration.

---

# 10. Invite browser handling

Prefer:

```
https://app.example/#/join/<token>
```

Client:

1. reads fragment;
2. removes/scrubs raw token from navigation state immediately;
3. ensures anonymous/permanent Auth session exists;
4. POSTs raw token to authoritative API over HTTPS;
5. drops raw token from application state after response.

No server logs receive token through initial document URL.

---

# 11. Invite claim transaction

1. verify caller JWT -> AuthSubject;
2. hash supplied token;
3. lock matching ACTIVE invite;
4. verify not expired;
5. lock SessionRuntime;
6. verify target slot claimable;
7. verify AuthSubject has no other ACTIVE binding in Session;
8. create ParticipantRef if needed;
9. insert access binding;
10. mark invite CLAIMED;
11. commit canonical ParticipantBound transition;
12. create current PlayerView/presentation work;
13. commit.

If any step fails, no access/canonical split-brain commits.

---

# 12. Anonymous cleanup

Supabase currently does not automatically clean anonymous users.

Cleanup policy is deferred until retention is accepted, but must obey:

> never delete an anonymous AuthSubject while it has an ACTIVE access binding to a Session that is nonterminal or within the supported resume/recovery window.

Cleanup must query access/session state first.

Deleting old Auth records never deletes canonical Session history.

---

# 13. Authoritative Edge API

Initial runtime:
- one routed Supabase Edge Function `api`.

JWT verification occurs before domain command handling.

Server derives:
- AuthSubject from verified JWT;
- ParticipantRef from access binding;
- EngineCommand Principal from server state.

Client input cannot assert a trusted ParticipantRef.

Representative endpoints:
- create Session;
- issue/revoke invite;
- claim invite;
- submit/replace/finalize action;
- get current PlayerInteractionView;
- minimal Session resume metadata.

---

# 14. Edge database transaction path

Current platform constraint:
- serverless/Edge uses Supabase transaction pooler;
- no prepared-statement/session-state assumptions;
- no dependency on query pipelining.

Preferred initial driver:
- Deno Postgres.

Optional query builder:
- Kysely.

Not frozen until integration spike proves:

1. BEGIN/COMMIT/ROLLBACK;
2. SELECT FOR UPDATE;
3. DEFERRABLE constraint behavior;
4. accepted lock order;
5. command generation fencing;
6. Outbox generation fencing;
7. connection reuse/recovery under Edge lifecycle.

Do not choose a more convenient driver if it weakens transaction guarantees.

---

# 15. DB role model

Provider-neutral NOLOGIN group roles:

```
engine_runtime
engine_worker
engine_publisher
engine_ops_readonly
```

Provider deployment creates separate login credentials:

```
engine_runtime_login -> engine_runtime
engine_worker_login  -> engine_worker
```

Migration/owner role remains separate.

Secrets:
- no DB password in Git;
- API gets runtime connection secret only;
- outbox worker gets worker connection secret only.

Supabase broad server secret keys are not ordinary engine DB credentials.

---

# 16. API versus worker privilege split

## api

Holds:
- runtime DB credential;
- ability to verify user JWT;
- worker **wake token** only.

Does NOT hold:
- worker DB credential;
- presentation provider secrets unless an API route explicitly needs them.

## outbox-drain

Holds:
- worker DB credential;
- worker wake-token verification secret;
- presentation/external provider secrets needed by worker tasks.

Does NOT hold canonical runtime DB write privileges.

---

# 17. Worker wake token

Use dedicated 256-bit random secret.

Stored:
- Supabase Vault for pg_net caller;
- outbox-drain Edge Function secret.

Request carries dedicated wake header.

The token:
- authorizes only "run drain loop";
- carries no Session/participant authority;
- is independently rotatable;
- must never be shipped to browser.

A leaked wake token can trigger extra work but cannot read/mutate canonical state directly.

---

# 18. Outbox wakeups

## Immediate latency optimization

After commit, `api` may:

```
EdgeRuntime.waitUntil(
  callOutboxDrainWithWakeToken()
)
```

It does not drain DB work with worker credentials itself.

## Recovery sweep

Supabase-specific platform SQL uses:
- pg_cron;
- pg_net;
- Vault-stored wake token/url;

to invoke `outbox-drain` periodically.

Initial operational default:
- 10 seconds.

This interval is tunable without architecture change.

## Authority

`engine.outbox` remains durable work authority.

Wake failures only delay processing.

---

# 19. Worker health

Monitor at minimum:

- age of oldest PENDING/FAILED_RETRYABLE Outbox item;
- PROCESSING rows past lease;
- repeated failed worker invocations;
- pg_cron/pg_net invocation failures;
- presentation generation latency/failure.

No second queue or distributed lock.

---

# 20. No pgmq for engine continuation

Do not adopt Supabase Queues/pgmq for canonical continuation/presentation work.

Reason:
- duplicates accepted Outbox retry/fencing model.

It may be considered later for unrelated workloads if it does not become canonical Session continuation authority.

---

# 21. Realtime

Use private Supabase Realtime Broadcast.

Topic:
`session:<session_id>`

Payload:

```
{
  type: "session_view_changed",
  session_id
}
```

Do not rely on StreamRevision in notification freshness because pending input can change PlayerInteractionView at the same canonical revision.

Notification contains no:
- world state;
- private PlayerView;
- pending action;
- secret;
- narrative content.

Client always fetches current view through API.

---

# 22. Realtime authorization helper

Browser authenticated role must not SELECT access tables directly.

Create narrow Supabase-specific helper:

```
access.can_receive_session_topic(
  auth_subject uuid,
  session_id uuid
) -> boolean
```

Requirements:
- SECURITY DEFINER;
- `search_path = ''`;
- fully qualified refs;
- PUBLIC execute revoked;
- grant only to required authenticated/Realtime execution role;
- returns boolean only.

Realtime `realtime.messages` SELECT policy calls helper.

No generic all-authenticated policy.

Clients receive only; no normal client Broadcast INSERT policy.

---

# 23. Engine/access schema exposure

Supabase API settings:

- do not expose `engine`;
- do not expose `access`.

Browser Data API roles:
- no USAGE/table grants on either.

If provider configuration changes exposure:
- deployment verification fails;
- do not rely on absence of RLS alone.

Internal authoritative access is through Edge runtime DB roles.

---

# 24. Edge runtime limits/revisit

Current hosted Edge fact:
- 2s CPU/request;
- 150s free / 400s paid wall time;
- 256MB memory.

The engine may use Edge only while runtime work fits comfortably.

Internal signal for reconsideration:
- representative p99 resolver/progression CPU > 500ms;
- recurring CPU limit warnings;
- worker wall-time pressure;
- persistent pooler/driver instability.

Migration to persistent runtime may happen without changing domain/contracts/schema authority.

Synthetic simulations/load testing run outside hosted Edge runtime.

---

# 25. Frontend

Keep:
- React;
- TypeScript;
- Vite.

Frontend hosting remains unfrozen.

Browser uses:
- Supabase publishable key for Auth/Realtime;
- authoritative API for gameplay state/actions.

No server secret or DB credential enters frontend bundle.

---

# 26. Explicitly still open

- exact Supabase project/region/plan;
- exact PostgreSQL 17 minor/build;
- UUID generation algorithm;
- query builder after driver spike;
- anonymous retention duration;
- invite expiry duration;
- frontend hosting;
- object/media storage;
- paid account/entitlement architecture;
- cross-device access recovery;
- permanent-account merge;
- final rate-limit implementation.

---

# 27. Acceptance dependencies

Before acceptance:

1. red-team all access races and cleanup;
2. verify current Supabase docs support anonymous Auth, PG17, private Realtime and Edge direct DB path;
3. verify no accepted engine invariant is weakened;
4. ensure provider glue is separated from core SQL;
5. produce ADR-015..017 matching this baseline.

Before executable migrations/application implementation:

6. create/select Supabase project;
7. record `show server_version`;
8. verify provider exposed-schema configuration;
9. run Deno Postgres transaction/pooler spike;
10. map/create DB group/login roles and verify grants;
11. run PostgreSQL Schema v0.5 DDL suite on actual target.
