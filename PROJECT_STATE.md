# PROJECT STATE

**Project:** Narrative Co-op Engine  
**Architecture phase:** Phase 0 COMPLETE  
**Accepted architecture baseline:** v0.3 — ACCEPTED  
**Current work:** Contract & Domain Model Design  
**Implementation status:** NOT STARTED  
**Last updated:** 2026-10-02

## Product invariant

Two players inhabit the same world, receive asymmetric information, take asymmetric actions, and produce one canonical timeline through deterministic causal merge.

## Accepted architecture

Architecture Baseline v0.3 is the binding Phase-0 architecture.

Canonical baseline:
- `docs/architecture/ARCHITECTURE_BASELINE_v0.3.md`

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

## Still PROPOSED / not frozen

- ADR-015 Guest identity / participation baseline.
- exact Action tag/family vocabulary;
- Claim selector syntax;
- exact State Mutation IR target encoding;
- exact fixed-point scales;
- exact PRNG;
- final Scenario DSL syntax;
- natural-language confirmation policy;
- snapshot frequency;
- concrete SQL tables/indexes;
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

Translate the accepted architecture into explicit contracts and invariants before writing production code.

Current contract proposal:
- `docs/domain/DOMAIN_CONTRACTS_v0.2.md` — PROPOSED

Historical contract proposal:
- `docs/domain/DOMAIN_CONTRACTS_v0.1.md`

Contract postmortem:
- `docs/domain/POSTMORTEM_DOMAIN_CONTRACTS_v0.1.md`

v0.1 was NOT accepted. The postmortem found contract-level ambiguities around window freezing, input/deadline races, same-window dependencies, generic canonical transition ownership, canonical collection hashing, state-hash self-reference, dynamic propositions, progression convergence and presentation/mechanical separation.

v0.2 addresses those findings without changing any ACCEPTED ADR.

The next checkpoint is an acceptance review/red-team of DOMAIN_CONTRACTS_v0.2. Do not proceed to DOMAIN_MODEL or SQL schema until the contract boundary is accepted.

Priority order:

1. Review and accept/amend concrete domain contracts:
   - EngineCommand;
   - ActionSubmission;
   - ActionDefinition;
   - ExpandedAction;
   - Claim;
   - InteractionEdge/InteractionGroup;
   - ResolutionWindow;
   - AdmissionResult/ActionOutcome;
   - ResolutionPlan/ResolutionRecord;
   - CanonicalEventContent/EventRecordMetadata;
   - State Mutation IR;
   - Proposition/Fact/Observation/Knowledge/Belief/Suspicion/CommunicationClaim;
   - ScheduledEffect;
   - Scenario Progression structures;
   - PresentationPlan/PresentationRecord;
   - Outbox item;
   - VersionManifest.

2. Define lifecycle state machines and cross-contract invariants.

3. Define the concrete domain/data model.

4. Define SQL schema and migration strategy.

5. Create implementation monorepo skeleton only after the contracts above are sufficiently stable.

6. Build resolver/state-transition/property-based test harness.

7. Build the disposable technical micro-scenario proving causal merge, asymmetric knowledge, delayed consequences and deterministic replay.

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
