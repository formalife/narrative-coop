# DOMAIN MODEL v0.1

**Status:** PROPOSED  
**Date:** 2026-10-02  
**Governing architecture:** Architecture Baseline v0.3 — ACCEPTED  
**Governing contracts:** Domain Contracts v0.2 — ACCEPTED  
**Purpose:** define aggregate ownership, entities, value objects, state partitions and lifecycle relationships before persistence/schema design.

---

# 1. Modeling rule

This document answers:

> **Who owns each piece of state, and what is its authoritative lifecycle?**

It does NOT define:

- SQL tables;
- indexes;
- API transport shapes;
- package/class layout;
- final Scenario DSL syntax.

A logical model object does not imply a dedicated database table.

---

# 2. Top-level state ownership

The runtime is divided into six ownership zones.

## 2.1 Scenario Definition Zone — immutable design-time/runtime definition

Authoritative owner:

`CompiledScenarioBundle`

Contains immutable definitions/schemas/rules referenced by a Session.

Not Session state.

## 2.2 Canonical Session Zone — event-sourced gameplay/progression truth

Authoritative historical owner:

`SessionEventStream`

Current projection:

`SessionAggregate / SessionStateProjection`

Contains only canonical gameplay/session progression state.

## 2.3 Pending Input Zone — authoritative but noncanonical pending input

Owns:

- WindowInputGate;
- ActionSubmissions;
- FrozenInputSet;
- CommandProcessingRecord;
- ResolutionAttempt.

These influence future canonical transitions but are not world truth.

## 2.4 Transition Evidence Zone — immutable causal/debug evidence

Owns:

- CanonicalTransitionRecord;
- ResolutionRecord;
- NamedRandomDraw evidence;
- optional expanded debug artifacts.

This is immutable audit/replay evidence but not mutable canonical state.

## 2.5 Presentation Interaction Zone — player-facing causal evidence

Owns:

- materialized PlayerInteractionView where retained;
- PresentationPlan;
- PresentationRecord.

Presentation artifacts do not mutate canonical state.

## 2.6 Operations Zone

Owns:

- TransactionalOutboxItem;
- worker attempt metadata;
- product analytics;
- technical logs.

No operations record becomes canonical world truth by itself.

---

# 3. Scenario Definition Aggregate

## Aggregate root: CompiledScenarioBundle

```
CompiledScenarioBundle
  scenario_id
  scenario_version
  bundle_hash
  bundle_schema_version

  component_definitions
  entity_archetypes
  role_definitions

  predicate_definitions
  relationship_definitions

  action_definitions
  interaction_policy_definitions
  resolver_strategy_configs

  decision_point_definitions
  resolution_window_templates
  progression_rule_definitions

  scheduler_rule_definitions

  goal_definitions
  commitment_definitions
  narrative_thread_definitions
  metric_definitions

  semantic_event_definitions
  mutation_schema_definitions

  ending_definitions

  presentation_policy_definitions
  asset_manifest

  mechanics_compatibility_manifest
```

## Bundle invariants

1. Immutable after publish.
2. Content-addressed by bundle_hash.
3. All internal references resolve.
4. All gameplay rules are compiled/pure.
5. No service secrets.
6. Defines schemas/policies, not mutable Session instances.
7. A Session pins exactly one bundle version/hash.

---

# 4. Definition objects

## 4.1 ComponentDefinition

```
ComponentDefinition
  component_type
  schema_version
  data_schema
  invariant_refs
  claimable_scopes
  mutation_permissions
```

Defines shape/invariants of one component type.

## 4.2 EntityArchetypeDefinition

```
EntityArchetypeDefinition
  archetype_ref
  required_component_types
  optional_component_types
  tags
```

No runtime class inheritance semantics are implied.

## 4.3 RoleDefinition

```
RoleDefinition
  role_ref
  character_constraints
  capability_refs
  action_access_rules
  initial_epistemic_rules
  optional objective_refs
```

RoleDefinition is scenario mechanics, not account identity.

## 4.4 RelationshipDefinition

```
RelationshipDefinition
  relationship_type_ref
  directionality
  endpoint_schema
  component_schema_refs
  invariant_refs
```

Directionality:

- DIRECTED
- UNDIRECTED

The model does not assume relationships are always scalar metrics.

## 4.5 GoalDefinition

```
GoalDefinition
  goal_definition_ref
  owner_schema
  activation_rule_ref
  success_rule_ref
  failure_rule_ref
  optional progress_schema
  optional presentation_ref
```

