# RED-TEAM — Implementation Platform Baseline v0.3

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/architecture/IMPLEMENTATION_PLATFORM_v0.3.md`  
**Reviewed ADRs:** ADR-015, ADR-016, ADR-017 — PROPOSED  
**Verdict:** READY FOR ACCEPTANCE

## 1. Accepted-architecture compatibility

PASS.

The platform preserves:
- one PostgreSQL canonical authority;
- Single Canonical Frontier;
- event/replay semantics;
- LLM exclusion from canonical mechanics;
- PostgreSQL Outbox as durable continuation authority.

Railway stores no canonical state.

## 2. Service privilege separation

PASS.

`engine-api` receives only the runtime DB credential and public Auth verification metadata.

`engine-worker` receives only the worker DB credential and worker-only presentation-provider configuration.

Neither normal service receives:
- migration-owner credential;
- Supabase secret/service-role key.

The API remains an authoritative trust boundary because its runtime DB role must be able to commit validated engine transitions.

## 3. Database connectivity

PASS with deployment verification.

Proposed path:
- Supabase PostgreSQL 17;
- shared session pooler;
- custom LOGIN role per service;
- small bounded application pools;
- database SSL enforcement;
- certificate/hostname verification.

Direct PostgreSQL connection remains an optional tested optimization.

## 4. JWT / access authorization

PASS.

Normal command path:
1. verify Supabase access-token signature and required claims;
2. obtain AuthSubject from `sub`;
3. require ACTIVE `access.session_principal_bindings`;
4. derive ParticipantRef server-side.

Binding revocation therefore denies engine access independently of ordinary JWT lifetime.

Asymmetric signing keys are preferred. A verified Auth-server fallback may use the user token plus publishable key if required; no admin key is needed.

## 5. Participant and invite races

PASS at design level.

The future access schema must enforce:
- one ACTIVE AuthSubject -> at most one ParticipantRef per Session;
- one ACTIVE ParticipantRef -> at most one AuthSubject;
- one active invite per target slot by default.

Invite claim and canonical ParticipantBound transition commit in one DB transaction.

## 6. Anonymous guest lifecycle

PASS with explicit MVP limitation.

Clearing local credentials/signing out/device switching may lose anonymous access.

Identity linking to the same Supabase user does not change ParticipantRef or canonical history.

Anonymous cleanup must skip users with ACTIVE bindings to nonterminal/resumable Sessions.

## 7. Worker / Outbox recovery

PASS.

Worker availability is not durability.

Outbox row locking, leases and lease-generation fencing cover:
- process restart;
- repeated worker attempts;
- multiple worker replicas;
- external-generation retries.

If worker is unavailable, presentation/realtime latency increases but committed canon remains intact.

## 8. Realtime

PASS.

Realtime remains invalidation only.

A narrow Supabase-specific database wrapper around `realtime.send`:
- accepts SessionId only;
- constructs fixed private topic/event/payload;
- has no generic arbitrary-message API;
- is executable only by the worker role.

Private subscription authorization resolves Auth identity to ACTIVE access binding.

Client always fetches current PlayerInteractionView through API.

## 9. Supabase Edge exclusion

PASS and justified.

Current Supabase docs state hosted Edge Functions receive default project DB/secret variables. This is broader than the service-specific privilege model required by the accepted schema.

Keeping Supabase for PostgreSQL/Auth/Realtime while placing authoritative compute in separately scoped persistent services restores the intended process boundary.

## 10. Railway fit

PASS as proposed compute host.

Current Railway documentation supports:
- separate persistent API and worker services;
- service-scoped variables;
- sealed values;
- custom start commands;
- restart policies;
- API healthcheck deployments.

Start with one API instance and one worker instance.

No sticky Session authority exists in process memory, so later scaling remains compatible with accepted DB locking/fencing.

## 11. Cross-provider transport

PASS with deployment requirement.

Place Railway services near the selected Supabase region.

Enable PostgreSQL SSL enforcement and use verified TLS.

Do not start with active-active multi-region authoritative API.

## 12. Provider portability

PASS.

Railway can be replaced without canonical-data migration.

Supabase Auth/Realtime can be replaced with operational migration while ParticipantRef/Character history remains stable.

Replacing Supabase PostgreSQL is more expensive but schema remains ordinary PostgreSQL.

## 13. Canonical replay without runtime providers

PASS.

State replay requires retained Scenario Bundle + SessionGenesis + SessionEventStream, not Railway/Auth/Realtime services.

## 14. Deployment checks required after acceptance

Before executable migrations/application implementation is declared ready:

1. create/select Supabase project;
2. record actual PostgreSQL server version;
3. enable/test DB SSL enforcement;
4. verify JWT signing/validation path;
5. create temporary custom runtime/worker LOGIN roles;
6. connect from selected Railway region through session pooler;
7. test transactions, row locks, SKIP LOCKED, deferred constraints and fencing;
8. verify password rotation/reconnect behavior;
9. audit API/worker variables for forbidden broad/admin credentials;
10. test private Realtime subscription authorization;
11. test narrow database Realtime wrapper;
12. measure cross-provider transaction latency.

Failure of these checks triggers platform/schema review before production migration rollout.

## 15. Residual risks

- second provider adds deployment/observability surface;
- cross-provider latency must be measured;
- idle Outbox polling must use backoff;
- runtime service code remains part of the trusted computing base;
- hosted-platform behavior can change and should be rechecked during implementation.

These do not justify another platform version before implementation spikes.

## 16. Verdict

**IMPLEMENTATION_PLATFORM_v0.3 is technically READY FOR ACCEPTANCE.**

No accepted ADR-001..014, Domain Contract, Domain Model, Persistence Model or PostgreSQL Schema requires supersession.

After explicit acceptance:
1. mark platform v0.3 and ADR-015..017 ACCEPTED;
2. design/red-team the concrete `access` schema;
3. perform Supabase/Railway connectivity/security spikes;
4. only then generate executable migrations;
5. create implementation monorepo skeleton;
6. prioritize deterministic resolver/replay tests before product UI work.
