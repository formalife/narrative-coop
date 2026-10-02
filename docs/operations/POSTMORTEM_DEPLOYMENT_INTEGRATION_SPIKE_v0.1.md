# POSTMORTEM — Deployment Integration Spike v0.1

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/operations/DEPLOYMENT_INTEGRATION_SPIKE_v0.1.md`  
**Governing architecture:** Architecture Baseline v0.5 — ACCEPTED  
**Verdict:** ARCHITECTURE CONFIRMED / IMPLEMENTATION GATE INCOMPLETE

## Executive verdict

The live Supabase target validates the core platform assumptions:

- PostgreSQL 17 is available and healthy;
- accepted custom-role/least-privilege model can be represented;
- deferred FKs behave as required;
- Supabase Realtime/Auth database primitives exist on the real target;
- the staging project can remain the integration target.

No accepted ADR or architecture baseline requires modification.

The spike is incomplete because the connected Railway account exposes only a personal workspace and no Formalife workspace.

The correct action is **not** to weaken or move the architecture.

The correct action is to finish the same spike once Railway ownership is correct.

---

# 1. What the spike proved

## PostgreSQL target

PASS.

The actual managed target is PostgreSQL 17.11.

This confirms the accepted PG17 baseline on a real project rather than documentation alone.

## Custom DB roles

PASS.

The project allows the accepted pattern:

```
NOLOGIN group role
<- dedicated LOGIN role
```

for API/worker separation.

## Least privilege

PASS.

PostgreSQL grants can distinguish:
- runtime column-level write capabilities;
- worker read-only/noncanonical capabilities.

No platform limitation forced shared service credentials.

## Deferred constraints

PASS.

The target behaves as required for atomic workflows whose referenced row may be inserted later in the same transaction.

## Realtime DB integration surface

PASS.

`realtime.send`, `auth.uid()`, `auth.jwt()` and RLS-enabled `realtime.messages` exist on the target.

The accepted narrow DB wrapper remains implementable.

---

# 2. What the spike did NOT prove

## Concurrent locking

NOT TESTED.

The MCP transport serialized database requests, so attempted parallel calls were not actual concurrent database sessions.

This is a tooling limitation, not a PostgreSQL result.

Required later:
- independent Node/Postgres connections from target compute/test runner.

## SSL enforcement

PARTIAL.

Server reports SSL support enabled.

Still required:
- hosted enforcement confirmation;
- verify-full client connection.

## JWT flow

PARTIAL.

The project/Auth surface exists.

Still required:
- configure/verify anonymous Auth;
- CAPTCHA/Turnstile;
- signing mode/JWKS;
- issue and validate a real anonymous-user access token.

## Supavisor custom-login connection

NOT TESTED.

Creating LOGIN roles inside PostgreSQL succeeded.

Authenticating those roles through the selected session pooler from Railway remains required.

---

# 3. Railway ownership blocker

## Observation

Connected Railway account exposes:
- personal workspace only;
- one unrelated staging project.

No Formalife workspace is visible.

## Why this matters

The project has already established source-of-truth/ownership discipline around Formalife.

Creating production-bound Narrative Co-op infrastructure under a personal Railway workspace would create:
- ownership debt;
- billing/admin ambiguity;
- future transfer risk;
- unnecessary environment migration.

## Decision

Do not create Narrative Co-op Railway infrastructure in the personal workspace.

This is the correct stop condition.

---

# 4. Supabase ownership result

PASS.

Supabase connector exposes the Formalife organization.

The staging project was created there directly.

This matches the intended organizational ownership.

---

# 5. Postmortem on the test process

## What worked

- accepted architecture gave a clear test matrix;
- spike used temporary reversible DB objects;
- no migrations were prematurely applied;
- invalid concurrency evidence was rejected rather than rationalized as a pass;
- connector account ownership was checked before resource creation.

## What failed

The integration gate assumed both provider accounts would already expose equivalent organization ownership.

That assumption was false.

## Correction

Deployment readiness check must begin with:

1. organization/workspace identity;
2. billing/ownership scope;
3. only then resource creation.

Add this ordering to future infrastructure runbooks.

---

# 6. Architecture impact

No change to:

- ADR-015;
- ADR-016;
- ADR-017;
- Architecture Baseline v0.5;
- PostgreSQL Schema v0.5;
- Access Schema v0.3.

The current architecture remains coherent.

---

# 7. Implementation gate status

## Completed

- Supabase Formalife staging target exists.
- PostgreSQL major/build recorded.
- custom role creation tested.
- grant separation tested.
- deferred FK behavior tested.
- Realtime DB primitives verified.

## Pending

- correct Formalife Railway workspace/account visibility;
- Railway project creation;
- engine-api/engine-worker service creation;
- cross-provider session-pooler connection;
- verify-full TLS;
- concurrent lock/SKIP LOCKED/fencing tests;
- JWT anonymous-user flow;
- private Realtime subscription/send integration;
- latency measurement;
- secret-scope audit.

---

# 8. Decision

**Do not generate production-ready migrations yet.**

The next step is to make the correct Formalife Railway workspace available to the connected Railway account/plugin, then resume the deployment integration spike as v0.2.

Once v0.2 passes:
- generate canonical provider-neutral migrations;
- generate Supabase-specific platform SQL;
- run migration/advisor/integration suite;
- then scaffold implementation runtime.