## 4.6 CommitmentDefinition

```
CommitmentDefinition
  commitment_definition_ref
  party_schema
  terms_schema
  activation_rule_ref
  fulfillment_rule_ref
  breach_rule_ref
  optional expiry_rule_ref
```

## 4.7 NarrativeThreadDefinition

```
NarrativeThreadDefinition
  thread_definition_ref
  activation_rule_ref
  escalation_rules
  resolution_rule_ref
  abandonment_rule_ref
  optional priority_model
```

NarrativeThread is canonical only if its state affects progression/presentation rules.

---

# 5. Session Aggregate

## Aggregate root: SessionAggregate

The Session Aggregate is the logical canonical consistency boundary.

```
SessionAggregate
  session_identity
  mechanics_manifest

  participation_state

  canonical_state_content
```

The aggregate's historical authority is the SessionEventStream.

The persisted/current SessionStateProjection is a rebuildable materialization of this aggregate.

---

# 6. SessionIdentity

```
SessionIdentity
  session_id
  created_from_scenario_id
  created_from_scenario_version
  scenario_bundle_hash

  session_seed   // REQUIRED for deterministic RNG; see Open Contract Conflict DM-001
```

## Important

Domain Contracts v0.2's MechanicsVersionManifest currently omits `session_seed`, while accepted ADR-011 explicitly requires RNG algorithm/version **and Session seed**.

This model therefore records a blocking contract mismatch:

> **DM-001 — session_seed must be added to the accepted replay contract before Domain Model acceptance.**

This is not silently accepted by this document.

---

# 7. ParticipationState

Canonical Session participation binding needed for deterministic authorization/replay.

```
ParticipationState
  participant_slots:
    CanonicalMap<ParticipantSlotId, ParticipantSlotBinding>
```

## ParticipantSlotBinding

```
ParticipantSlotBinding
  participant_slot_id

  participant_ref
  character_entity_id
  role_ref

  status:
    RESERVED
    JOINED
    ACTIVE
    DISCONNECTED
    LEFT

  optional joined_at_revision
  optional left_at_revision
```

## Ownership boundary

The Session stores only opaque participant references and gameplay bindings.

Account profile/email/billing identity remains outside the canonical Session.

## Invariants

1. ActionSlot participant/actor binding resolves through ParticipationState.
2. External account changes cannot silently alter historical Session role/character authority.
3. Character/role reassignment, if ever supported after Session start, is canonical and evented.
4. Invite-token mechanics are outside canonical world state until they produce a participant binding.

---

# 8. CanonicalStateContent

```
CanonicalStateContent
  lifecycle_state
  logical_time_state

  entity_registry
  relationship_graph

  epistemic_state

  goal_ledger
  commitment_ledger

  scheduler_state
  progression_state

  scenario_metric_state
```

All collections have explicit canonical list/set/map semantics per accepted contracts.

---

# 9. EntityRegistry

```
EntityRegistry
  entities:
    CanonicalMap<EntityId, Entity>
```

## Entity

```
Entity
  entity_id
  archetype_ref

  lifecycle:
    ACTIVE
    INACTIVE

  tags: CanonicalSet<TagRef>

  components:
    CanonicalMap<ComponentType, ComponentInstance>
```

## ComponentInstance

```
ComponentInstance
  component_type
  component_schema_version
  data
```

## Invariants

1. EntityId unique within Session.
2. Required archetype components always present while valid.
3. Component data validates against pinned ComponentDefinition.
4. Entity deactivation does not erase historical references.
5. Components contain state, not arbitrary executable behavior.
6. A component field used by Claims/Mutations must be addressable through compiled definitions, not free paths.

---

# 10. RelationshipGraph

Relationships are canonical first-class links, separate from world Entities but componentized similarly.

```
RelationshipGraph
  relationships:
    CanonicalMap<RelationshipId, Relationship>
```

## Relationship

```
Relationship
  relationship_id
  definition_ref

  endpoints:
    OrderedList<EntityId>

  lifecycle:
    ACTIVE
    INACTIVE

  components:
    CanonicalMap<ComponentType, RelationshipComponentInstance>
```

## Relationship identity

For DIRECTED definitions:

`(definition_ref, ordered endpoints)`

is semantically meaningful.

For UNDIRECTED definitions:

endpoints are canonicalized according to the definition before identity/hash construction.

## Why not model every relationship as an Entity?

