# DEPLOYMENT INTEGRATION SPIKE v0.1

**Status:** PARTIAL / BLOCKED  
**Date:** 2026-10-02  
**Governing architecture:** Architecture Baseline v0.5 — ACCEPTED  
**Governing platform:** Implementation Platform v0.3 — ACCEPTED  
**Governing schemas:** PostgreSQL Schema v0.5 + Access Schema v0.3 — ACCEPTED

---

# 1. Purpose

Verify the accepted implementation platform against a real Supabase target and the connected Railway account before executable migrations.

Required gates:

- target PostgreSQL version;
- SSL capability;
- custom least-privilege roles;
- relational primitives used by the schema;
- Realtime database primitives;
- JWT/Auth target availability;
- Railway workspace/project context;
- cross-provider pooler/TLS connection;
- live concurrency/fencing tests.

---

# 2. Supabase target created

Organization:
- Formalife

Project:
- `narrative-coop-staging`

Region:
- `eu-central-1`

Status:
- `ACTIVE_HEALTHY`

Creation cost reported by provider:
- 0 monthly

Project reference:
- `ylqipkcqypmtsnbflkpv`

The project is intentionally a staging/integration target, not production.

---

# 3. PostgreSQL target verification

Provider project metadata reports:

- PostgreSQL engine: 17
- provider build: 17.11.0.002
- GA release channel

Direct SQL reports:

```
server_version = 17.11
server_version_num = 170011
ssl = on
database = postgres
```

Result:
**PASS** for PostgreSQL 17 target compatibility.

Important:
`ssl=on` proves TLS support, not that the project's external connection policy currently enforces TLS or that an external client has successfully performed hostname/certificate verification.

External `verify-full` remains a separate deployment test.

---

# 4. Publishable client key model

The project exposes:

- modern publishable key;
- legacy anon key.

The browser implementation should use the modern publishable key.

No Supabase secret/service-role key is needed by the accepted player-web/API architecture.

Result:
**PASS** for publishable-key availability.

---

# 5. PostgreSQL extension/platform availability

The target includes the relevant managed platform facilities, including availability of:

- `pg_stat_statements` installed;
- `pgcrypto` installed;
- `supabase_vault` installed;
- `uuid-ossp` installed;
- `pg_cron`, `pg_net`, `pgmq` available but not selected by architecture.

No optional extension is required by canonical engine correctness.

Result:
**PASS**.

---

# 6. Custom PostgreSQL role model

Temporary spike roles were created:

```
spike_nce_runtime      NOLOGIN
spike_nce_worker       NOLOGIN
spike_nce_runtime_login LOGIN IN ROLE spike_nce_runtime
spike_nce_worker_login  LOGIN IN ROLE spike_nce_worker
```

Database verification showed:

- runtime login can LOGIN;
- runtime login is member only of runtime group;
- worker login can LOGIN;
- worker login is member only of worker group.

Result:
**PASS** for ability to materialize the accepted group-role/login-role model.

The spike roles were deleted after testing.

---

# 7. Least-privilege GRANT behavior

Temporary table privilege test:

Runtime role:
- schema USAGE: true;
- table SELECT: true;
- selected-column UPDATE: true.

Worker role:
- SELECT: true;
- table UPDATE: false;
- selected-column UPDATE: false.

Result:
**PASS** for the privilege primitives required by PostgreSQL Schema v0.5.

---

# 8. DEFERRABLE foreign-key behavior

Test:

1. begin transaction;
2. insert child row referencing not-yet-inserted parent;
3. insert parent;
4. commit.

With `DEFERRABLE INITIALLY DEFERRED`, commit succeeded and referential integrity was valid.

Result:
**PASS**.

This validates the mechanism used by canonical transition/event and atomic access/participation workflows.

---

# 9. Access-lock concurrency test

Required invariant:

`FOR SHARE` authorization lock must conflict with binding revocation UPDATE.

Attempt:
- issue two Supabase MCP SQL calls concurrently.

Observed:
- connector/tool transport serialized the calls instead of maintaining simultaneous PostgreSQL sessions.

Therefore the observed timing did not test the intended race.

Result:
**BLOCKED / NOT TESTED**.

