# DOMAIN MODEL v0.2

**Status:** PROPOSED  
**Date:** 2026-10-02  
**Governing architecture:** Architecture Baseline v0.3 — ACCEPTED  
**Accepted contracts:** Domain Contracts v0.2 — ACCEPTED  
**Required contract amendment:** Domain Contracts v0.3 — PROPOSED  
**Supersedes as working proposal:** DOMAIN_MODEL_v0.1

## 1. Scope

This model defines ownership and lifecycle of runtime state before persistence/schema design.

It intentionally does not define SQL tables, indexes, APIs, package/class layout or final authoring syntax.

## 2. Ownership zones

### Scenario Definition Zone

Root:
- CompiledScenarioBundle

Owns immutable definitions, schemas, policies and compiled pure rules.

### Session Bootstrap Zone

Root:
- SessionGenesis

Owns immutable bootstrap data required to reconstruct revision 0, including session_seed.

This zone becomes binding only if Domain Contracts v0.3 is accepted.

### Canonical Session Zone

Historical truth:
- SessionEventStream

Current projection:
- SessionStateProjection

Logical consistency boundary:
- SessionAggregate

### Pending Input Zone

Owns:
- WindowInputGate;
- ActionSubmission;
- FrozenInputSet;
- CommandProcessingRecord;
- ResolutionAttempt.

Authoritative pending/application state, but not canonical world truth.

### Transition Evidence Zone

Owns:
- CanonicalTransitionRecord;
- ResolutionRecord;
- NamedRandomDraw evidence;
- optional debug artifacts.

Immutable causal/replay evidence.

### Presentation Interaction Zone

Owns:
- PlayerInteractionView where retained/cached;
- PresentationPlan;
- PresentationRecord.

Never canonical world authority.

### Operations Zone

Owns:
- TransactionalOutboxItem;
- worker attempts;
- presence/connectivity;
- analytics;
- technical logs.

---

## 3. CompiledScenarioBundle

```
CompiledScenarioBundle
  scenario_id
  scenario_version
  bundle_hash
  bundle_schema_version

  component_definitions
  entity_archetypes
  role_definitions

  relationship_definitions

  predicate_definitions
  information_classification_definitions

  action_definitions
  interaction_policies
  resolver_strategy_configs

  decision_point_definitions
  resolution_window_templates
  progression_rules

  scheduler_rules

  goal_definitions
  commitment_definitions
  narrative_thread_definitions
  metric_definitions

  event_definitions
  mutation_schema_definitions

  ending_definitions

  presentation_policy_definitions
  asset_manifest
```

Bundle invariants:

1. immutable after publish;
2. content-addressed;
3. all references resolve;
4. runtime rules pure/deterministic;
5. no service secrets;
6. definitions never become mutable runtime objects.

---

## 4. SessionGenesis

Conditional on Domain Contracts v0.3 acceptance.

```
SessionGenesis
  session_id

  mechanics_version_manifest_ref
  mechanics_version_manifest_hash

  scenario_bundle_hash

  session_seed

  initial_canonical_state_hash

  created_at // operational
```

Revision 0 canonical state is derived from:

```
CompiledScenarioBundle + SessionGenesis
```

Then Domain Events create revisions 1+.

Invariants:

1. exactly one SessionGenesis per Session;
2. immutable;
3. session_seed durably retained;
4. bundle hash matches MechanicsVersionManifest;
5. created_at excluded from canonical hashes;
6. revision-0 state hash is reproducible.

---

## 5. SessionAggregate

```
SessionAggregate
  session_id
  mechanics_manifest_ref

  participation_state
  canonical_state_content
```

The aggregate is not stored as a monolithic authoritative document by definition.

Historical authority remains SessionEventStream.

---

## 6. ParticipationState

Canonical gameplay assignment, not realtime connectivity.

```
ParticipationState
  participant_slots:
    CanonicalMap<ParticipantSlotId, ParticipantBinding>
```

```
ParticipantBinding
  participant_slot_id
  participant_ref
  character_entity_id
  role_ref

  binding_status:
    UNBOUND
    BOUND
    RELEASED

  optional bound_at_revision
  optional released_at_revision
```

Invariants:

1. role/character authority resolves through this state;
2. external account-profile changes cannot alter historical Session authority;
3. assignment/reassignment after revision 0 is canonical/evented;
4. invites and socket presence are not canonical participation state.