Because relationship identity/endpoint invariants differ materially from ordinary world objects, while still benefiting from typed components.

## Invariants

1. Endpoints reference valid canonical Entities.
2. Directionality comes from RelationshipDefinition.
3. No duplicate active relationship for a definition/endpoints combination unless the definition explicitly allows multiplicity.
4. Relationship components are schema-validated exact numeric/scalar structures.

---

# 11. EpistemicState aggregate component

```
EpistemicState
  proposition_registry

  facts

  observations

  knowledge
  beliefs
  suspicions

  communication_claims
  evidence_relations
```

This is part of the Session aggregate, not a separate transactional aggregate.

---

# 12. PropositionRegistry

```
PropositionRegistry
  propositions:
    CanonicalMap<PropositionKey, PropositionValue>
```

## Rule

A PropositionValue becomes part of canonical Session state when first referenced by a canonical epistemic/fact/evidence record that requires durable resolution.

Events that introduce a previously unseen proposition include enough content to reconstruct its PropositionValue deterministically.

## Invariants

1. PropositionKey == deterministic content hash/key of PropositionValue.
2. Same key cannot map to different content.
3. PredicateDefinition comes from pinned Scenario Bundle.
4. Runtime-created Entity references are allowed if they are valid at the relevant historical point.

---

# 13. FactLedger

```
FactLedger
  facts:
    CanonicalMap<FactId, FactRecord>

  active_fact_index:
    CanonicalMap<PropositionKey, CanonicalSet<FactId>>
```

`active_fact_index` is a projection/index inside CanonicalStateContent only if needed by deterministic rules; it must be derivable from FactRecords.

## Invariants

1. FactRecord history is immutable except explicit end-validity fields applied canonically.
2. Contradictory active Facts are allowed only when the PredicateDefinition semantics permit multi-valued truth; otherwise post-state validation rejects them.
3. Ordinary Entity component state is not duplicated into FactLedger unless epistemic reference requires a FactRecord.

---

# 14. ObservationLedger

```
ObservationLedger
  observations:
    CanonicalMap<ObservationId, ObservationRecord>
```

Observations are historical provenance facts about perception.

They are append-only at domain level.

A later loss of access/attention does not delete a historical ObservationRecord.

---

# 15. KnowledgeLedger

```
KnowledgeLedger
  records:
    CanonicalMap<KnowledgeId, KnowledgeRecord>

  active_by_subject_proposition:
    CanonicalMap<SubjectPropositionKey, KnowledgeId>
```

## Invariant

At most one active KnowledgeRecord per:

`(knower_entity_id, proposition_key)`

If new provenance arrives for already-known knowledge, implementation may:

- add provenance to a canonical replacement record;
- or emit a no-op epistemic change plus provenance evidence;

but must not create ambiguous multiple active knowledge states.

Exact provenance-normalization strategy remains open for data-model review.

---

# 16. BeliefLedger

```
BeliefLedger
  records
  active_by_subject_predicate_scope
```

Unlike Knowledge, Belief requires conflict/supersession semantics.

## Proposed invariant

For a predicate/argument/temporal scope that is defined as single-valued, a subject may have at most one active BeliefRecord value unless the scenario explicitly models ambivalence/multiple hypotheses.

This is PROPOSED and requires red-team.

---

# 17. SuspicionLedger

Structured similarly to BeliefLedger.

Suspicion is only used when the Scenario Bundle gives it mechanical/narrative meaning.

Do not automatically create Suspicion for every uncertain Belief.

---

# 18. CommunicationClaimLedger

```
CommunicationClaimLedger
  claims:
    CanonicalMap<CommunicationClaimId, CommunicationClaim>
```

A CommunicationClaim records assertion occurrence.

It does not encode actual reception.

Reception is ObservationRecord.

---

# 19. EvidenceGraph

```
EvidenceGraph
  relations:
    CanonicalMap<EvidenceRelationId, EvidenceRelation>
```

Evidence objects themselves are usually world Entities.

EvidenceRelation connects an evidence Entity/reference to a PropositionValue.

No automatic inference occurs.

---

# 20. GoalLedger

## GoalInstance

```
GoalInstance
  goal_instance_id
  definition_ref

  owner_refs: CanonicalSet<Ref>

  status:
    DORMANT
    ACTIVE
    SUCCEEDED
    FAILED
    CANCELLED

  activated_by_event_ref
  optional resolved_by_event_ref

  optional progress_state
```

