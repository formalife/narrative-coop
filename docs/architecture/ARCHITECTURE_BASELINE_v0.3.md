# ARCHITECTURE BASELINE v0.3

**Status:** ACCEPTED  
**Date:** 2026-10-02  
**Accepted Phase-0 baseline; supersedes:** v0.2 proposal  
**Implementation:** NOT STARTED

## 1. Architectural thesis

The product is a deterministic two-player narrative simulation with:

- asymmetric information;
- asymmetric agency;
- simultaneous/sequential semantic actions;
- deterministic causal merge;
- one canonical Session timeline.

The distinguishing mechanic is:

> **ASYMMETRIC AGENCY + CAUSAL MERGE**

The runtime must remain mechanically correct with all LLM generation disabled.

The system is not:

- a branching-story tree;
- a two-player chatbot;
- an LLM Game Master;
- a universal simulator;
- a distributed multi-agent platform.

---

# 2. Canonical authority model

There are four distinct categories of data.

## 2.1 Canonical world/session truth

Authoritative fictional state and canonical gameplay progression.

Examples:

- entity/component state;
- facts/propositions where explicitly modeled;
- relationships;
- goals/commitments;
- logical time;
- scheduled consequences;
- active/closed ResolutionWindows;
- activated DecisionPoints;
- ending/session state.

Source of historical truth:

> ordered canonical Session event stream.

## 2.2 Authoritative pending input

Inputs not yet part of world truth.

Examples:

- ActionSubmissions while a window remains open;
- command journal/idempotency records.

These may be persisted, replaced or rejected without becoming canonical world events.

Once a window is frozen, the selected input set is recorded in its ResolutionRecord.

## 2.3 Rebuildable projections

Examples:

- current SessionState;
- player view;
- recap;
- admin debug view;
- analytics summaries.

They never override canonical history.

## 2.4 Immutable interaction artifacts

Not world truth, but causally relevant to human play:

- exact player-facing PresentationPlan;
- rendered/generated prose;
- parser output;
- LLM metadata;
- output hashes.

These are stored so later debugging can reconstruct what a player actually saw.

---

# 3. Core invariants

1. One Session has one ordered canonical event stream.
2. Canonical state changes only through committed canonical event batches.
3. At most one canonical state-changing progression/resolution operation advances a Session at a time.
4. Simultaneous actions resolve against one frozen canonical base state.
5. Network arrival order has no semantic gameplay meaning.
6. Canonical rules are pure and deterministic except through explicit named seeded randomness.
7. Canonical numeric mechanics use deterministic scalar representations; unconstrained floating-point is forbidden.
8. LLMs never directly modify canonical world/progression state.
9. Scenario definitions are immutable/version-pinned for active Sessions.
10. Player visibility is derived from explicit epistemic/permission state.
11. Gameplay availability and ending selection are canonical progression, not presentation behavior.
12. Historical replay never depends on current wall-clock time or current LLM behavior.
13. Current state can be rebuilt from pinned initial bundle + canonical events under the historical compatibility contract.
14. Post-commit work that matters to continuing the Session is durably scheduled.
15. Scenario creation should normally require data/configuration rather than engine code.

---

# 4. Logical bounded contexts

These are logical module boundaries, not microservices.

## Scenario Authoring & Compilation

Owns:

- source scenario configuration;
- schemas;
- rules;
- authoring validation;
- compilation into immutable CompiledScenarioBundle.

## Session & Participation

Owns:

- Session lifecycle;
- participants;
- invitations;
- character/role assignment;
- action slots;
- pending submissions;
- synchronization/session participation state.

## Command Processing

Owns:

- authoritative EngineCommand intake;
- authenticated/system principal binding;
- command idempotency;
- command routing;
- command journal/audit.

## Canonical Simulation

Owns:

- Action expansion;
- interaction analysis;
- resolver;
- canonical Domain Events;
- State Mutation IR application;
- state validators;
- Session stream revision.

## Scenario Progression

Canonical deterministic layer owning:

- trigger evaluation;
- progression state;
- DecisionPoint activation;
- ResolutionWindow opening/closing;
- canonical pacing gates;
- ending activation.

It is part of canonical mechanics.

## Epistemics & Visibility

Owns:

- Propositions;
- Facts where explicitly modeled;
- Observations;
- Knowledge;
- Beliefs;
- Suspicions;
- CommunicationClaims;
- Evidence relationships;
- player-safe views.

