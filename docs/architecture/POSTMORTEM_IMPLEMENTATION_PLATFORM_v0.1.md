# POSTMORTEM — Implementation Platform Baseline v0.1

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/architecture/IMPLEMENTATION_PLATFORM_v0.1.md`  
**Reviewed ADRs:** ADR-015, ADR-016, ADR-017 — PROPOSED  
**Verdict:** MODIFY

## Executive verdict

The central direction is sound:

- managed Supabase/PostgreSQL 17;
- Supabase anonymous Auth;
- AuthSubject separated from ParticipantRef/Character;
- authoritative Edge Function API;
- accepted PostgreSQL Outbox as durable work authority;
- private Realtime Broadcast as invalidation only.

The red-team found security/operational details that must be promoted from implication to explicit platform contracts before acceptance.

No accepted ADR-001..014 or accepted schema/persistence decision needs supersession.

---

# 1. Anonymous-auth abuse

## v0.1

Requires CAPTCHA/Turnstile and rate limiting.

## Finding

PASS directionally, but application abuse and Auth-user creation abuse are distinct.

Supabase currently:
- supports Cloudflare Turnstile/hCaptcha;
- recommends CAPTCHA for anonymous sign-ins;
- applies a configurable anonymous-sign-in IP limit.

Sources:
- https://supabase.com/docs/guides/auth/auth-anonymous
- https://supabase.com/docs/guides/auth/auth-captcha

## Correction

v0.2 must require:
- Turnstile/CAPTCHA at anonymous Auth creation;
- separate application quotas for Session creation/invite generation/claim;
- no raw IP persistence by default.

---

# 2. Anonymous-user cleanup can break active access

## Finding

Supabase currently has no automatic anonymous-user cleanup.

A naive age-based cleanup can delete an anonymous AuthSubject that still owns an active Session binding.

## Correction

Anonymous Auth cleanup MUST be access-aware:

Never delete an anonymous auth user while it has an ACTIVE `session_principal_binding` to a nonterminal/recoverable Session.

Cleanup policy runs only after retention policy is accepted.

This is operational identity cleanup, not gameplay state deletion.

---

# 3. One AuthSubject can currently claim both roles

## Problem

v0.1 did not explicitly forbid one authenticated guest from binding to two ParticipantRefs in the same Session.

That can accidentally defeat the asymmetric two-player product invariant.

## Correction

Default MVP access invariants:

- at most one ACTIVE AuthSubject per ParticipantRef in a Session;
- at most one ACTIVE ParticipantRef per AuthSubject in a Session.

Testing/admin synthetic players use privileged tooling, not normal guest binding.

A future same-human multi-role mode requires explicit product decision.

---

# 4. Access-binding owner needs its own schema/boundary

## Problem

v0.1 names a relation but not an ownership namespace.

## Correction

Add internal, non-exposed:

`access` schema.

Initial relations:
- `access.session_principal_bindings`;
- `access.session_invites`.

This keeps:
- canonical game persistence in `engine`;
- operational authorization/invite state in `access`.

Neither schema is browser-exposed.

This is an additive schema extension intentionally deferred by accepted PostgreSQL Schema v0.5.

---

# 5. Session bootstrap + creator binding should avoid orphan Session creation

## Problem

v0.1 describes Session/Genesis creation and creator ParticipantBound as sequential steps.

A process crash can leave a valid revision-0 Session with no access binding.

This is recoverable but unnecessary.

## Correction

Preferred create-session transaction:

1. Session;
2. Genesis;
3. Runtime revision 0;
4. access binding;
5. canonical ParticipantBound transition revision 1+;
6. Runtime after-state;
7. optional invite;
8. required view/presentation/outbox;
9. commit.

Revision 0 remains deterministic and account-free even though revision 1 is committed in the same database transaction.

If implementation keeps bootstrap/first-transition as two transactions, CommandKey/idempotent recovery must close the orphan window. One transaction is preferred.

---

# 6. Invite security

## Finding

v0.1 bearer-token approach is sound.

Required hardening:
- SHA-256 or pinned cryptographic hash over random 256-bit token is sufficient;
- token hash unique;
- raw token excluded from analytics/logs/error reporting;
- URL fragment scrubbed immediately after client reads it;
- one active invite per target slot by default;
- claimed/revoked/expired tokens are terminal.

Exact expiry duration remains product configuration.

---

# 7. Auth upgrade semantics

## Finding

Supabase supports converting an anonymous user by linking email/phone/OAuth identity.

Ordinary linking preserves the underlying user identity.

Source:
https://supabase.com/docs/guides/auth/auth-anonymous

## Correction

Supported MVP upgrade path:
- link a permanent identity to the current anonymous user.

This requires no access-binding/canonical Participant change.

Signing into a different existing account and merging/reassigning access is NOT the same operation and remains future explicit recovery/merge work.

---

# 8. Realtime authorization must not expose access tables

## Problem

Realtime RLS needs to test AuthSubject -> Session binding, but browser `authenticated` role must not receive SELECT on `access.session_principal_bindings`.

## Correction

Use a narrow SECURITY DEFINER authorization helper, e.g.:

`access.can_receive_session_topic(auth_subject, session_id)`

Requirements:
- `search_path = ''`;
- fully qualified relation refs;
- PUBLIC execute revoked;
- EXECUTE granted only to required Realtime/authenticated role;
- no returned participant/private data; boolean only.

Realtime policy on `realtime.messages` calls this helper.

No generic "all authenticated can read Broadcast" policy.

---

# 9. Realtime payload

PASS with tightening.

Only invalidate:

```
{
  type: "session_view_changed",
  session_id,
  revision
}
```

Potentially omit revision where a pending-input-only view change occurs without StreamRevision advance.

Therefore better payload:

```
{
  type: "session_view_changed",
  session_id
}
```

Client always fetches authoritative current PlayerInteractionView.

Do not infer freshness solely from StreamRevision because PlayerView can change at the same canonical revision.

---

# 10. Worker endpoint authentication is underspecified

## Problem

pg_cron/pg_net needs to invoke `outbox-drain`.

Using a broad Supabase server/admin secret merely to wake a worker is excessive authority.

## Correction

Use a dedicated random **worker wake token**:

- 256-bit random secret;
- stored in Supabase Vault for pg_net caller;
- stored in Edge Function secret environment;
- passed in a dedicated header;
- compared before any DB work;
- grants only ability to trigger a drain invocation.

The worker DB credentials themselves remain separate and unavailable to the caller.

A leaked wake token can cause nuisance/DoS but does not grant database/session data authority.

Rotate independently.

---

# 11. API fast-path should not carry worker DB credentials

## Problem

v0.1 says API may call `waitUntil(drainRelevantOutbox())`.

That encourages the API deployment to hold both runtime and worker DB credentials.

## Correction

After commit, API may use `waitUntil` only to send a best-effort authenticated wake HTTP request to `outbox-drain`.

`api` holds:
- runtime DB credentials;
- worker wake token.

`outbox-drain` holds:
- worker DB credentials;
- worker wake token validation secret;
- external presentation/provider secrets as required.

This preserves least privilege.

---

# 12. Cron is wakeup, not queue

PASS.

Use pg_cron/pg_net periodic wakeup as recovery.

Initial 10 seconds remains configurable operational value, not a structural semantic.

Need monitor:
- oldest PENDING outbox age;
- cron invocation failures;
- stale PROCESSING leases.

Correctness stays in engine.outbox.

---

# 13. Provider-specific SQL must not pollute provider-neutral core migrations

## Problem

v0.1 proposes pg_cron/pg_net and Realtime policies but does not distinguish migration ownership.

## Correction

Canonical repository split:

```
db/migrations/
  provider-neutral PostgreSQL engine/access schema

