# IMPLEMENTATION PLATFORM BASELINE v0.1

**Status:** PROPOSED  
**Date:** 2026-10-02  
**Governing architecture:** Architecture Baseline v0.3 — ACCEPTED  
**Governing persistence:** Persistence Data Model v0.3 — ACCEPTED  
**Governing schema:** PostgreSQL Schema v0.5 — ACCEPTED  
**Purpose:** freeze the minimum provider/runtime/identity decisions required before executable migrations and implementation skeleton.

---

# 1. Decision scope

This proposal chooses only what is implementation-blocking now:

- managed PostgreSQL/provider baseline;
- target PostgreSQL major;
- guest authentication baseline;
- Session participant/principal separation;
- authoritative HTTP runtime;
- database connection mode;
- outbox execution/wakeup model;
- realtime invalidation model;
- internal schema exposure/security stance.

It deliberately does NOT freeze:

- final frontend visual architecture;
- frontend hosting provider;
- media/object storage provider;
- payment provider;
- analytics vendor details;
- long-term account/profile system;
- cross-device guest recovery UX;
- paid entitlement model.

---

# 2. Proposed platform

## Database/platform

Use **managed Supabase** for MVP infrastructure:

- PostgreSQL 17 baseline;
- Supabase Auth;
- Supabase Realtime;
- Supabase Edge Functions;
- Supavisor/managed connection pooling;
- Vault/cron capabilities only where needed.

### FACT

As of 2026-10-02, Supabase states that Postgres 17 is the current default platform generation; existing projects can report/upgrade their exact server version.

Primary sources:
- https://supabase.com/docs/guides/platform/upgrading
- https://supabase.com/changelog/46080-self-hosted-supabase-upgrading-from-pg-15-to-17-breaking-change
- https://supabase.com/docs/guides/database/postgres/which-version-of-postgres

### PROPOSAL

Create the project on Postgres 17 and record the actual deployed server version before executable migrations.

Do not rely on Postgres 17-only optional extensions unless explicitly accepted later.

---

# 3. Why Supabase now

The project already requires:

- PostgreSQL;
- anonymous/guest auth;
- private realtime notification;
- server-side TypeScript entrypoints;
- migrations/local testing;
- short-lived/serverless DB connection support.

Supabase provides those in one operational boundary.

This reduces MVP infrastructure while preserving portability because:

- canonical domain/resolver packages remain provider-independent;
- PostgreSQL schema remains ordinary PostgreSQL;
- Realtime is notification only;
- LLM/presentation work remains outside canonical mechanics.

---

# 4. Alternatives considered

## A. Supabase managed platform — PROPOSED

Advantages:
- one managed PostgreSQL/Auth/Realtime/Edge platform;
- anonymous Auth is first-party;
- private Realtime authorization is built in;
- Edge Functions are TypeScript;
- direct PostgreSQL transactions are possible;
- local stack/CLI available.

Risks:
- Edge Function CPU/time limits;
- platform-specific auth/realtime glue;
- serverless DB connection complexity.

## B. Managed PostgreSQL + separate Auth + persistent Node service

Advantages:
- conventional long-running DB connections;
- no 2s Edge CPU cap;
- worker loop is straightforward.

Costs:
- at least 2–3 providers/services immediately;
- more secrets/network/runtime configuration;
- separate auth integration;
- no demonstrated runtime load need yet.

Reject for MVP unless Edge runtime limits become a measured problem.

## C. Cloudflare-centric stateful backend

Could use Workers plus database connectivity and potentially Durable Objects.

Rejected for this phase:
- would add a second stateful coordination model while PostgreSQL Single Canonical Frontier is already accepted;
- Durable Objects remain explicitly premature without measured need.

## D. Supabase Queues / pgmq replacing project Outbox

Rejected.

The accepted Transactional Outbox already solves project-specific post-commit reliability with required fencing/evidence semantics.

Adding pgmq now would duplicate authority/retry state.

---

# 5. Edge Functions runtime

## Proposed functions

Initially two server-side deployables:

### `api`

Authenticated authoritative HTTP API.

Representative routes:
- create Session;
- create/claim invite;
- submit/replace/finalize action;
- get PlayerInteractionView;
- resume/list permitted Session metadata.

Use one routed function initially rather than many tiny functions.