---

## 7. PresenceState — operational

```
PresenceState
  participant_ref
  connection_status
  last_seen_at
  optional connection_ids
```

PresenceState is not part of CanonicalStateContent.

If connectivity triggers gameplay consequences, a system/deadline command converts that fact into an explicit canonical transition according to policy.

---

## 8. CanonicalStateContent

```
CanonicalStateContent
  lifecycle_state
  logical_time_state

  participation_state

  entity_registry
  relationship_graph

  epistemic_state
  information_classification_state

  goal_ledger
  commitment_ledger

  scheduler_state
  progression_state

  scenario_metric_state
```

No derived lookup indexes belong here unless the indexed value is itself semantic canonical state.

---

## 9. SessionDerivedIndexes

Rebuildable acceleration structures excluded from canonical state hash.

Examples:

```
SessionDerivedIndexes
  entities_by_archetype
  relationships_by_endpoint
  active_facts_by_proposition
  active_knowledge_by_subject_proposition
  active_beliefs_by_scope
  due_schedule_index
  active_goal_index
```

Rules:

- always derivable from CanonicalStateContent;
- safe to drop/rebuild;
- never resolve conflicts with canonical records;
- validators may compare them against canonical source records.

---

## 10. EntityRegistry

```
EntityRegistry
  entities:
    CanonicalMap<EntityId, Entity>
```

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

```
ComponentInstance
  component_type
  schema_version
  data
```

Invariants:

1. required archetype components valid;
2. component schemas pinned by bundle;
3. inactive entities remain historically referenceable;
4. state only, no arbitrary executable behavior;
5. claim/mutation scopes must be compiler-known.

---

## 11. RelationshipGraph

```
RelationshipGraph
  relationships:
    CanonicalMap<RelationshipId, Relationship>
```

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
    CanonicalMap<ComponentType, RelationshipComponent>
```

Rules:

- directed relationship endpoint order is semantic;
- undirected endpoint identity is canonicalized;
- multiplicity follows RelationshipDefinition;
- components are typed/versioned.

Relationship is not forced into ordinary Entity identity because endpoint/directionality invariants are distinct.

---

## 12. EpistemicState

```
EpistemicState
  proposition_registry
  fact_ledger
  observation_ledger
  knowledge_ledger
  belief_ledger
  suspicion_ledger
  communication_claim_ledger
  evidence_graph
```

No generic single "stance" object is introduced.

---

## 13. PropositionRegistry

```
PropositionRegistry
  propositions:
    CanonicalMap<PropositionKey, PropositionValue>
```

Rules:

1. PropositionKey is content-derived;
2. same key cannot map to different content;
3. PredicateDefinition comes from bundle;
4. runtime-created/deactivated Entity refs remain valid historical references;
5. introducing event contains enough PropositionValue data to reconstruct the registry.

---

## 14. FactLedger

```
FactLedger
  facts:
    CanonicalMap<FactId, FactRecord>
```

No active_fact_index inside hash-authoritative state.

Invariants:

1. validity may end; history is not deleted;
2. contradictory simultaneously valid Facts are rejected unless predicate semantics explicitly permit multi-valued truth;
3. component state is not duplicated as Fact unless epistemic reference requires it.

---

## 15. ObservationLedger

```
ObservationLedger
  observations:
    CanonicalMap<ObservationId, ObservationRecord>
```

Append-only at domain semantics.

Observation is perception provenance, not automatic Knowledge.

---

## 16. KnowledgeLedger

```
KnowledgeLedger
  records:
    CanonicalMap<KnowledgeId, KnowledgeRecord>
```

Semantic current-state invariant:

> At most one active KnowledgeRecord per (knower_entity_id, proposition_key).

Additional evidence/provenance does not create multiple conflicting "known" states.

If provenance changes are mechanically important, use an explicit superseding record/provenance relation policy; otherwise the additional observation/evidence remains in its own ledger.

---

## 17. Predicate epistemic cardinality

PredicateDefinition must expose belief-cardinality semantics relevant to a subject/predicate+arguments+temporal scope.

Initial model:

```
belief_cardinality:
  SINGLE_VALUE
  MULTI_HYPOTHESIS
