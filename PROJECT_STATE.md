# PROJECT STATE

**Project:** Narrative Co-op Engine  
**Phase:** Phase 0 — Architecture Discovery  
**Active working baseline:** v0.2 — PROPOSED  
**Previous baseline:** v0.1 — historical proposal  
**Implementation status:** NOT STARTED  
**Last updated:** 2026-10-02

## Product invariant

Two players inhabit the same world, receive asymmetric information, take asymmetric actions, and produce one canonical timeline through deterministic causal merge.

## Current status

Architecture Baseline v0.1 has been red-teamed.

Canonical review:
- `docs/architecture/RED_TEAM_v0.1.md`

Current working proposal:
- `docs/architecture/ARCHITECTURE_BASELINE_v0.2.md`

No architecture ADR is ACCEPTED yet.

Chat discussion and PROPOSED baseline text are not binding decisions until explicitly accepted and promoted into ADRs.

## Material changes introduced in v0.2

- Split client `ActionSubmission` from server-derived `ExpandedAction`.
- Claims, preconditions and canonical effects are derived from immutable ActionDefinitions, not authored by clients.
- Action-family taxonomy no longer drives mechanics.
- Claims gained explicit access modes.
- Conflict Graph became a typed Interaction Graph.
- Admission validity was separated from in-world ActionOutcome.
- Simultaneous windows use set/snapshot semantics independent of network arrival order.
- Resolver uses bounded, versioned strategies rather than an arbitrary scripting/constraint engine.
- Canonical events use finite `event_family` + precise `event_code`.
- Event visibility is no longer a generic event-envelope flag; epistemics are explicit canonical state.
- Semantic Domain Events are separated from a typed State Mutation IR.
- Randomness uses stable named draw scopes rather than sequential call position.
- State/event hashes require canonical serialization.
- CQRS distinguishes synchronous gameplay-critical projection from laggable secondary projections.
- Narrative Director decisions affecting gameplay are deterministic/versioned; only realization wording may be noncanonical.
- Wall-clock deadlines are triggers that create recorded events; historical replay never re-runs the historical clock.
- Target addressability became an explicit domain/security invariant.

## Current proposed architecture

- Session is the runtime consistency boundary.
- One ordered canonical event stream per Session.
- Selective Event Sourcing for canonical session history only.
- PostgreSQL remains the proposed primary persistence baseline.
- CQRS is logical; no distributed read/write infrastructure by default.
- Current SessionState is a synchronous rebuildable projection.
- Componentized Entity Model rather than heavyweight ECS.
- Immutable compiled Scenario Bundle.
- Minimal client ActionSubmission → authoritative ExpandedAction.
- Typed claims + Interaction Graph.
- Deterministic bounded resolver strategies.
- Semantic Domain Event + typed State Mutation IR.
- Explicit Fact / Observation / Knowledge / Belief / Communication Claim semantics.
- Separate engine order, logical time and wall clock.
- Canonical Scheduler for logical delayed consequences.
- Named seeded versioned randomness.
- Deterministic Simulation and Narrative Director.
- Optional/noncanonical Narrative Realization.
- Version pinning + canonical hashes + state/resolver replay.
- Browser cannot directly mutate canonical state.
- LLM remains outside canonical state transitions.

All items above remain PROPOSALS until accepted via ADR.

## Immediate next work

1. Review Architecture Baseline v0.2 with the user.
2. Accept, modify or reject the structural decisions.
3. Create/mark the corresponding ADRs only for decisions explicitly accepted.
4. Produce an ACCEPTED Phase-0 architecture baseline or a v0.3 proposal if material corrections remain.
5. Only after the governing contracts are accepted:
   - concrete domain/data model;
   - event/action/mutation schemas;
   - DB schema;
   - repository implementation skeleton;
   - resolver test harness;
   - tiny disposable technical micro-scenario.

## Highest-priority decisions to freeze before implementation

- Event Sourcing scope and Session consistency boundary.
- ActionSubmission / ActionDefinition / ExpandedAction contract.
- Claim + Interaction Graph model.
- Resolver determinism and strategy boundary.
- Domain Event envelope/taxonomy.
- State Mutation IR boundary.
- Epistemic model.
- Temporal model.
- Scenario bundle/versioning.
- Replay/hashing contract.
- Simulation / Director / Realization boundary.

## Decisions intentionally not frozen

- exact action-family/tag vocabulary;
- exact claim-selector/path syntax;
- exact Mutation IR target encoding;
- exact PRNG;
- final Scenario DSL syntax;
- natural-language confirmation policy;
- snapshot frequency;
- concrete DB schema;
- exact component schema representation;
- Scenario Studio UI;
- NPC agent implementation;
- vector/graph databases;
- Redis/Durable Objects;
- microservices;
- payment provider;
- retention periods;
- recap video pipeline.

## Canonical working rule

GitHub is the canonical technical source of truth.

When a structural decision is explicitly accepted:
1. create/update its ADR;
2. mark it ACCEPTED;
3. update the active architecture baseline;
4. update this PROJECT_STATE file.

Never silently convert a PROPOSAL into a DECISION.