### `outbox-drain`

Internal worker endpoint.

Responsibilities:
- claim Outbox rows;
- BUILD_PRESENTATION;
- DELIVER_PRESENTATION;
- emit Realtime invalidation;
- emit noncanonical analytics tasks where applicable.

No canonical state mutation except by submitting a new authorized EngineCommand where a later workflow explicitly requires it.

---

# 6. Edge runtime constraint

### FACT

Supabase hosted Edge Functions currently document:

- 256 MB memory;
- max wall time 150s free / 400s paid;
- 2s active CPU time per request.

Source:
https://supabase.com/docs/guides/functions/limits

### PROPOSAL

MVP authoritative command handling may run on Edge Functions only while:

- resolver/progression CPU remains comfortably below platform limits;
- DB transaction work remains short;
- large synthetic simulations never run inside request runtime.

### Revisit trigger

Move authoritative API/worker execution to a persistent Node/Deno service without changing contracts/schema if:

- runtime canonical commands approach the 2s CPU ceiling;
- p99 resolver CPU is persistently > 500ms under representative scenarios;
- worker job orchestration regularly approaches Edge wall-time;
- driver/pooler behavior creates operational instability.

The 500ms value is an engineering warning threshold, not a gameplay semantic deadline.

---

# 7. Database connections from Edge Functions

### FACT

Supabase recommends transaction-pooler connections for serverless/edge workloads.

Transaction mode does not support prepared statements or query pipelining.

Current Supabase documentation also warns that `postgres.js` pipelining can conflict with shared transaction pooling.

Sources:
- https://supabase.com/docs/guides/database/connecting-to-postgres
- https://supabase.com/docs/guides/database/postgres-js
- https://supabase.com/docs/guides/functions/connect-to-postgres

### PROPOSAL

For authoritative engine transactions:

- use a direct PostgreSQL client from Edge Functions;
- prefer the Deno Postgres driver / Kysely adapter path demonstrated by Supabase;
- use the transaction pooler for deployed Edge Functions;
- keep client pool size minimal;
- do not depend on session-level PostgreSQL state;
- do not use prepared statements/session advisory locks through transaction mode;
- execute accepted lock/fencing protocols in explicit SQL transactions.

Do not use PostgREST/supabase-js as the canonical multi-row transaction engine.

It can still be used for Auth and non-authoritative simple reads where appropriate.

Exact query-builder library remains unfrozen until a transaction integration test passes.

---

# 8. Guest authentication

Use Supabase Auth anonymous sign-in for first-play guests.

### FACT

Supabase anonymous sign-in:

- creates a real authenticated user without requiring PII;
- uses the `authenticated` PostgreSQL role;
- exposes an `is_anonymous` JWT claim;
- can later link an authentication method;
- cannot be recovered after sign-out/browser-data loss unless an identity was linked;
- has no automatic anonymous-user cleanup;
- Supabase strongly recommends CAPTCHA/Turnstile to prevent abuse.

Source:
https://supabase.com/docs/guides/auth/auth-anonymous

### PROPOSAL

- enable anonymous Auth;
- require Turnstile/CAPTCHA for anonymous account creation;
- apply application rate limits to Session creation/invite claim;
- define cleanup only after retention policy is accepted;
- do not collect mandatory email/name/phone for guest play.

---

# 9. Identity separation

Three identities remain distinct.

## AuthSubject

Supabase Auth user id/JWT subject.

Operational security identity.

May be anonymous or later linked to a permanent login.

Not canonical game state.

## ParticipantRef

Internal opaque gameplay participant identity.

Used by canonical ParticipationState.

Does not contain email/auth provider/PII.

## CharacterEntityId

Canonical in-world character identity.

ParticipantRef may be bound to CharacterEntityId/RoleRef by canonical Session state.

### Invariant

```
AuthSubject != ParticipantRef != CharacterEntityId
```

Never use Supabase Auth user id as Character Entity id or semantic ParticipantRef.

---

# 10. Access-binding persistence

Add a separate internal operational access area in a later additive schema revision.

Proposed relation:

```
session_principal_bindings
  session_id
  participant_ref
  auth_subject
  binding_status
  created_at
  optional revoked_at
```

Purpose:

- authorize an authenticated caller to act for one canonical ParticipantRef;
- avoid placing AuthSubject/PII inside canonical world state.

This table is security authority for **who may authenticate as ParticipantRef**, not game-world authority for who the Participant/Character is.

The canonical Participant binding remains in Session events/state.

### Atomic rule

When a guest first claims a participant slot:

- access binding;
- canonical ParticipantBound transition;

must commit atomically in one PostgreSQL transaction or neither commits.

A later authentication-method link that preserves the same AuthSubject requires no canonical gameplay change.

---

# 11. Session creation identity flow

Proposed flow:

1. browser obtains/creates anonymous Supabase Auth session;
2. API validates JWT;
3. API creates Session + Genesis revision 0;
4. API creates internal ParticipantRef for creator;
5. API writes access binding AuthSubject -> ParticipantRef;
6. API commits canonical ParticipantBound event/transition as revision 1+;
7. API creates partner invite if requested.

Revision 0 remains free of dynamic account identity.

---

# 12. Invite design

Invitation is operational access bootstrap, not canonical game truth.

Proposed invite relation:

```
session_invites
  invite_id
  session_id
  target_participant_slot
  token_hash
  status
  created_by_participant_ref
  created_at
  expires_at
  optional claimed_at
```

## Token properties

- cryptographically random 256-bit bearer token;
- base64url/URL-safe representation;
- store only cryptographic hash in database;
- token plaintext appears only in generated invite URL/client exchange;
- one-time claim;
- expiration required;
- exact expiry duration remains product-configurable.

Because the token is high entropy/random, a fast cryptographic hash is sufficient; it is not a human password.

## Browser URL

Prefer placing the raw invite token in a URL fragment rather than server query string:

```
https://app.example/#/join/<token>
```

The client immediately exchanges it over HTTPS and removes it from browser-visible navigation state.

Do not log invite tokens.

---

# 13. Invite claim transaction

Authoritative claim:

1. authenticated AuthSubject calls API with raw invite token;
2. backend hashes token;
3. lock matching active invite;
4. verify not expired/claimed;
5. lock SessionRuntime;
6. verify target participant slot remains claimable;
7. create ParticipantRef if not preallocated;
8. insert session principal binding;
9. mark invite claimed;
10. emit/commit canonical ParticipantBound transition;
11. create PlayerInteractionView/presentation work as required;
12. commit.

Duplicate/replayed invite claims are rejected/idempotently report claimed state.

No client-supplied ParticipantRef is authoritative.

---

# 14. Guest limitations

MVP anonymous guest identity has a deliberate limitation:

> clearing browser storage, signing out, or moving to another device can lose access to a claimed guest identity.

Do not create a complicated recovery-token system before real need.

Future options:
- link email/OAuth/passkey before switching device;
- explicit participant-access recovery flow;
- host/admin-supported rebind.

Any rebind changes operational access identity, not canonical Character history.

---

# 15. Engine schema exposure

Supabase Data API exposes `public` by default and only exposes custom schemas when configured.

Source:
https://supabase.com/docs/guides/api/using-custom-schemas

### PROPOSAL

- `engine` schema is never added to Supabase exposed schemas;
- browser roles receive no `USAGE` or table privileges on `engine`;
- authoritative engine reads/writes go through Edge Functions/direct Postgres;
- use RLS only as defense-in-depth where provider exposure or service tables require it.

This preserves accepted PostgreSQL Schema v0.5 boundaries.

---

# 16. Realtime

Use Supabase Realtime **private Broadcast** only as invalidation/notification.

### FACT

Supabase Realtime supports private Broadcast channels authorized by RLS policies on `realtime.messages`.

Sources:
- https://supabase.com/docs/guides/realtime/authorization
- https://supabase.com/docs/guides/realtime/broadcast

### PROPOSAL

Client subscribes to a private Session/participant topic.

Messages contain minimal nonsecret invalidation information, e.g.:

```
{
  type: "session_view_changed",
  session_id,
  revision
}
```

They do NOT contain:
- canonical world state;
- another player's ActionSubmission;
- secret facts;
- narrative private payload.

On notification, client fetches its current PlayerInteractionView from authoritative API.

## Authorization

Realtime RLS checks AuthSubject -> Session principal binding.

Do not grant all `authenticated` users access to all Session topics.