```
GoalLedger
  goals:
    CanonicalMap<GoalInstanceId, GoalInstance>
```

## Invariants

1. GoalDefinition owns rule semantics.
2. GoalInstance stores only canonical runtime status/progress required by future rules.
3. Derived progress should not be duplicated if cheaply reconstructible from other canonical state.
4. Status transition is canonical/evented.

---

# 21. CommitmentLedger

Commitments represent canonical obligations/promises/agreements only when scenario mechanics use them.

## CommitmentInstance

```
CommitmentInstance
  commitment_instance_id
  definition_ref

  obligor_refs: CanonicalSet<Ref>
  beneficiary_refs: CanonicalSet<Ref>

  terms
  optional due_logical_tick

  status:
    PROPOSED
    ACTIVE
    FULFILLED
    BREACHED
    CANCELLED
    EXPIRED

  created_by_event_ref
  optional resolved_by_event_ref
```

```
CommitmentLedger
  commitments:
    CanonicalMap<CommitmentInstanceId, CommitmentInstance>
```

## Red-team note

`PROPOSED` as a canonical commitment status may conflate negotiation intent with an actual world obligation. This is flagged for postmortem review.

---

# 22. SchedulerState

```
SchedulerState
  scheduled_effects:
    CanonicalMap<ScheduleId, ScheduledEffect>

  due_index
```

`due_index` is derivable and may be projection-only rather than part of hashable content in final model.

## Invariants

1. PENDING -> FIRED or CANCELLED once.
2. No fired schedule re-fires.
3. Due conditions evaluate deterministically.
4. BoundaryOrderingPolicy governs due effects at an action frontier.

---

# 23. ProgressionState

```
ProgressionState
  progression_markers

  decision_point_instances

  narrative_thread_instances

  active_resolution_window

  active_ending_state
```

## Invariant

There is at most one action-collecting active ResolutionWindow.

Automatic progression may occur without creating a fake window.

---

# 24. NarrativeThreadInstance

```
NarrativeThreadInstance
  thread_instance_id
  definition_ref

  status:
    DORMANT
    ACTIVE
    ESCALATING
    RESOLVED
    ABANDONED

  optional level_or_stage

  activated_by_event_ref
  optional resolved_by_event_ref
```

NarrativeThread state is canonical only if rules/presentation selection depend on it.

Pure editorial labels remain bundle metadata instead.

---

# 25. EndingState

```
EndingState
  status:
    NOT_REACHED
    REACHED

  optional ending_ref
  optional reached_by_transition_key
```

## Invariants

1. Only Scenario Progression activates an ending.
2. Once REACHED, the Session cannot return to NOT_REACHED.
3. Ending activation and Session completion semantics must be explicitly related by ending/progression rules.

---

# 26. ScenarioMetricState

```
ScenarioMetricState
  metrics:
    CanonicalMap<MetricRef, CanonicalNumberOrScalar>
```

## Rules

- MetricDefinition comes from bundle.
- A metric belongs here only if future canonical rules/presentation/progression consume it.
- Do not create generic psychology metrics that have no mechanical role.
- Units/scales are scenario-defined and deterministic.

---

# 27. Session lifecycle

```
SessionLifecycleState
  CREATED
  WAITING_FOR_PARTICIPANTS
  READY
  ACTIVE
  PAUSED
  COMPLETED
  ABANDONED
```

## Proposed ownership

Lifecycle is canonical Session state.

Participant invite/transport presence may be operational, but transition to READY/ACTIVE/PAUSED/etc. is canonical when it changes legal gameplay transitions.

---

# 28. SessionEventStream

Historical canonical owner.

```
SessionEventStream
  session_id
  OrderedList<CanonicalEventContent>
```

Conceptually append-only.

## Invariants

1. Revision convention from accepted contracts.
2. Events immutable after commit.
3. Event content applies typed Mutation IR to rebuild Session canonical state.
4. Event batch boundaries are owned by CanonicalTransitionRecord.
5. Current SessionStateProjection must reproduce the event stream's resulting hash.

---

# 29. CanonicalTransitionHistory

Immutable transition evidence.

```
CanonicalTransitionHistory
  transitions:
    OrderedList<CanonicalTransitionRecord>
```

Order can be derived from base/resulting revisions.

A transition is not another source of world-state truth; its event batch is.

The record owns:

- batch causation;
- before/after hash evidence;
- transition kind;
- optional ResolutionRecord relation.

---