## Temporal & Scheduler

Owns:

- logical time;
- deterministic scheduled logical consequences;
- due-effect evaluation;
- timing policies.

## Narrative Direction

Presentation-only deterministic logic.

May decide:

- ordering/emphasis of already-authorized information;
- presentation grouping;
- recap emphasis;
- nonmechanical presentation sequencing.

May not open actions, change progression or decide endings.

## Narrative Realization

Converts a player-safe PresentationPlan into:

- prose;
- dialogue;
- media references;
- localized output.

May use an LLM but remains noncanonical.

## Identity & Entitlements

Guest/account/ownership/billing boundary outside simulation core.

## Analytics & Operations

Telemetry, performance and operational observability outside canonical state.

---

# 5. Single Canonical Frontier

A Session advances canonical gameplay through one serialized frontier.

## Rule

While a simultaneous ResolutionWindow is open:

- players may create/replace pending ActionSubmissions;
- gameplay canonical state remains fixed at the window's opening/base revision;
- unrelated world-changing operations do not independently commit through the window.

When something must happen:

- the window resolves;
- is interrupted/cancelled according to explicit policy;
- or an explicit canonical progression operation occurs between windows.

## Scheduler

Logical scheduled effects are evaluated at deterministic progression boundaries.

If a scenario requires an interrupting scheduled effect, the rule explicitly determines how the current window is closed/interrupted before that effect commits.

## Wall-clock deadlines

A real-world deadline never directly mutates world state.

It produces an idempotent EngineCommand such as:

`CloseWindowDueToDeadline`

which is serialized through the same Session frontier.

Expected-revision concurrency remains a safety mechanism, not the normal conflict-resolution mechanic.

---

# 6. EngineCommand boundary

All authoritative runtime inputs enter through commands.

Proposed envelope:

```
EngineCommand
  command_id
  command_type
  session_id
  principal
  optional expected_revision
  optional correlation_id
  optional causation_ref
  payload
  received_at   // operational metadata
```

## Principal

The authoritative principal is server-bound from authenticated/system context.

The client must not gain authority by submitting another participant id in payload.

Possible principals:

- Participant;
- SystemScheduler;
- DeadlineService;
- AdminRecovery under restricted policy.

## Command examples

- SubmitAction
- ReplaceAction
- CloseResolutionWindow
- DeadlineElapsed
- ResumeSession
- ApplyAutomaticProgression

Exact list remains extensible.

## Idempotency

`command_id` is unique within its relevant scope.

Reprocessing the same command must not duplicate canonical effects.

Command processing records outcome/status for reliable retry.

---

# 7. Pending ActionSubmission

ActionSubmission is authoritative pending input, not yet a world event.

Proposed shape:

```
ActionSubmission
  submission_id
  session_id
  resolution_window_id
  action_slot_id
  action_type_id
  parameters
  selected_target_refs[]
  source
  received_at
```

Participant identity is bound by server command context.

## Replacement

A window policy may allow multiple submissions for one slot.

Each replacement has its own immutable submission id.

At freeze time, the window records the selected final submission id for every slot.

Older submissions remain audit history but are not interpreted as canonical world actions.

---

# 8. ActionDefinition

Pinned inside CompiledScenarioBundle:

```
ActionDefinition
  action_type_id
  parameter_schema
  actor_constraints
  target/addressability_rules
  admission_predicates
  claim_derivation
  read_write_footprint
  timing_model
  resolver_policy_refs
  semantic_event_templates
  mutation_templates
  optional_tags
```

Client input never owns:

- claims;
- resource costs;
- preconditions;
- canonical effects;
- permission rules;
- semantic priority.

Scenario rules are compiled pure deterministic logic.

---

# 9. ExpandedAction

Derived server-side from:

`ActionSubmission + ActionDefinition + frozen SessionState`

```
ExpandedAction
  action_instance_key
  submission_id
  action_slot_id
  definition_ref
  actor_entity_id
  normalized_parameters
  resolved_targets[]
  derived_claims[]
  read_write_footprint
  logical_interval
  resolver_policy
  base_revision
```

`action_instance_key` must be deterministic from stored resolution/input identity or otherwise be stored before replay; replay must not depend on generating a fresh random UUID.

---

# 10. AdmissionResult and ActionOutcome

