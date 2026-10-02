# PROJECT STATE

**Project:** Narrative Co-op Engine  
**Architecture phase:** Phase 0 COMPLETE  
**Accepted architecture baseline:** v0.5 — ACCEPTED  
**Current work:** Deployment Integration Validation  
**Implementation status:** NOT STARTED  
**Last updated:** 2026-10-02

## Product invariant

Two players inhabit the same world, receive asymmetric information, take asymmetric actions, and produce one canonical timeline through deterministic causal merge.

## Accepted architecture

Architecture Baseline v0.5 is the binding current architecture; v0.3 remains the accepted historical Phase-0 baseline.

Canonical baseline:
- `docs/architecture/ARCHITECTURE_BASELINE_v0.5.md`

Acceptance record:
- `docs/architecture/PHASE_0_ACCEPTANCE.md`

Historical review chain:
- `docs/architecture/RED_TEAM_v0.1.md`
- `docs/architecture/ARCHITECTURE_BASELINE_v0.2.md`
- `docs/architecture/POSTMORTEM_v0.2.md`

## Accepted ADRs

- ADR-001 — Selective Event Sourcing Scope
- ADR-002 — Session Stream and Single Canonical Frontier
- ADR-003 — PostgreSQL Persistence Baseline
- ADR-004 — Componentized Entity Model
- ADR-005 — EngineCommand and ActionSubmission Authority Boundary
- ADR-006 — Claims, Interaction Graph and Deterministic Resolver
- ADR-007 — Semantic Domain Events and State Mutation IR
- ADR-008 — Epistemic Model
- ADR-009 — Temporal and Scheduler Model
- ADR-010 — Compiled Scenario Bundle and Pure Rules
- ADR-011 — Versioning, Canonical Serialization, Hashing and Replay
- ADR-012 — Scenario Progression / Narrative Direction / Realization Separation
- ADR-013 — Logical CQRS / Critical Projection / Transactional Outbox
- ADR-014 — LLM Runtime Boundaries
- ADR-015 — Guest Identity and Session Participation
- ADR-016 — Supabase PostgreSQL / Auth / Realtime Platform Baseline
- ADR-017 — Persistent API / Worker Runtime and Durable Outbox Processing

## Still PROPOSED / not frozen

- exact Action tag/family vocabulary;
- Claim selector syntax;
- exact State Mutation IR target encoding;
- exact fixed-point scales;
- exact PRNG;
- final Scenario DSL syntax;
- natural-language confirmation policy;
- snapshot frequency;
- detailed Relationship/Goal/Commitment schemas;
- Scenario Studio UI;
- NPC agent architecture;
- vector/graph databases;
- Redis/Durable Objects;
- microservices;
- payment provider;
- retention periods;
- recap/video pipeline.

## Current objective

Freeze the minimum implementation platform and access-security boundary required to safely materialize the accepted PostgreSQL schema before writing production code.

Accepted contract baseline:
- `docs/domain/DOMAIN_CONTRACTS_v0.3.md` — ACCEPTED
- `docs/domain/DOMAIN_CONTRACTS_v0.2.md` — SUPERSEDED

Historical contract proposal:
- `docs/domain/DOMAIN_CONTRACTS_v0.1.md`

Contract postmortem:
- `docs/domain/POSTMORTEM_DOMAIN_CONTRACTS_v0.1.md`

v0.1 was NOT accepted. The postmortem found contract-level ambiguities around window freezing, input/deadline races, same-window dependencies, generic canonical transition ownership, canonical collection hashing, state-hash self-reference, dynamic propositions, progression convergence and presentation/mechanical separation.

v0.2 addresses those findings without changing any ACCEPTED ADR.

The acceptance review/red-team of DOMAIN_CONTRACTS_v0.2 is complete.

Review:
- `docs/domain/RED_TEAM_DOMAIN_CONTRACTS_v0.2.md`