Do not interpret this as either PASS or FAIL.

A real two-connection integration test from the deployed runtime remains mandatory.

---

# 10. SKIP LOCKED / fencing live concurrency

The target PostgreSQL version supports the accepted SQL primitives by PostgreSQL contract, but no valid simultaneous external-session test was possible through the current Supabase connector.

Result:
**BLOCKED / NOT TESTED LIVE**.

Required later:
- two independent client connections from Railway/test runner;
- Outbox row claim race;
- stale lease-generation completion rejection.

---

# 11. Supabase Realtime database primitives

Real target introspection confirms:

```
realtime.send(
  payload jsonb,
  event text,
  topic text,
  private boolean
) -> void
```

exists.

Also present:

```
auth.uid() -> uuid
auth.jwt() -> jsonb
```

`realtime.messages`:
- exists;
- has RLS enabled;
- owner is `supabase_realtime_admin`.

The table includes:
- topic;
- payload;
- event;
- private;
- id;
- operational timestamp fields.

Result:
**PASS** for the database primitives assumed by the accepted Realtime wrapper design.

Provider-specific RLS/helper functions are not yet installed.

---

# 12. Auth/JWT status

A Supabase Auth project exists and exposes the normal Auth schema.

The accepted browser/API architecture requires:

- anonymous Auth enabled;
- preferred asymmetric JWT signing/JWKS validation;
- CAPTCHA/Turnstile;
- API verification of actual issued access tokens.

Those project-level settings were not fully verifiable/configurable through the current spike tool path.

No real anonymous user/token was created during this spike.

Result:
**PARTIAL / CONFIGURATION TEST PENDING**.

---

# 13. SSL enforcement / verify-full

Database reports:

`ssl = on`.

Still unverified:

- hosted-project SSL enforcement switch/state;
- actual Supavisor hostname;
- external certificate chain;
- successful Node/Postgres connection using `verify-full`.

Result:
**PARTIAL**.

This must be tested from the target compute environment.

---

# 14. Railway account/workspace discovery

Connected Railway identity is available.

Visible workspace:
- personal workspace only.

Visible existing project:
- `formalife-twenty-staging`, unrelated to Narrative Co-op Engine.

No Formalife team/workspace is visible through the connected Railway account.

Result:
**BLOCKED** for creating the canonical Narrative Co-op Railway project.

No Railway project/service was created in the personal workspace to avoid placing project infrastructure in the wrong ownership scope.

---

# 15. Cross-provider tests blocked

Because the proper Railway workspace is unavailable, the following could not be executed:

- create Narrative Co-op Railway project;
- create `engine-api` and `engine-worker`;
- select Railway region near Supabase `eu-central-1`;
- inject separate runtime/worker DB credentials;
- connect through Supabase session pooler;
- test external TLS verify-full;
- test custom LOGIN authentication through pooler;
- test two real concurrent DB sessions;
- measure Railway -> Supabase latency;
- test password rotation/reconnect;
- audit service environment secret separation.

Result:
**BLOCKED**.

---

# 16. Cleanup

All temporary database spike artifacts were removed:

- `spike_nce` schema;
- temporary runtime/worker group roles;
- temporary runtime/worker login roles.

No engine/access migration was applied.

The staging project itself remains available for subsequent integration work.

---

# 17. Gate result

## PASS

- real Supabase project creation in Formalife;
- region `eu-central-1`;
- PostgreSQL 17.11 target;
- SSL-enabled PostgreSQL;
- custom LOGIN/group role creation;
- least-privilege GRANT primitives;
- DEFERRABLE FK behavior;
- Realtime/Auth database function availability;
- Realtime messages RLS presence.

## PARTIAL

- Auth/JWT runtime configuration;
- SSL enforcement and external verify-full.

## BLOCKED

- real concurrent connection tests;
- SKIP LOCKED/fencing races;
- Railway service creation;
- Supavisor custom-login external connection;
- latency/region measurement;
- service-secret isolation verification.

---

# 18. Decision

This spike does NOT justify changing Architecture Baseline v0.5.

It also does NOT yet authorize executable migration rollout.

The next integration run must start only after the correct Railway Formalife workspace is available to the connector/account.