## AdmissionResult

Before world resolution:

- ADMITTED
- REJECTED_INVALID
- REJECTED_UNAUTHORIZED
- REJECTED_STALE
- REJECTED_NOT_ADDRESSABLE
- REJECTED_PRECONDITION

Rejected submissions remain audit/input history and normally do not generate world events.

## ActionOutcome

For every ADMITTED action exactly one outcome:

- SUCCEEDED
- PARTIAL
- FAILED
- INTERRUPTED
- TRANSFORMED
- NO_EFFECT

An unsuccessful admitted attempt can still create canonical semantic events when the attempt matters.

---

# 11. Claims

Claims describe contention-relevant access/demand.

```
Claim
  target_ref
  optional scope
  mode
  optional quantity
  optional unit
  optional logical_interval
```

Initial claim modes:

- READ
- WRITE
- CONSUME
- RESERVE
- PRODUCE
- EXCLUSIVE

Authority, capability, knowledge and addressability remain validation concerns.

## Interaction grouping

Actions sharing potentially interacting claim scopes are grouped regardless of whether any single pair would exceed capacity.

Resource/capacity resolution evaluates the aggregate group, avoiding pairwise-only errors.

---

# 12. Interaction Graph

Vertices are ADMITTED ExpandedActions.

Typed edges:

- CONTENTION
- EXCLUSION
- INTERFERENCE
- DEPENDENCY
- COMPLEMENTARITY
- ORDER_SENSITIVE

Independent actions have no edge.

Connected interaction components may be resolved independently.

The complete ResolutionPlan still undergoes global invariants before commit.

Property:

> Permuting independent components must not change final canonical state.

---

# 13. Bounded resolver strategies

Avoid a universal solver and arbitrary scenario code.

Core strategy families may include:

- COMPOSE;
- CAPACITY_ALLOCATE;
- SELECT_ONE / EXCLUSIVE_PRIORITY;
- ORDERED_RESOLVE;
- GROUP_TRANSFORM;
- MUTUAL_FAILURE where explicitly meaningful.

Exact strategy names/parameters remain schema work.

Exceptional combinations may use compiled declarative interaction transforms.

No scenario-provided arbitrary runtime JavaScript.

---

# 14. Rule purity

Every compiled gameplay rule is a pure deterministic function of declared inputs.

Allowed inputs include:

- pinned CompiledScenarioBundle;
- frozen SessionState;
- ActionSubmission/ExpandedAction;
- declared logical time;
- explicit named RNG API.

Forbidden implicit inputs:

- current wall clock;
- network;
- filesystem;
- mutable environment/global state;
- provider APIs;
- unseeded randomness.

This applies to:

- conditions;
- claim derivation;
- interaction detection;
- resolution;
- progression;
- ending logic.

---

# 15. Deterministic resolution pipeline

1. Receive idempotent command requesting window resolution/closure.
2. Freeze ResolutionWindow.
3. Freeze selected final ActionSubmissions per slot.
4. Confirm canonical Session revision equals window base revision.
5. Load pinned CompiledScenarioBundle.
6. Validate submissions/window/participant binding.
7. Validate addressability and preconditions.
8. Derive ExpandedActions deterministically.
9. Produce AdmissionResults.
10. Build Interaction Graph/groups.
11. Partition interaction components.
12. Resolve components under engine invariants + compiled policies.
13. Request named deterministic random draws where declared.
14. Produce one ActionOutcome per admitted action.
15. Produce complete ResolutionPlan.
16. Run global plan validators.
17. Materialize semantic CanonicalEventContent + State Mutation IR.
18. Apply batch to transient SessionState.
19. Evaluate deterministic Scenario Progression resulting from the new state.
20. Add progression events/mutations such as opening next DecisionPoint/Window or ending Session.
21. Run complete post-state validators.
22. Canonicalize/hash deterministic before-state, event batch and after-state.
23. Atomically:
    - append canonical event batch;
    - update synchronous critical SessionState/revision/hash;
    - persist ResolutionRecord;
    - enqueue required transactional outbox work.
24. Commit.
25. Post-commit workers build/send presentation, realtime notifications and secondary projections idempotently.

---

# 16. Scenario Progression

Scenario Progression is canonical.

It evaluates deterministic rules for:

- beat/scene progression state;
- trigger activation;
- DecisionPoint activation;
- ResolutionWindow creation;
- window policies;
- objective/goal progression;
- ending activation;
- canonical transition to completed/paused state.

If multiple progression candidates are valid, selection must be resolved by:

- explicit deterministic priority;
- deterministic composition where compatible;
- or named seeded randomness declared in the bundle.

Progression emits canonical events.

The Narrative Director may not override these results.

---

# 17. Narrative Direction

Narrative Direction receives a player-safe projection and canonical progression state.

It may determine:

- which already-known facts/events to foreground;
- presentation ordering;
- recap emphasis;
- nonmechanical dramatic emphasis.

It may not:

- grant knowledge;
- activate a DecisionPoint;
- add/remove legal actions;
- alter resources/world state;
- choose an ending;
- bypass Scenario Progression.

If presentation selection itself uses randomness, it must be seeded/versioned if exact replay of PresentationPlan is required; otherwise the generated/stored PresentationPlan is authoritative interaction evidence.

---

# 18. Narrative Realization and PresentationRecord

Narrative Realization turns a player-safe PresentationPlan into user-facing output.

LLM use is optional.

A deterministic/template fallback must exist for runtime continuity.

Every delivered presentation that can influence a later player choice should have an immutable PresentationRecord containing:

```
PresentationRecord
  presentation_id
  session_id
  audience
  related_progression/event_refs
  presentation_plan_hash
  realizer_version
  template_version
  optional model/provider metadata
  optional prompt_version
  exact_delivered_output
  output_hash
  delivery/generation status
```

PresentationRecord is not Canonical World State.

It is part of the immutable Session interaction/audit record.

Historical narrative replay displays the stored delivered output.

---

# 19. Event model

Each canonical event uses:

- finite engine-level `event_family`;
- precise core/scenario-specific `event_code`;
- typed versioned payload;
- optional State Mutation IR.

Semantic meaning and mechanical transition remain distinct but linked.

Example:

```
event_family: world.condition
event_code: island.radio_sabotaged
```

The event store must not degrade into generic "state changed" records.

---

# 20. Deterministic event content versus record metadata

## CanonicalEventContent

Hash/replay-relevant fields:

```
session_id
stream_revision
event_batch_key
batch_index

event_family
event_code
event_schema_version

logical_time

correlation_ref
causation_refs[]
actor_refs[]
subject_refs[]

payload
mutations[]
```

## EventRecordMetadata

Stored operationally but excluded from semantic batch hash/recomputation:

```
database_record_id
recorded_at
writer_instance/debug metadata
```

## Identity

Prefer deterministic event keys within Session such as:

`event_batch_key + batch_index`

rather than generating fresh semantic UUIDs during replay.

If surrogate UUIDs are used in storage, they do not define semantic equality.

---

# 21. State Mutation IR

The Mutation IR is a typed replay/apply representation, not the domain language shown to scenario authors.

Proposed categories:

## World
- entity.create
- entity.deactivate
- component.set
- number.adjust
- collection.add
- collection.remove

## Epistemic
- fact.assert
- fact.end_validity
- observation.record
- knowledge.add/end
- belief.add/end
- suspicion.add/end
- communication_claim.record

## Scheduler
- schedule.add
- schedule.cancel

## Progression
- decision_point.activate/deactivate
- window.open/close
- session.complete/pause
- progression_marker.set

Exact IR vocabulary remains subject to contract work.

Rules:

- target references are schema-resolved;
- protected engine state cannot be mutated by arbitrary scenario paths;
- all operations validate types/units;
- event batches apply atomically;
- ordinary canonical history is never hard-deleted.

---

# 22. Deterministic numeric model

Canonical mechanics do not use unconstrained floating-point.

Allowed defaults:

- integer;
- boolean/string/enumeration;
- fixed-point/scaled integer;
- explicit units;
- string-encoded large integer when beyond safe JSON number domain.

Examples:

- trust: integer on declared scale;
- probability: basis points or parts-per-million;
- money: integer minor units where applicable;
- energy: integer/scaled canonical unit.

No NaN or Infinity.

Unit conversion must be declared/deterministic.

Display rounding belongs to presentation, not mechanics.

---

# 23. Canonical logical time

Canonical logical time uses a deterministic integer representation such as scenario ticks or fixed smallest logical unit.

Authored calendar/time-of-day representation is a projection/format over logical time unless a scenario specifically defines calendar semantics.