Verdict:
- DOMAIN_CONTRACTS_v0.2 — ACCEPTED on 2026-10-02.

Domain Model work completed through the current checkpoint.

Historical model:
- `docs/domain/DOMAIN_MODEL_v0.1.md` — NOT ACCEPTED
- `docs/domain/POSTMORTEM_DOMAIN_MODEL_v0.1.md`

Accepted domain model:
- `docs/domain/DOMAIN_MODEL_v0.2.md` — ACCEPTED
- `docs/domain/RED_TEAM_DOMAIN_MODEL_v0.2.md` — PASS

The Domain Model postmortem found additive contract gaps: durable Session seed/bootstrap ownership, explicit information classification/Secret contract, predicate epistemic cardinality and terminal ending/window consistency.

No ACCEPTED ADR requires supersession.

Domain Contracts v0.3, Domain Model v0.2, Persistence Data Model v0.3 and PostgreSQL Schema v0.5 are accepted. Executable migrations remain blocked until the implementation platform/access boundary and deployment spikes are accepted.

Priority order:

1. Implementation Platform v0.3 — ACCEPTED.
2. ADR-015 Guest Identity and Session Participation — ACCEPTED.
3. ADR-016 Supabase PostgreSQL / Auth / Realtime Platform — ACCEPTED.
4. ADR-017 Persistent API / Worker Runtime and Durable Outbox — ACCEPTED.
5. Access Schema v0.3 — ACCEPTED.
6. Deployment Integration Spike v0.1 — PARTIAL / BLOCKED on Railway ownership scope.
7. Resume as v0.2 after Formalife Railway workspace becomes available; then complete TLS/JWT/concurrency/Realtime tests.
8. Only after those gates generate executable `db/migrations/`.
8. Create implementation monorepo skeleton.
9. Build resolver/state-transition/property-based/replay test harness.
10. Build the disposable technical micro-scenario before product UI/story implementation.

## Non-negotiable implementation constraints inherited from v0.3

- One canonical Session event stream.
- Single Canonical Frontier.
- Server-bound authority and idempotent commands.
- Client cannot author claims/effects.
- Deterministic rules, logical time and numeric mechanics.
- Explicit epistemic semantics.
- Canonical Scenario Progression separate from presentation.
- Semantic Domain Events + typed Mutation IR.
- Named seeded randomness.
- Synchronous critical projection + transactional outbox.
- Versioned canonical hashing and replay.
- LLM outside canonical mechanics.

## Canonical working rule

GitHub remains the technical source of truth.

Accepted ADRs may not be silently reinterpreted.

If contract design reveals a conflict with an ACCEPTED ADR:
1. identify the conflict;
2. stop treating the conflicting design as implementation detail;
3. propose a superseding ADR;
4. update baseline/project state only after explicit acceptance.


## Persistence design checkpoint

Persistence design has completed three iterations.

Historical proposals:
- `docs/persistence/PERSISTENCE_MODEL_v0.1.md` — NOT ACCEPTED
- `docs/persistence/POSTMORTEM_PERSISTENCE_MODEL_v0.1.md`
- `docs/persistence/PERSISTENCE_MODEL_v0.2.md` — NOT ACCEPTED
- `docs/persistence/RED_TEAM_PERSISTENCE_MODEL_v0.2.md`

Current proposal:
- `docs/persistence/PERSISTENCE_MODEL_v0.3.md` — ACCEPTED
- `docs/persistence/RED_TEAM_PERSISTENCE_MODEL_v0.3.md` — PASS

v0.3 adds:
- global DB lock hierarchy;
- command-processing fencing generations;
- outbox lease fencing generations;
- ActionSubmission content hashes;
- FrozenInputSet commitment to selected content hashes;
- stable presentation generation/delivery identity;
- fenced presentation result acceptance;
- immutable asset identity;
- event-index metadata repair semantics;
- minimal session_runtime without duplicated canonical convenience fields.