Client has receive permission only; normal gameplay clients do not need Broadcast-send permission.

---

# 17. Outbox execution

Keep the accepted project Outbox.

## Fast path

After a successful command/transition response path, the API may call:

`EdgeRuntime.waitUntil(drainRelevantOutbox())`

to reduce player-visible latency.

### FACT

Supabase supports background tasks via `EdgeRuntime.waitUntil`, but tasks remain bounded by function CPU/wall limits.

Source:
https://supabase.com/docs/guides/functions/background-tasks

## Recovery path

Use `pg_cron` + `pg_net` to invoke the internal `outbox-drain` Edge Function periodically.

Initial proposal:
- every 10 seconds.

Supabase documents pg_cron/Edge invocation, and its current automatic-embeddings reference uses a 10-second pg_cron interval.

Sources:
- https://supabase.com/docs/guides/functions/schedule-functions
- https://supabase.com/docs/guides/ai/automatic-embeddings

### Invariant

Neither `waitUntil` nor pg_cron is the work authority.

`engine.outbox` remains the durable authority.

A missed wakeup delays work but cannot lose it.

---

# 18. Do not adopt Supabase Queues yet

Supabase now offers Postgres-native durable queues/pgmq.

FACT source:
https://supabase.com/docs/guides/queues

Do not use it for engine continuation work in MVP.

Reason:
- duplicates accepted project Outbox;
- introduces another retry/visibility state model;
- the project Outbox already encodes PresentationId, fencing and canonical transition context.

Revisit only if the custom Outbox becomes demonstrably burdensome and a migration can preserve its semantics.

---

# 19. Frontend

Keep current proposal:

- React;
- TypeScript;
- Vite/browser-first SPA/PWA-compatible structure.

Do not freeze final hosting provider yet.

Cloudflare static hosting remains a reasonable option but is not a migration/runtime blocker.

---

# 20. Repository/runtime boundaries

Proposed implementation packages remain provider-separated:

```
packages/domain
packages/contracts
packages/state
packages/resolver
packages/epistemics
packages/temporal
packages/projections
packages/replay
packages/validation

packages/persistence-postgres
packages/platform-supabase

apps/player-web

supabase/functions/api
supabase/functions/outbox-drain

db/migrations
```

Rules:

- resolver/domain packages import no Supabase SDK;
- Supabase JWT/Auth/Realtime code lives in platform adapter boundary;
- PostgreSQL SQL remains provider-neutral under `db/migrations`;
- Edge Function wrappers call application/domain services.

---

# 21. Proposed ADRs

This baseline requires:

- ADR-015 — Guest Identity and Session Participation;
- ADR-016 — Managed Supabase/PostgreSQL 17 Platform Baseline;
- ADR-017 — Edge Function API and Outbox Wakeup Model.

All remain PROPOSED until explicit acceptance.

---

# 22. Red-team targets

Before acceptance test:

1. anonymous user creates excessive auth rows;
2. guest clears local storage after claiming role;
3. invite link leaks through logs/referrers;
4. two users claim same invite simultaneously;
5. one AuthSubject attempts both participant roles;
6. stale/deleted AuthSubject still has binding row;
7. binding row exists but canonical ParticipantBound commit fails;
8. canonical participant exists but access binding transaction fails;
9. engine schema accidentally added to exposed schemas;
10. generic `authenticated` RLS accidentally grants every guest Realtime access;
11. Realtime notification leaks secret state;
12. Edge Function exceeds 2s CPU;
13. DB pooler transaction mode conflicts with chosen driver;
14. worker dies after `waitUntil` starts;
15. pg_cron wakeup stops;
16. both fast-path and cron invoke Outbox concurrently;
17. old worker wakes after lease takeover;
18. Supabase Auth account is linked/upgraded;
19. anonymous Auth user cleanup deletes active Session user;
20. Supabase provider outage / migration portability;
21. Postgres major upgrade changes behavior;
22. service secret leaks to browser bundle.

---

# 23. Acceptance gate

Red-team/postmortem this baseline.

If material defects are found:
- create implementation platform v0.2;
- update ADR proposals accordingly.

Do not create executable migrations or implementation skeleton before platform baseline + ADR-015..017 are sufficiently stable/accepted.