Do not use platform-local timezone/DST behavior as canonical game mechanics.

Wall-clock timestamps remain operational trigger/audit data.

---

# 24. Epistemic model

## Proposition

Structured referencable statement with explicit predicate/arguments/value.

Example:

```
predicate: radio.sabotaged
arguments: [radio_01]
value: true
```

Falsehood is represented explicitly through proposition value/polarity, not via a generic DISBELIEVED state.

## Facts

Fact records are used when truth must be referenced by:

- epistemics;
- communication;
- evidence;
- narrative/progression rules.

Do not mirror every component field as a Fact.

Normal time-varying fact lifecycle:

- assert/open validity;
- end validity;
- supersede if modeled.

Historical truth is not silently retracted.

## Observations

Explicit records that a Character perceived an event/evidence/proposition.

## KnowledgeRecord

Represents deterministic granted knowledge of a Proposition with provenance.

Unknown is normally absence of KnowledgeRecord.

## BeliefRecord

Represents a Character belief about a Proposition/value.

May include confidence if the scenario needs it.

Belief is not automatically truth.

## SuspicionRecord

Represents weaker tentative attitude toward a Proposition.

## CommunicationClaim

Speaker asserts a Proposition/value to recipient/channel.

The claim never automatically changes canonical Fact.

## No automatic closure

The engine does not infer arbitrary logic such as:

"If A knows X and X implies Y, A knows Y"

unless a scenario rule explicitly grants that inference.

---

# 25. Addressability

Existence of an Entity ID is insufficient to target it.

ActionDefinition addressability rules decide whether an actor can reference/select/control a target.

Addressability is distinct from:

- existence;
- observation;
- knowledge;
- reachability;
- ownership;
- capability;
- permission.

This is both a gameplay and security invariant.

---

# 26. Temporal model

Three separate concepts remain.

## Engine order

Canonical total order from stream revision + batch index.

## Logical time

Deterministic in-world time.

## Wall clock

Real-world UX/deadline time.

Wall-clock values do not directly define historical state transitions.

A deadline produces a Command that the canonical frontier processes.

---

# 27. Scheduler

Canonical scheduled effects contain:

- schedule id/key;
- causation;
- due logical time/condition;
- event/effect template reference;
- cancellation policy;
- version info.

Scheduler evaluation occurs at deterministic progression boundaries.

Scheduled effects may cause explicit interruption only through declared scenario policy.

---

# 28. Randomness

No ambient random calls.

Each canonical random decision uses:

- Session seed;
- resolution/progression identity;
- rule identity;
- stable draw_key;
- pinned RNG algorithm/version;
- distribution/parameters;
- recorded result.

Adding an unrelated draw should not perturb existing named draws.

Exact PRNG remains a later contract decision.

---

# 29. Canonical serialization and hashes

Hashing requires declared canonical serialization.

RFC 8785/JCS is a useful candidate for JSON-compatible structures, subject to the canonical numeric restrictions above.

Pin:

- canonicalization version;
- hash algorithm.

Hashes include deterministic semantic structures only.

Record at least:

- CompiledScenarioBundle hash;
- before-state hash;
- canonical event-batch hash;
- after-state hash;
- PresentationPlan/output hashes where useful.

Operational fields such as database IDs and `recorded_at` are excluded from deterministic event-batch hash.

---

# 30. Event Sourcing scope

Event Source canonical Session history because causal reconstruction is a product requirement.

Do not Event Source by default:

- user profiles/accounts;
- entitlements;
- product analytics;
- page views;
- mutable source scenario drafts;
- media generation jobs;
- temporary UI state;
- ordinary technical logs.

Pending submissions/command journal are durable audit/input records but are not canonical world history until selected/resolved.

Snapshots are optimization only.

---

# 31. CQRS and projections

CQRS remains logical.

## Synchronous critical state

Updated in the same transaction as canonical append:

- current SessionState;
- stream revision;
- state hash;
- critical scheduler/progression state;
- required outbox entries.

## Secondary projections

Can lag/rebuild:

- recap;
- analytics;
- aggregate admin reports;
- search/reporting views.

Player view is derived deterministically from current state + epistemic/permission state.

No second database or broker is required.

---

# 32. Transactional Outbox

Any post-commit task required for reliable Session continuation is inserted into an outbox in the same transaction as canonical commit.