Persistence v0.3 was explicitly ACCEPTED on 2026-10-02.

Concrete PostgreSQL schema design is now authorized. Actual migration files remain blocked until the schema proposal is red-teamed and accepted.


## PostgreSQL schema checkpoint

Persistence Data Model v0.3 is ACCEPTED.

Schema design review chain:

- `docs/persistence/POSTGRESQL_SCHEMA_v0.1.md` — NOT ACCEPTED
- `docs/persistence/POSTMORTEM_POSTGRESQL_SCHEMA_v0.1.md`
- `docs/persistence/POSTGRESQL_SCHEMA_v0.2.md` — NOT ACCEPTED
- `docs/persistence/POSTMORTEM_POSTGRESQL_SCHEMA_v0.2.md`
- `docs/persistence/POSTGRESQL_SCHEMA_v0.3.md` — NOT ACCEPTED
- `docs/persistence/POSTMORTEM_POSTGRESQL_SCHEMA_v0.3.md`
- `docs/persistence/POSTGRESQL_SCHEMA_v0.4.md` — NOT ACCEPTED
- `docs/persistence/POSTMORTEM_POSTGRESQL_SCHEMA_v0.4.md`
- `docs/persistence/POSTGRESQL_SCHEMA_v0.5.md` — ACCEPTED
- `docs/persistence/RED_TEAM_POSTGRESQL_SCHEMA_v0.5.md` — PASS

Key v0.5 corrections include:

- atomic PlayerInteractionView + PENDING PresentationRecord + BUILD Outbox handoff;
- explicit complete hash framing for PlayerView/Plan/output;
- composite Plan -> Presentation integrity;
- composite Outbox -> Presentation/PlayerView/GenerationContext integrity;
- DB-enforced PresentationRecord lifecycle immutability;
- correct PostgreSQL PUBLIC function-EXECUTE default handling;
- provider-neutral `db/migrations/` archive;
- logical DB privilege classes and migration bootstrap ordering.

No accepted ADR, Domain Contract, Domain Model or Persistence Model requires supersession.

### Migration implementation remains blocked

PostgreSQL Schema v0.5 was explicitly ACCEPTED on 2026-10-02.

Executable migrations are not implementation-ready until:

1. Implementation Platform v0.3 and ADR-015..017 are explicitly accepted;
2. additive access-schema design is red-teamed/accepted;
3. the target Supabase PostgreSQL 17 project/version is recorded;
4. custom runtime/worker DB roles are tested through the selected connection path;
5. TLS verify-full, JWT validation and private Realtime authorization are tested;
6. Railway/Supabase region and deployment role mapping are fixed.

Do not create production migration files before these gates.


## Implementation platform checkpoint

Historical proposals:
- `docs/architecture/IMPLEMENTATION_PLATFORM_v0.1.md` — NOT ACCEPTED
- `docs/architecture/POSTMORTEM_IMPLEMENTATION_PLATFORM_v0.1.md`
- `docs/architecture/IMPLEMENTATION_PLATFORM_v0.2.md` — NOT ACCEPTED
- `docs/architecture/POSTMORTEM_IMPLEMENTATION_PLATFORM_v0.2.md`

Current proposal:
- `docs/architecture/IMPLEMENTATION_PLATFORM_v0.3.md` — ACCEPTED
- `docs/architecture/RED_TEAM_IMPLEMENTATION_PLATFORM_v0.3.md` — PASS

Proposed topology:
- Supabase: managed PostgreSQL 17 + Auth + private Realtime.
- Railway: separate persistent `engine-api` and `engine-worker`.
- custom least-privilege PostgreSQL LOGIN role per service.
- no Supabase secret/service-role key in normal API/worker runtime.
- JWT verification via Supabase Auth public signing metadata.
- private Realtime invalidation emitted through a narrow DB wrapper.
- PostgreSQL Outbox remains durable work authority; no Redis/pgmq/broker.