```

### SINGLE_VALUE

A subject may have at most one active BeliefRecord value for the same predicate/arguments/temporal scope.

### MULTI_HYPOTHESIS

Multiple active proposition values are allowed.

Suspicion may use the same or its own compiled cardinality policy.

This is scenario-definition semantics, not runtime guesswork.

---

## 18. BeliefLedger

```
BeliefLedger
  records:
    CanonicalMap<BeliefId, BeliefRecord>
```

Beliefs can end/supersede.

Belief does not imply Fact.

Current active conflict rules derive from PredicateDefinition cardinality.

---

## 19. SuspicionLedger

```
SuspicionLedger
  records:
    CanonicalMap<SuspicionId, SuspicionRecord>
```

Suspicion exists only when mechanically/narratively used.

Do not automatically derive Suspicion from uncertainty.

---

## 20. CommunicationClaimLedger

```
CommunicationClaimLedger
  claims:
    CanonicalMap<CommunicationClaimId, CommunicationClaim>
```

CommunicationClaim records assertion occurrence.

It does not prove reception or truth.

Actual reception is ObservationRecord.

---

## 21. EvidenceGraph

```
EvidenceGraph
  relations:
    CanonicalMap<EvidenceRelationId, EvidenceRelation>
```

Evidence objects are usually Entity records.

Relations support/contradict PropositionValue without automatic inference.

---

## 22. InformationClassificationState

Models SECRET/visibility/access classification without redefining truth.

```
InformationClassificationState
  classifications:
    CanonicalMap<ClassificationId, InformationClassification>
```

```
InformationClassification
  classification_id

  subject_ref:
    PropositionKey
    | EvidenceRef
    | EntityInformationRef
    | NarrativeThreadRef

  policy_ref

  status:
    ACTIVE
    RELEASED
    REVOKED

  optional changed_by_event_ref
```

Rules:

1. classification does not itself grant Knowledge;
2. player visibility remains derived from epistemics + permissions + classification policy;
3. static classification policy may come from bundle;
4. runtime release/reclassification is canonical only when future rules depend on it;
5. Secret is not a Fact type.

---

## 23. GoalLedger

```
GoalLedger
  goals:
    CanonicalMap<GoalInstanceId, GoalInstance>
```

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

  optional irreducible_progress_state
```

Rules:

- store progress only when it is a true canonical variable required by mechanics;
- purely derived progress belongs in GoalView/secondary projection;
- terminal status does not reopen; recurrence creates a new GoalInstance unless definition explicitly models reactivation.

---

## 24. CommitmentLedger

```
CommitmentLedger
  commitments:
    CanonicalMap<CommitmentInstanceId, CommitmentInstance>
```

```
CommitmentInstance
  commitment_instance_id
  definition_ref

  obligor_refs: CanonicalSet<Ref>
  beneficiary_refs: CanonicalSet<Ref>

  terms
  optional due_logical_tick

  status:
    ACTIVE
    FULFILLED
    BREACHED
    CANCELLED
    EXPIRED

  established_by_event_ref
  optional resolved_by_event_ref
```

Rules:

1. CommitmentInstance begins only when an obligation is mechanically established.
2. A proposal/offer is not automatically a CommitmentInstance.
3. If negotiation offers need generic mechanics, model them separately rather than overload commitment state.
4. terminal commitment status does not reopen; new obligation creates a new instance.

---

## 25. SchedulerState

```
SchedulerState
  scheduled_effects:
    CanonicalMap<ScheduleId, ScheduledEffect>
```

No due_index inside CanonicalStateContent.

Invariants:

- fire/cancel once;
- deterministic due semantics;
- BoundaryOrderingPolicy governs action-frontier collision;
- active window prevents independent scheduled canonical commit.

---

## 26. ProgressionState

```
ProgressionState
  progression_markers

  decision_point_instances
  narrative_thread_instances

  optional active_resolution_window

  ending_state
```

At most one action-collecting ResolutionWindow is active.

---

## 27. DecisionPointInstance

Uses accepted contract semantics.

Multiple instances can reference one immutable DecisionPointDefinition.

Instance identity is never confused with definition identity.

---

## 28. NarrativeThreadInstance

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

  optional irreducible_stage

  activated_by_event_ref
  optional resolved_by_event_ref
