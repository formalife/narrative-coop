# Domain Contracts v0.2 Acceptance

**Date:** 2026-10-02  
**Decision:** ACCEPTED  
**Accepted contract baseline:** DOMAIN_CONTRACTS_v0.2  
**Governing architecture:** Architecture Baseline v0.3 — ACCEPTED

## Acceptance

Domain Contracts v0.2 are accepted as the binding logical runtime/domain contract baseline.

The accepted contract boundary includes:

- semantic EngineCommand idempotency;
- resumable CommandProcessingRecord;
- one CanonicalTransitionRecord per canonical event batch;
- frozen StreamRevision convention;
- canonical ResolutionWindow vs pending WindowInputGate separation;
- FrozenInputSet and ResolutionAttempt semantics;
- server-authoritative ActionDefinition/ExpandedAction;
- Submission Eligibility vs ResolutionRequirement;
- Claims / Interaction Graph / aggregate interaction resolution;
- named deterministic RNG derivation;
- BoundaryOrderingPolicy;
- CanonicalStateContent / SessionStateProjection separation;
- State Mutation IR v0.2;
- runtime PropositionValue / PropositionKey;
- epistemic records and communication reception separation;
- ScheduledEffect semantics;
- DecisionPointDefinition / Instance separation;
- bounded ProgressionEvaluation;
- PlayerInteractionView / InteractionSurface;
- PresentationPlan / PresentationRecord / generation-context pinning;
- transactional outbox and stale-delivery guard;
- canonical commit atomicity.

## Still open

Acceptance does not freeze:

- opaque ID encoding;
- Claim.scope concrete syntax;
- Mutation target concrete syntax;
- event-family registry;
- scenario-specific fixed-point scales/units;
- PRNG implementation;
- resolver strategy parameter schemas;
- concrete Relationship/Goal/Commitment domain shapes;
- authoring DSL;
- SQL tables/indexes;
- snapshot policy;
- retention;
- PlayerInteractionView caching/materialization.

## Change control

If Domain Model, persistence or implementation design conflicts with an accepted contract:

1. identify the conflict;
2. do not implement around it silently;
3. propose a contract revision or superseding ADR where necessary;
4. red-team;
5. obtain explicit acceptance before implementation.