Potential outbox work:

- build/deliver PresentationRecord;
- realtime "state advanced" notification;
- secondary projection update;
- analytics emission;
- noncritical media generation trigger.

Consumers are idempotent.

Initial implementation may simply poll PostgreSQL.

No Kafka/Redis/broker is implied by this decision.

---

# 33. Commit protocol

Typical resolution commit:

```
window is frozen at revision N
resolve deterministically against N
produce canonical batch + after-state + progression
validate
canonicalize/hash

BEGIN
  verify Session current revision == N
  record ResolutionRecord
  append canonical ordered event batch
  update synchronous SessionState/revision/hash
  insert required outbox items
COMMIT
```

A revision mismatch is exceptional under Single Canonical Frontier and causes the uncommitted plan to be discarded/re-evaluated under explicit policy.

---

# 34. Versioning/build reproducibility

Each Session pins at least:

- engine_build_id / artifact digest;
- source commit for traceability;
- scenario id/version/bundle hash;
- state schema version;
- event schema version(s);
- Action/Command contract versions;
- Mutation IR version;
- resolver/progression rule bundle version;
- canonicalization/hash version;
- RNG algorithm/version + Session seed;
- Director/Realizer/template/prompt versions for interaction records.

The repository lockfile/build configuration must permit reconstruction where practical.

Published event records remain immutable.

Upcasters/migrations transform read interpretation, not historical stored bytes.

---

# 35. Replay modes

## State replay

Pinned initial Session state + canonical event mutations in stream order → historical SessionState/hash.

## Resolver verification replay

Historical frozen submissions + pinned build/bundle/rules + deterministic RNG derivation → recomputed AdmissionResults/InteractionGraph/ResolutionPlan/event semantic batch.

Compare with historical result/hash.

## Forensic replay

Use stored ResolutionRecord, random evidence, canonical events and PresentationRecords to investigate a historical Session even when obsolete executable runtime support is unavailable.

## Narrative replay

Display exact historical delivered PresentationRecord output.

Do not regenerate old prose with a current model and call it replay.

---

# 36. Scenario authoring/compilation

Authoring source may be YAML/JSON/DSL/UI-backed.

Runtime consumes only a compiled immutable bundle.

Compiler responsibilities include:

- schema validation;
- reference resolution;
- pure-rule compilation;
- claim/mutation validation;
- protected-state checks;
- progression ambiguity checks;
- ending reachability where statically decidable;
- deterministic priority validation;
- asset manifest/hash validation where relevant;
- version manifest;
- content hash.

No arbitrary runtime script escape hatch in MVP.

---

# 37. Validation architecture

## Command/Input

- idempotency;
- authentication/principal;
- Session/window status;
- payload schema.

## Pre-admission

- actor/role;
- addressability;
- capability;
- permissions;
- preconditions;
- target/resource validity.

## ResolutionPlan

- every admitted action has exactly one outcome;
- aggregate resource constraints;
- interaction completeness;
- allowed rules/strategies;
- named RNG evidence;
- allowed mutations.

## Progression

- deterministic candidate resolution;
- no illegal overlapping windows;
- ending consistency;
- scheduler ordering.

## Post-state

- component schemas;
- references;
- resources;
- numeric/unit invariants;
- temporal consistency;
- epistemic provenance;
- relationship/goals/commitments;
- progression/window consistency.

## Replay

- event/state hashes;
- resolver semantic result;
- version compatibility.

---

# 38. Testing architecture

Highest-priority properties:

1. same frozen inputs + build/bundle + seed → same semantic event batch;
2. permutation of simultaneous submission arrival order → same outcome;
3. duplicate command id → no duplicate effect;
4. duplicate submission id → no duplicate selected action;
5. independent interaction component order → same after-state;
6. canonical state remains fixed while simultaneous window collects submissions;
7. aggregate capacity constraints hold for all actions in component;
8. hidden/non-addressable entity ids cannot be targeted;
9. no epistemic record appears without allowed provenance;
10. event replay → exact historical state hash;
11. operational timestamps/DB ids do not affect semantic hash;
12. unrelated named random draw does not perturb existing draw result;
13. Realizer failure cannot corrupt canon;
14. outbox retry does not duplicate delivered canonical consequences;
15. progression cannot open contradictory active windows/endings;
16. canonical mechanics never require platform floating-point tolerance assertions.

