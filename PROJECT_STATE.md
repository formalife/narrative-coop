# PROJECT STATE

**Project:** Narrative Co-op Engine  
**Phase:** Phase 0 — Architecture Discovery / Acceptance Review  
**Active working baseline:** v0.3 — PROPOSED  
**Previous baselines:** v0.1, v0.2 — historical proposals  
**Implementation status:** NOT STARTED  
**Last updated:** 2026-10-02

## Product invariant

Two players inhabit the same world, receive asymmetric information, take asymmetric actions, and produce one canonical timeline through deterministic causal merge.

## Current status

v0.2 underwent a second-pass architecture review and postmortem.

Canonical artifacts:

- `docs/architecture/RED_TEAM_v0.1.md`
- `docs/architecture/ARCHITECTURE_BASELINE_v0.2.md`
- `docs/architecture/POSTMORTEM_v0.2.md`
- `docs/architecture/ARCHITECTURE_BASELINE_v0.3.md`

### Verdict on v0.2

**MODIFY.**

v0.2 preserved the correct architectural center but was not accepted as the final Phase-0 baseline because it left material gaps in:

- authoritative command/idempotency boundary;
- canonical state behavior while a ResolutionWindow is open;
- deterministic event hash scope/identity;
- deterministic numeric semantics;
- epistemic stance semantics;
- Fact lifecycle;
- canonical Scenario Progression versus presentation-only Narrative Direction;
- reliable post-commit handoff;
- causally relevant PresentationRecords;
- pure rule execution;
- historical build/replay compatibility.

v0.3 addresses these issues.

No architecture ADR is ACCEPTED yet.

## Current v0.3 proposal

### Canonical progression

- One canonical ordered event stream per Session.
- Single Canonical Frontier: at most one canonical gameplay progression operation advances a Session at a time.
- Simultaneous ResolutionWindows collect pending input against a fixed canonical base revision.
- Logical scheduler/progression events do not independently mutate through an open simultaneous window.
- Expected-revision concurrency is a safety check rather than the normal gameplay arbitration mechanism.

### Commands and input

- All authoritative inputs enter through idempotent EngineCommands.
- Authenticated/system principal is server-bound.
- ActionSubmission is durable pending input, not canonical world truth.
- Window freeze selects the final submission per ActionSlot.

### Actions and resolution

- ActionDefinition remains scenario authority.
- ExpandedAction is derived server-side.
- Claims are typed and authoritative.
- Interaction Graph/groups support contention, exclusion, interference, dependency, complementarity and order sensitivity.
- Resolver remains deterministic and limited to bounded versioned strategies.

### Progression and narrative

- Scenario Progression is canonical and owns DecisionPoint/window/ending activation.
- Narrative Direction is presentation-only.
- Narrative Realization is noncanonical but exact delivered output is stored in immutable PresentationRecords because it can causally influence later human choices.

### Events/state/replay

- CanonicalEventContent is separated from operational EventRecordMetadata.
- Semantic hashes exclude wall-clock/database metadata.
- Event/action identities used by replay must be deterministic or stored historical inputs.
- Semantic Domain Events remain separate from typed State Mutation IR.
- Canonical numeric mechanics use integers/fixed-point/declared units, not unconstrained floating point.
- Logical time uses deterministic integer representation.
- Epistemics separates Proposition, Fact, Observation, KnowledgeRecord, BeliefRecord, SuspicionRecord and CommunicationClaim.
- Normal Fact lifecycle ends/supersedes validity; it does not erase historical truth.
- Named seeded randomness remains required.
- State, resolver, forensic and narrative replay are distinguished.

### Persistence/reliability

- Selective Event Sourcing for canonical Session history.
- PostgreSQL remains proposed primary persistence.
- Logical CQRS with synchronous critical projection.
- Transactional outbox provides durable post-commit handoff.
- No broker/Redis/microservices required by baseline.

All remain PROPOSALS until explicitly accepted and recorded via ADR.

## Immediate next work

1. Review/accept or amend Architecture Baseline v0.3.
2. Promote accepted structural decisions into ADRs.
3. Mark the accepted Phase-0 baseline.
4. Only then design:
   - concrete domain/data model;
   - command/action/event/mutation contracts;
   - SQL schema;
   - implementation monorepo skeleton;
   - resolver test harness;
   - tiny disposable technical micro-scenario.

## Highest-priority ADRs after acceptance

- ADR-001 Selective Event Sourcing scope.
- ADR-002 Session stream + Single Canonical Frontier.
- ADR-003 PostgreSQL persistence baseline.
- ADR-004 Componentized Entity Model.
- ADR-005 EngineCommand + ActionSubmission authority boundary.
- ADR-006 Claim/Interaction/Resolver model.
- ADR-007 Semantic Domain Event + State Mutation IR.
- ADR-008 Epistemic model.
- ADR-009 Temporal/Scheduler model.
- ADR-010 Scenario compilation + pure deterministic rules.
- ADR-011 Versioning, canonical serialization, hashing and replay.
- ADR-012 Scenario Progression / Narrative Direction / Realization split.
- ADR-013 Logical CQRS + synchronous critical projection + transactional outbox.
- ADR-014 LLM runtime boundaries.
- ADR-015 Guest identity/participation baseline (later, before implementation).

## Decisions intentionally not frozen

- exact Action tag vocabulary;
- Claim selector syntax;
- exact Mutation IR target representation;
- exact fixed-point scales;
- PRNG implementation;
- final Scenario DSL syntax;
- free-text action confirmation policy;
- snapshot frequency;
- SQL table/index design;
- detailed relationship/goal/commitment structures;
- Scenario Studio UI;
- NPC agent architecture;
- vector/graph databases;
- Redis/Durable Objects;
- microservices;
- payment provider;
- retention periods;
- recap video pipeline.

## Canonical working rule

GitHub is the canonical technical source of truth.

Never silently convert a PROPOSAL into a DECISION.

When a structural decision is explicitly accepted:
1. create/update ADR;
2. mark ADR ACCEPTED;
3. update active baseline status;
4. update this PROJECT_STATE.