# 30. PendingInputAggregate

Logical noncanonical aggregate keyed by active Window.

```
PendingInputAggregate
  window_input_gate
  submissions
  frozen_input_set

  command_processing_records
  resolution_attempts
```

## Concurrency boundary

Pending input needs transactional compare-and-set independent from canonical StreamRevision.

It must still enforce:

- Window is the currently active canonical window;
- gate references the same base revision;
- secret submission access policy;
- freeze is atomic.

## Lifecycle

Created when WindowOpen canonical event commits.

Destroyed/archived after Window resolution/cancel/interrupt is durable.

Historical selected inputs remain available through ResolutionRecord/audit retention.

---

# 31. CommandProcessingStore

Although grouped logically with pending/app state, commands may also exist outside a window.

```
CommandProcessingStore
  CanonicalMap<(SessionId, CommandKey), CommandProcessingRecord>
```

## Invariants

1. Key is scoped at least to Session + CommandKey.
2. Same semantic key/payload retry returns same logical result.
3. Processing record can outlive a worker process.
4. It is not replayed as canonical gameplay state.

---

# 32. ResolutionEvidenceStore

```
ResolutionEvidenceStore
  resolution_records
  optional debug_artifacts
```

NamedRandomDraw evidence required for replay is retained according to accepted replay policy.

If draw results are represented inside ResolutionRecord/event payload, avoid a competing separate truth source.

---

# 33. PlayerInteraction domain

## PlayerInteractionView

Derived from:

- canonical SessionStateProjection;
- ParticipationState;
- authorized pending-input state;
- pinned bundle definitions.

It may be:

- computed on demand;
- cached;
- optionally persisted by hash for debugging.

Its content hash `PlayerViewKey` defines identity.

It is not canonical world state.

## Important boundary

Mechanical `InteractionSurface` is authoritative-safe derived data.

Narrative Direction cannot modify it.

---

# 34. PresentationAggregate

```
PresentationAggregate
  presentation_plan
  presentation_record
```

A PresentationRecord is immutable delivered interaction evidence.

No Presentation object becomes input to canonical rules directly.

A later player ActionSubmission may reference the PresentationRecord/PlayerView that causally preceded it for audit/playtest analysis.

---

# 35. TransactionalOutbox aggregate

Operational reliability data.

```
OutboxStore
  items:
    CanonicalMap<OutboxId, TransactionalOutboxItem>
```

"CanonicalMap" here means deterministic logical identity/uniqueness, not inclusion in canonical Session hash.

## Invariants

- canonical transition-required outbox item is inserted atomically with event commit;
- task is at-least-once;
- side-effect consumer is idempotent;
- presentation task pins PlayerViewKey + PresentationGenerationContext.

---

# 36. Definition-to-instance relationships

```
CompiledScenarioBundle
  ComponentDefinition ──────> ComponentInstance
  EntityArchetypeDefinition -> Entity
  RoleDefinition ───────────> ParticipantSlotBinding
  RelationshipDefinition ───> Relationship
  PredicateDefinition ──────> PropositionValue
  ActionDefinition ─────────> ExpandedAction
  GoalDefinition ───────────> GoalInstance
  CommitmentDefinition ─────> CommitmentInstance
  NarrativeThreadDefinition -> NarrativeThreadInstance
  DecisionPointDefinition ──> DecisionPointInstance
  ResolutionWindowTemplate ─> ResolutionWindow
  ProgressionRuleDefinition -> ProgressionTransition/Evaluation
  MetricDefinition ─────────> ScenarioMetricState entry
  EndingDefinition ─────────> EndingState
```

Definitions never become mutable runtime state.

---

# 37. Canonical aggregate mutation boundary

Only a validated `TransitionPlan` may produce a canonical Session state change.

Logical pipeline:

```
authoritative command / system trigger
  -> prepare authoritative inputs
  -> build TransitionPlan
  -> validate complete transient after-state
  -> append canonical events
  -> reduce Mutation IR
  -> obtain next SessionAggregate projection
```

Direct aggregate mutation outside event application is forbidden in committed persistence behavior.

Pure in-memory transient planning is allowed before commit.

---

# 38. Aggregate invariants

## Session

1. One Session -> one pinned Scenario Bundle.
2. One Session -> one MechanicsVersionManifest.
3. One Session -> one canonical StreamRevision frontier.
4. At most one active action-collecting ResolutionWindow.
5. Canonical state revision equals last event stream revision.
6. Completed/Abandoned sessions cannot accept normal gameplay commands.