Use:

- unit tests;
- state-transition tests;
- property-based tests;
- compiler validation;
- deterministic replay regression;
- synthetic PlayerControllers later;
- load tests only after functional architecture is proven.

---

# 39. Persistence baseline

Still proposed:

- PostgreSQL primary store;
- append-only canonical `session_events`;
- durable command/input tables;
- ResolutionRecords;
- synchronous current SessionState projection;
- transactional outbox;
- relational Session/scenario/version metadata;
- schema-validated JSON/JSONB where flexibility is justified.

A specialized event-store product is not required for MVP.

Concrete tables/indexes remain unfrozen.

---

# 40. Technical infrastructure baseline

Still under evaluation:

- React + TypeScript browser/PWA;
- PostgreSQL/Supabase baseline for MVP data/auth/API/realtime;
- realtime as notification only;
- Cloudflare hosting/storage where useful;
- R2 for media;
- PostHog with minimized/custom telemetry;
- monorepo with domain/resolver/compiler isolated from provider SDKs.

Do not add:

- Redis;
- Kafka/broker;
- graph database;
- vector database;
- Durable Objects;
- microservices;

without measured or demonstrated need.

---

# 41. Security/privacy invariants

- browser cannot append canonical events;
- browser cannot author canonical claims/effects;
- authenticated principal is server-bound;
- target IDs are subject to addressability, not merely existence;
- participants cannot fetch another participant's private view;
- private state/free text is not sent to analytics by default;
- logs redact sensitive/private game state where possible;
- guest play requires minimal PII;
- scenario bundles contain no runtime service secrets.

---

# 42. Decisions deliberately still unfrozen

- exact Action tags/families;
- Claim selector syntax;
- exact Mutation IR target encoding;
- exact fixed-point scales by metric;
- exact PRNG;
- final Scenario DSL syntax;
- free-text action confirmation policy;
- snapshot frequency;
- exact SQL schema/indexes;
- detailed Relationship/Goal/Commitment representation;
- Scenario Studio UI;
- NPC agent architecture;
- vector/graph databases;
- Redis/Durable Objects;
- microservices;
- payment provider;
- retention periods;
- recap video pipeline.

---

# 43. Phase-0 acceptance criteria

This baseline is ready for explicit acceptance when we agree that the following structural boundaries are correct:

1. Session canonical stream + Single Canonical Frontier.
2. EngineCommand/idempotency boundary.
3. Pending ActionSubmission versus canonical world history.
4. ActionDefinition/ExpandedAction server authority.
5. Claims + Interaction Graph/groups.
6. Deterministic bounded resolver.
7. Canonical Scenario Progression separate from Narrative Direction.
8. Semantic event + typed Mutation IR.
9. Deterministic numeric/logical-time representation.
10. Fact/Observation/Knowledge/Belief/Claim separation.
11. Named deterministic randomness.
12. Immutable CompiledScenarioBundle.
13. Logical CQRS + synchronous critical state + transactional outbox.
14. Versioned canonical serialization/hashing.
15. State/resolver/forensic/narrative replay boundaries.
16. LLM exclusion from canonical mechanics.

Accepted on 2026-10-02. Future structural changes require an explicit superseding ADR and a new baseline version.


---

# 44. Acceptance record

Architecture Baseline v0.3 was explicitly accepted on 2026-10-02.

Binding ADRs:

- ADR-001 Selective Event Sourcing Scope
- ADR-002 Session Stream and Single Canonical Frontier
- ADR-003 PostgreSQL Persistence Baseline
- ADR-004 Componentized Entity Model
- ADR-005 EngineCommand and ActionSubmission Authority Boundary
- ADR-006 Claims, Interaction Graph and Deterministic Resolver
- ADR-007 Semantic Domain Events and State Mutation IR
- ADR-008 Epistemic Model
- ADR-009 Temporal and Scheduler Model
- ADR-010 Compiled Scenario Bundle and Pure Rules
- ADR-011 Versioning, Canonical Serialization, Hashing and Replay
- ADR-012 Scenario Progression, Narrative Direction and Realization Separation
- ADR-013 Logical CQRS, Critical Projection and Transactional Outbox
- ADR-014 LLM Runtime Boundaries

ADR-015 Guest Identity / Participation remains PROPOSED and is not part of the accepted Phase-0 architecture.