platform/supabase/
  config.toml / deployment config
  migrations-or-sql/
    Realtime authorization
    pg_cron/pg_net wakeup
    Vault integration
    Supabase-specific grants/hooks
```

Both remain canonical GitHub artifacts.

Provider-specific SQL must never redefine engine canonical tables/contracts.

---

# 14. Edge database driver risk

## Finding

Supabase recommends transaction pooling for Edge/serverless.

Current Supabase docs warn that `postgres.js` query pipelining can conflict with shared transaction mode.

Supabase also documents Deno Postgres/Kysely as direct Edge database approaches.

Sources:
- https://supabase.com/docs/guides/database/connecting-to-postgres
- https://supabase.com/docs/guides/database/postgres-js
- https://supabase.com/docs/guides/functions/connect-to-postgres
- https://supabase.com/docs/guides/functions/kysely-postgres

## Correction

Do not freeze postgres.js.

Baseline preference:
- Deno Postgres direct driver;
- Kysely optional query builder;
- explicit integration spike must prove:
  - BEGIN/COMMIT/ROLLBACK;
  - SELECT FOR UPDATE;
  - DEFERRABLE FK behavior;
  - lock order;
  - processing/outbox generation CAS;
  - transaction pooler compatibility.

If this spike fails, move runtime to persistent service or supported connection mode; do not weaken transactions.

---

# 15. Edge CPU limit

PASS as risk, but warning threshold is not a product SLA.

Keep:
- 2s current platform CPU fact;
- 500ms representative p99 as internal reconsideration signal.

Add:
- test resolver independently in Node/Deno;
- synthetic simulations never use hosted Edge request runtime as test harness.

---

# 16. Outbox concurrency

PASS.

Fast wake + cron wake may race.

Accepted Outbox row locks + lease generations make concurrent drains safe.

No additional distributed lock.

---

# 17. AuthSubject deletion/stale binding

## Finding

An access-binding row may outlive deleted auth user.

This is acceptable if authorization always starts from a valid signed JWT.

## Correction

- never authorize by binding row alone;
- JWT validation must succeed first;
- binding lookup uses verified `sub`;
- stale bindings can be reconciled/revoked asynchronously.

Do not add a hard FK from access bindings to Supabase `auth.users`; this reduces provider coupling and avoids auth-schema lifecycle constraints.

---

# 18. Service keys

Current Supabase is moving from legacy `anon/service_role` keys toward publishable/secret keys.

Use current publishable key naming in browser configuration.

Broad Supabase secret keys remain server-only and should not be the ordinary engine database credential.

The engine runtime/worker connect through dedicated PostgreSQL login roles with least privilege.

Sources:
- https://supabase.com/docs/guides/realtime/broadcast
- https://supabase.com/docs/guides/auth/users

---

# 19. PostgreSQL role mapping

Proposed pattern:

## NOLOGIN group roles in migrations

- engine_runtime;
- engine_worker;
- engine_publisher;
- engine_ops_readonly.

## Provider login roles/secrets

Created/configured during platform deployment:

- engine_runtime_login -> member of engine_runtime;
- engine_worker_login -> member of engine_worker.

Passwords/connection strings are secrets and never committed.

Migration owner remains separate.

Exact Supabase role-creation mechanics must be verified in deployment spike.

---

# 20. Postgres target

Supabase's current platform defaults to PostgreSQL 17.

Choose PostgreSQL 17 as major baseline.

Before migration execution:
- create/select actual Supabase project;
- run `show server_version`;
- record actual build/version;
- execute accepted DDL integration suite.

Do not pin canonical mechanics to a Postgres minor version.

---

# 21. Root causes

## A. Identity separation was defined conceptually before its security datastore

The domain distinction was correct, but operational authorization needs explicit ownership and constraints.

## B. "Wake the worker" was conflated with "be the worker"

Separating wake authority from worker DB/external-provider authority substantially reduces secret blast radius.

## C. Provider-neutral and provider-specific migrations were not separated

The core schema is portable; Realtime/cron/Auth glue is not and should be explicit.

## D. Serverless convenience can obscure transaction-driver assumptions

The accepted engine depends on real PostgreSQL transactions/locks. Driver/runtime selection must prove those, not assume them.

---

# 22. Verdict

**IMPLEMENTATION_PLATFORM_v0.1 is NOT ready for acceptance.**

Preserve the Supabase/Postgres17/anonymous-auth/Edge direction, but create v0.2 with the corrections above.

Update ADR-015..017 accordingly.

No executable migrations or application skeleton yet.
