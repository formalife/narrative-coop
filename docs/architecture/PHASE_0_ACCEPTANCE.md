# Phase 0 Architecture Acceptance

**Project:** Narrative Co-op Engine  
**Date:** 2026-10-02  
**Decision:** ACCEPTED  
**Accepted baseline:** Architecture Baseline v0.3

## Acceptance statement

Architecture Baseline v0.3 is accepted as the governing Phase-0 architecture for the Narrative Co-op Engine.

The acceptance covers the structural boundaries recorded in ADR-001 through ADR-014.

This acceptance authorizes the project to move from architecture discovery into concrete contract/domain/data-model design.

It does **not** authorize silent changes to accepted architectural boundaries during implementation.

## Accepted decisions

1. Selective Event Sourcing for canonical Session history.
2. One Session canonical stream and Single Canonical Frontier.
3. PostgreSQL primary persistence baseline.
4. Componentized Entity Model.
5. Idempotent EngineCommand boundary and server-authoritative Action expansion.
6. Typed Claims, Interaction Graph and bounded deterministic resolver.
7. Semantic Domain Events separated from typed State Mutation IR.
8. Explicit epistemic model separating Fact, Observation, Knowledge, Belief, Suspicion and CommunicationClaim.
9. Separation of engine order, logical time and wall clock with canonical scheduler semantics.
10. Immutable CompiledScenarioBundle with pure deterministic runtime rules.
11. Version pinning, deterministic numeric/serialization/hash rules and replay modes.
12. Canonical Scenario Progression separated from Narrative Direction and Narrative Realization.
13. Logical CQRS, synchronous critical projection and Transactional Outbox.
14. LLM exclusion from canonical mechanics.

## Explicitly not accepted/frozen

The following remain open and may be decided during later design without superseding the accepted baseline unless they conflict with an ADR:

- guest identity/participation model;
- exact Action tag/family vocabulary;
- Claim selector syntax;
- exact Mutation IR target encoding;
- exact fixed-point scales;
- exact PRNG;
- final Scenario DSL syntax;
- free-text confirmation policy;
- snapshot frequency;
- SQL table/index layout;
- detailed Relationship/Goal/Commitment schemas;
- Scenario Studio UI;
- NPC agent architecture;
- vector/graph databases;
- Redis/Durable Objects;
- microservices;
- payments;
- retention periods;
- recap/video pipeline.

## Change control

If later contract or implementation work conflicts with an ACCEPTED ADR:

1. identify the conflict explicitly;
2. stop treating it as a local implementation choice;
3. write a superseding ADR proposal;
4. red-team it;
5. obtain explicit acceptance;
6. only then update baseline and implementation.

## Next authorized work

Proceed to:

1. concrete domain contracts;
2. invariants/lifecycle state machines;
3. data model;
4. SQL schema;
5. implementation skeleton;
6. deterministic resolver/replay tests;
7. disposable technical micro-scenario.