```

Store stage only if it is an explicit canonical variable.

Purely derived tension/priority belongs in projection/director input.

---

## 29. EndingState

```
EndingState
  status:
    NOT_REACHED
    REACHED

  optional ending_ref
  optional reached_by_transition_key
```

Terminal invariant:

> A committed terminal ending/COMPLETED Session may not retain an OPEN canonical ResolutionWindow.

The same TransitionPlan must resolve, cancel or interrupt the Window and close/cancel its pending input gate under defined policy.

No input gate remains ACCEPTING after terminal Session state.

---

## 30. ScenarioMetricState

```
ScenarioMetricState
  metrics:
    CanonicalMap<MetricRef, CanonicalScalar>
```

Rules:

- only mechanically meaningful metrics;
- fixed-point/unit semantics come from MetricDefinition;
- avoid universal psychology vector;
- presentation-only scores belong in projections.

---

## 31. SessionLifecycleState

```
CREATED
WAITING_FOR_PARTICIPANTS
READY
ACTIVE
PAUSED
COMPLETED
ABANDONED
```

Canonical when lifecycle changes legal gameplay.

Transport connectivity remains operational PresenceState.

---

## 32. SessionStateProjection

```
SessionStateProjection
  session_id
  stream_revision

  mechanics_version_manifest_ref
  mechanics_version_manifest_hash

  state_hash

  content: CanonicalStateContent

  optional derived_indexes
```

`derived_indexes` are excluded from state_hash and may be omitted/rebuilt.

---

## 33. SessionEventStream

```
SessionEventStream
  session_id
  OrderedList<CanonicalEventContent>
```

Historical canonical authority.

Invariants:

- append-only;
- revisions follow accepted convention;
- events reduce revision-0 state into current CanonicalStateContent;
- event batch ownership by CanonicalTransitionRecord;
- replay reproduces state hash.

---

## 34. CanonicalTransitionHistory

```
CanonicalTransitionHistory
  transitions:
    OrderedList<CanonicalTransitionRecord>
```

Transition record is causal/hash evidence, not a second source of world state.

World-state truth comes from event content.

---

## 35. PendingInputAggregate

```
PendingInputAggregate
  window_input_gate
  action_submissions
  optional frozen_input_set

  command_processing_records
  resolution_attempts
```

Created/activated when canonical WindowOpen commits.

Archived after canonical Window resolve/cancel/interrupt.

Concurrency:

- compare-and-set SlotInputRevision;
- base Window/revision must still be active;
- secret input policy enforced independently from player-world visibility.

---

## 36. CommandProcessingStore

```
CommandProcessingStore
  records:
    CanonicalMap<(SessionId, CommandKey), CommandProcessingRecord>
```

"CanonicalMap" here describes deterministic logical uniqueness, not canonical world-state hashing.

---

## 37. TransitionEvidenceStore

```
TransitionEvidenceStore
  transition_records
  resolution_records
  optional debug_artifacts
```

RNG evidence must remain available according to replay policy.

Avoid duplicating the same semantic result as independently mutable truth in multiple stores.

---

## 38. PlayerInteraction domain

PlayerInteractionView derives from:

- SessionStateProjection;
- ParticipationState;
- authorized pending-input state;
- bundle definitions.

A view may change while StreamRevision stays constant because own pending input changes.

PlayerViewKey captures the full mechanically relevant authorized view.

Mechanical InteractionSurface is not owned by Narrative Direction.

---

## 39. Presentation domain

```
PresentationAggregate
  presentation_plan
  presentation_record