Reason for v0.2 -> v0.3:
current Supabase Edge Functions receive broad default project DB/secret credentials, which conflicts with the accepted service-level privilege separation. Supabase remains the data/auth/realtime platform; authoritative compute moves to scoped persistent services.

No ACCEPTED ADR-001..014 or accepted domain/persistence/schema decision requires supersession.

Implementation Platform v0.3 and ADR-015..017 were explicitly ACCEPTED on 2026-10-02. Access-schema design is now authorized; executable migrations remain blocked until access schema and deployment spikes pass.


## Access schema checkpoint

Accepted prerequisites:
- Architecture Baseline v0.4 — ACCEPTED
- ADR-015 — ACCEPTED
- Implementation Platform v0.3 — ACCEPTED
- PostgreSQL Schema v0.5 — ACCEPTED

Review chain:
- `docs/persistence/ACCESS_SCHEMA_v0.1.md` — NOT ACCEPTED
- `docs/persistence/POSTMORTEM_ACCESS_SCHEMA_v0.1.md`
- `docs/persistence/ACCESS_SCHEMA_v0.2.md` — NOT ACCEPTED
- `docs/persistence/POSTMORTEM_ACCESS_SCHEMA_v0.2.md`
- `docs/persistence/ACCESS_SCHEMA_v0.3.md` — ACCEPTED
- `docs/persistence/RED_TEAM_ACCESS_SCHEMA_v0.3.md` — PASS

Key v0.3 decisions:
- AuthSubject remains noncanonical.
- one lifetime binding per Session/AuthSubject in MVP.
- one lifetime binding per Session/ParticipantRef in MVP.
- no access recovery/rebind in MVP schema.
- authorized private/write transactions revalidate and lock ACTIVE binding.
- FOR SHARE authorization locks linearize against revocation.
- invite creator/claim references access binding rather than duplicating Participant/transition data.
- invite plaintext is never persisted.
- invite expiry is evaluated at a defined locked wall-clock point.
- SQL FKs prove identity/context; semantic validators prove ParticipantBound/slot meaning.
- Realtime revocation is not treated as immediate security authority because authorization may be cached; Realtime payload remains content-free invalidation.

Access Schema v0.3 was explicitly ACCEPTED on 2026-10-02. Executable migrations remain blocked until the deployment integration spike passes.


## Deployment integration checkpoint

Access Schema v0.3 was explicitly ACCEPTED on 2026-10-02.

Current accepted architecture baseline:
- `docs/architecture/ARCHITECTURE_BASELINE_v0.5.md` — ACCEPTED.

Live integration artifacts:
- `docs/operations/DEPLOYMENT_INTEGRATION_SPIKE_v0.1.md` — PARTIAL / BLOCKED.
- `docs/operations/POSTMORTEM_DEPLOYMENT_INTEGRATION_SPIKE_v0.1.md`.

### Supabase result

A Formalife staging project now exists:
- project name: `narrative-coop-staging`;
- region: `eu-central-1`;
- PostgreSQL target: 17.11 / provider build 17.11.0.002;
- status: healthy.

Verified on real target:
- custom LOGIN/group role model;
- least-privilege grants;
- deferred FK behavior;
- Realtime/Auth database primitives;
- SSL support enabled.

Still pending:
- actual external verify-full connection;
- anonymous Auth/JWT live flow;
- two-session concurrency/SKIP LOCKED/fencing;
- private Realtime end-to-end.

### Railway result

Connected Railway account currently exposes only a personal workspace and an unrelated existing project.

No Formalife team/workspace is visible.

Decision:
- do NOT create Narrative Co-op Railway infrastructure under the personal workspace;
- resume deployment spike when Formalife Railway ownership scope is available.

### Migration gate

Executable production-ready migrations remain BLOCKED until deployment spike v0.2 completes the pending cross-provider tests.

No accepted architecture/ADR/schema decision requires modification.