## Entity

7. Required components satisfy archetype definition.
8. Component schema/version matches pinned bundle.
9. Inactive entity remains referenceable historically.

## Relationship

10. Endpoints valid.
11. Directionality/multiplicity obey RelationshipDefinition.
12. Relationship components validate.

## Epistemic

13. PropositionKey content-stable.
14. Knowledge provenance explicit.
15. Communication assertion != reception != belief != fact.
16. No unsupported duplicate active epistemic state.

## Goals/commitments/threads

17. Runtime instance status follows definition-allowed state transitions.
18. Resolved terminal state cannot silently reopen unless definition explicitly allows a new instance.

## Scheduler/progression

19. Schedule fires/cancels once.
20. Progression evaluation converges or aborts before commit.
21. Ending only activated by Scenario Progression.

---

# 39. Cross-zone reference rules

Allowed directions:

```
Scenario Bundle definitions
       ↓
Canonical Session State
       ↓
PlayerInteractionView
       ↓
Presentation artifacts
       ↓
ActionSubmission provenance only

Pending Input
       -> references current canonical Window/base revision

Transition Evidence
       -> references canonical events/inputs/versions

Outbox
       -> references committed transition/state/player view context
```

Forbidden authority inversions:

- PresentationRecord cannot mutate Session state.
- Analytics cannot mutate Session state.
- CommandProcessingRecord cannot be interpreted as a world event.
- Pending ActionSubmission cannot grant knowledge or reserve world resources before canonical resolution unless the scenario explicitly models a canonical reservation action.
- External Account state cannot retroactively alter Session role bindings.

---

# 40. Deletion and lifecycle policy

## Canonical entities/relationships

Prefer deactivate/end validity rather than hard delete.

## Epistemic history

Historical observation/communication/fact provenance is not hard-deleted as part of gameplay.

## Pending input

Nonselected/replaced submissions may be retained under audit/privacy policy, but are not required for state replay after frozen selected evidence is retained.

## Presentation

Delivered PresentationRecord is immutable interaction evidence subject to future retention/privacy policy.

## Operational logs/outbox

May be compacted according to reliability/retention policy after their guarantees are satisfied.

---

# 41. Reconstruction hierarchy

To reconstruct canonical gameplay:

```
CompiledScenarioBundle initial state
+ Session identity/participation initialization
+ ordered SessionEventStream
= SessionAggregate at revision N
```

Do NOT require:

- command journal;
- nonselected pending submissions;
- ResolutionAttempt;
- PresentationRecord;
- analytics;
- outbox.

To perform resolver verification/forensic investigation, additional evidence may be required per ADR-011/accepted contracts.

---

# 42. Domain-model decisions intentionally still open

- DM-001 contract correction for Session seed.
- exact Relationship component vocabulary.
- exact Belief/Suspicion active-conflict semantics.
- whether Commitment status PROPOSED belongs in canonical commitment state.
- exact Goal progress representation.
- NarrativeThread level/stage representation.
- whether derivable indexes are inside hashable CanonicalStateContent or projection-only.
- opaque ID construction.
- exact mutation target/scoping representation.
- exact persistence split between normalized columns and JSONB.
- archival/retention implementation.

---

# 43. Domain model red-team targets

Before acceptance, test:

1. Session reconstruction without operational stores.
2. Account/participant identity change mid-session.
3. Character deactivation while referenced by old knowledge/evidence.
4. Directed vs undirected relationship identity.
5. Duplicate active KnowledgeRecord.
6. Contradictory belief hypotheses.
7. Proposed commitment never accepted.
8. Goal derived progress drifting from duplicated stored progress.
9. Recurring DecisionPointDefinition creating multiple instances.
10. Schedule firing at same frontier as ending.
11. Ending reached while active Window exists.
12. Nonselected ActionSubmission deletion and forensic replay.
13. Runtime-created proposition referencing deactivated entity.
14. PlayerInteractionView change with same StreamRevision due to pending input.
15. Rebuild state hash without Transition/Resolution/Presentation stores.
16. Session RNG replay requiring session_seed.

---

# 44. Next step

Perform Domain Model v0.1 red-team/postmortem.

If the model passes after corrections:
- accept Domain Model baseline;
- only then design persistence model / PostgreSQL schema.

If Domain Model reveals a conflict with accepted Domain Contracts v0.2, create an explicit contract revision before accepting the Domain Model.