```

PresentationRecord stores delivered causal evidence.

Presentation cannot become canonical authority.

ActionSubmission may reference prior PlayerView/Presentation for audit causality.

---

## 40. Operations

### TransactionalOutboxItem

Durable post-commit work.

### PresenceState

Transient connectivity/presence.

### Analytics

Privacy-minimized derived product data.

### Logs

Operational diagnostics with redaction.

None may mutate canonical state except by producing a new authorized EngineCommand through normal boundaries.

---

## 41. Reconstruction contract

Mechanical reconstruction requires:

```
CompiledScenarioBundle
+ SessionGenesis
+ ordered SessionEventStream
= SessionStateProjection at revision N
```

It MUST NOT require:

- WindowInputGate;
- nonselected submissions;
- CommandProcessingRecord;
- ResolutionAttempt;
- CanonicalTransitionHistory;
- PresentationRecord;
- Outbox;
- PresenceState;
- Analytics.

Transition/Resolution evidence is required for verification/forensic replay according to accepted replay policy, but not basic state reconstruction.

---

## 42. Retention classes

### Mechanical reconstruction required

- SessionGenesis;
- pinned bundle/version availability or immutable content;
- canonical Session events.

### Resolver verification / forensic required according to support policy

- FrozenInputSet selected submissions;
- ResolutionRecord;
- NamedRandomDraw evidence;
- CanonicalTransitionRecord;
- relevant build/version artifacts.

### Player-experience forensic evidence

- delivered PresentationRecord;
- optionally materialized PlayerInteractionView.

### Optional/privacy-policy audit

- replaced/nonselected submissions;
- raw free-text;
- parser candidates;
- detailed worker/debug traces.

Deleting optional audit material must not break state replay.

---

## 43. Definition-to-runtime mapping

```
ComponentDefinition -> ComponentInstance
EntityArchetypeDefinition -> Entity
RoleDefinition -> ParticipantBinding
RelationshipDefinition -> Relationship
PredicateDefinition -> PropositionValue
InformationClassificationDefinition -> InformationClassification
ActionDefinition -> ExpandedAction
GoalDefinition -> GoalInstance
CommitmentDefinition -> CommitmentInstance
NarrativeThreadDefinition -> NarrativeThreadInstance
DecisionPointDefinition -> DecisionPointInstance
ResolutionWindowTemplate -> ResolutionWindow
MetricDefinition -> ScenarioMetricState entry
EndingDefinition -> EndingState
```

Definitions immutable; runtime instances canonical when mechanically active.

---

## 44. Canonical mutation boundary

Only a validated TransitionPlan may produce committed canonical changes.

```
EngineCommand / deterministic system trigger
  -> prepare/freeze authoritative inputs
  -> build TransitionPlan
  -> validate transient after-state
  -> canonicalize/hash
  -> atomic commit events + projection + evidence + outbox
```

No normal direct patch of SessionStateProjection.

---

## 45. Aggregate invariants

### Session

1. one pinned bundle per Session;
2. one immutable SessionGenesis if Domain Contracts v0.3 accepted;
3. one canonical revision frontier;
4. at most one active action-collecting Window;
5. current projection revision equals last canonical event revision;
6. terminal Session rejects normal gameplay commands;
7. terminal Session has no OPEN Window/input gate.

### Participation

8. gameplay role/character binding canonical;
9. network disconnect not canonical by itself.

### Entity/Relationship

10. schema-valid components;
11. historical refs survive deactivation;
12. relationship directionality/multiplicity enforced.

### Epistemics

13. PropositionKey content-stable;
14. knowledge provenance explicit;
15. one active Knowledge state per subject/proposition;
16. belief/suspicion multiplicity follows PredicateDefinition;
17. assertion != reception != belief != knowledge != fact;
18. Secret/classification != truth.

### Goal/Commitment/Thread

19. store irreducible canonical progress only;
20. terminal instance does not silently reopen.

### Scheduler/Progression

21. scheduled item fires/cancels once;
22. progression converges or aborts;
23. ending/window/input-gate consistency atomic.

---

## 46. Open details

Still deliberately unfrozen:

- opaque ID format;
- concrete claim scope;
- mutation target encoding;
- event-family registry;
- exact fixed-point scales;
- PRNG implementation;
- detailed Relationship component schemas;
- exact Goal/Commitment terms schemas;
- authoring DSL;
- SQL normalization/JSONB split;
- snapshot policy;
- retention windows;
- PlayerInteractionView cache/materialization.

---

## 47. Acceptance dependencies

DOMAIN_MODEL_v0.2 cannot be accepted before:

1. Domain Contracts v0.3 is explicitly accepted, because SessionGenesis/session_seed is required for reconstruction;
2. red-team confirms no new contract/ADR conflict;
3. the following cases pass:
   - reconstruction from Genesis + events only;
   - participant disconnect/presence separation;
   - duplicate knowledge provenance;
   - multi-hypothesis beliefs;
   - commitment offer vs obligation;
   - derived indexes removed from hash;
   - ending closes active window/input gate;
   - secret classification does not grant knowledge.

Only after acceptance may persistence/SQL model begin.
