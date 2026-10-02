# DOMAIN CONTRACTS v0.3

**Status:** PROPOSED  
**Date:** 2026-10-02  
**Supersedes if accepted:** DOMAIN_CONTRACTS_v0.2  
**Governing architecture:** Architecture Baseline v0.3 — ACCEPTED  
**Purpose:** additive correction to accepted v0.2 for deterministic Session bootstrap/RNG ownership; otherwise preserves v0.2 semantics.

---

# 1. Contract rules

These contracts refine, and may not silently contradict, ADR-001 through ADR-014.

## 1.1 Authority

- All authoritative external/runtime inputs enter through EngineCommand.
- Principal is server-bound.
- Client never authors Claims, canonical Mutations, resource costs, resolver priority or permissions.
- Canonical world/progression state changes only through committed canonical transitions/events.
- Admin recovery uses canonical compensating/recovery commands, never ordinary projection patching.

## 1.2 Determinism

Canonical mechanics do not depend on:

- ambient wall clock;
- transport/network arrival ordering;
- unseeded randomness;
- unconstrained floating point;
- unordered collection iteration;
- mutable environment/network/filesystem;
- arbitrary scenario runtime code.

## 1.3 Definition versus instance

Bundle definitions and runtime instances are distinct.

Examples:

- DecisionPointDefinition != DecisionPointInstance
- PredicateDefinition != PropositionValue
- ResolutionWindowTemplate != ResolutionWindow

## 1.4 Canonical versus operational

Keep distinct:

- canonical world/progression state;
- authoritative pending input;
- operational attempt state;
- rebuildable projections;
- presentation/interaction evidence.

---

# 2. Shared scalar and collection semantics

```
type SessionId = OpaqueId
type ParticipantId = OpaqueId
type EntityId = OpaqueId
type WindowId = OpaqueId
type DecisionPointInstanceId = OpaqueId
type ActionSlotId = OpaqueId
type SubmissionId = OpaqueId
type CommandKey = OpaqueStableKey
type AttemptId = OpaqueId
type TransitionKey = OpaqueStableKey
type ResolutionId = OpaqueId
type ScenarioId = OpaqueId
type ScheduleId = OpaqueId
type PresentationId = OpaqueId
type PlayerViewKey = ContentHash
type PropositionKey = ContentHash

type StreamRevision = NonNegativeInteger
type SlotInputRevision = NonNegativeInteger
type BatchIndex = NonNegativeInteger
type LogicalTick = Integer
type SchemaVersion = NonEmptyString
type ContentHash = NonEmptyString
```

## 2.1 Collection semantics

Every replay/hash-relevant collection declares one of:

```
OrderedList<T>
CanonicalSet<T>
CanonicalMap<K,V>
```

### OrderedList

Array order is semantic.

### CanonicalSet

- duplicates forbidden;
- logical order is irrelevant;
- values are canonical-sorted before serialization/hashing.

### CanonicalMap

- unique keys;
- key order is not semantic;
- canonical serialization sorts keys deterministically.

No canonical hash may depend on database/object iteration order.

---

# 3. CanonicalNumber

```
CanonicalNumber =
  IntegerValue
  | FixedPointValue

IntegerValue
  integer
  optional unit_ref

FixedPointValue
  scaled_integer
  scale
  optional unit_ref
```

Rules:

- exact integer semantics;
- scale is explicit;
- no NaN/Infinity;
- no binary-float tolerance in canonical mechanics;
- conversion between units must use compiled deterministic rules.

---

# 4. MechanicsVersionManifest

Pinned to the Session.

```
MechanicsVersionManifest
  engine_build_id
  source_commit

  scenario_id
  scenario_version
  scenario_bundle_hash

  command_contract_version
  action_contract_version
  state_schema_version
  event_contract_version
  mutation_ir_version

  resolver_version
  progression_version

  canonicalization_version
  hash_algorithm

  rng_algorithm
  rng_version
```

## Invariant

A live Session never silently changes MechanicsVersionManifest.

A future supported live-upgrade mechanism requires an explicit ADR.

---

# 4A. SessionGenesis

Immutable bootstrap record for one Session.

```
SessionGenesis
  session_id

  mechanics_version_manifest_ref
  mechanics_version_manifest_hash

  scenario_bundle_hash

  session_seed

  initial_canonical_state_hash

  created_at   // operational only
```

## Semantics

- SessionGenesis is created exactly once per Session.
- `session_seed` is immutable and is the authoritative seed root for NamedRandomDraw derivation.
- `scenario_bundle_hash` MUST match the pinned MechanicsVersionManifest.
- `initial_canonical_state_hash` hashes the revision-0 CanonicalStateContent derived deterministically from the pinned CompiledScenarioBundle and immutable Genesis fields only.
- Revision 0 MUST NOT depend on implicit external/account state.
- Dynamic participant/role bindings established after Session creation are canonical Domain Events at revision 1+.
- Canonical random setup MUST NOT occur as hidden bootstrap mutation. If a scenario needs random initialization, it runs as an explicit first canonical transition using NamedRandomDraw evidence.
- `created_at` is operational metadata and is excluded from canonical state/hash semantics.
- SessionGenesis is required for deterministic state/resolver reconstruction but is not itself a Domain Event.
- Canonical event StreamRevision starts at 1 after the revision-0 bootstrap state.

## Replay bootstrap

Mechanical reconstruction begins from:

```
CompiledScenarioBundle static initial state
+ SessionGenesis immutable deterministic metadata
= revision-0 CanonicalStateContent
```

Then ordered canonical Domain Events produce revisions 1+.

Any dynamic participant/role binding established after revision 0 MUST be represented through canonical Session events.

Revision-0 state is therefore deliberately boring and reproducible: no hidden randomization, no mutable account lookup, no wall-clock-derived gameplay state.

---

# 5. PresentationGenerationContext

Pinned per presentation-generation request, not mechanically to the Session.

```
PresentationGenerationContext
  presentation_policy_version
  director_version
  realizer_version
  template_version

  optional provider
  optional model
  optional prompt_version

  localization_version
```

An outbox retry MUST use the same PresentationGenerationContext selected when the task was created.

---

# 6. Principal

```
Principal =
  ParticipantPrincipal
  | SystemPrincipal
  | AdminRecoveryPrincipal

ParticipantPrincipal
  participant_id

SystemPrincipal
  system_role:
    DEADLINE_SERVICE
    INTERNAL_WORKER
    SYSTEM_TRIGGER

AdminRecoveryPrincipal
  operator_ref
  recovery_reason
  recovery_policy_ref
```

## Invariants

- ParticipantPrincipal comes from authenticated server context.
- Client payload cannot override Principal.
- AdminRecovery may invoke only registered canonical recovery/compensation commands.
- Normal recovery never edits historical events/current projection directly.

---

# 7. EngineCommand

```
EngineCommand
  command_key: CommandKey
  optional attempt_id: AttemptId

  command_contract_version
  command_type
  session_id
  principal

  optional expected_revision
  optional correlation_ref
  optional causation_ref

  semantic_payload
  semantic_payload_hash

  received_at   // operational only
```

## CommandKey

CommandKey is the semantic idempotency identity.

Retries/duplicate deliveries of the same logical command MUST reuse the same CommandKey.

System-triggered commands use deterministic semantic keys where possible.

Example:

```
deadline:<session>:<window>:<deadline_instance>
```

## CommandResult

```
CommandResult
  command_key

  status:
    APPLIED
    ACCEPTED_PENDING
    REJECTED
    NO_OP

  optional rejection_code
  optional resulting_revision
  optional created_refs
```

## Idempotency rules

1. First processing stores semantic_payload_hash + CommandResult.
2. Exact retry with same CommandKey and same semantic_payload_hash returns the stored CommandResult.
3. Same CommandKey with a different semantic_payload_hash is rejected as `IDEMPOTENCY_KEY_REUSE`.
4. Duplicate delivery does not produce a second canonical transition.

---

# 8. CommandProcessingRecord

Durable application/idempotency state for resumable command execution.

```
CommandProcessingRecord
  command_key
  semantic_payload_hash

  processing_status:
    RECEIVED
    ACCEPTED_PENDING
    PROCESSING
    COMPLETED
    REJECTED
    FAILED_RETRYABLE
    FAILED_TERMINAL

  optional stored_command_result
  optional frozen_input_set_hash
  optional transition_key

  first_received_at
  last_updated_at
```

## Rules

- idempotency lookup occurs before side effects;
- a command that freezes pending input but crashes before canonical commit remains resumable;
- retry with the same CommandKey continues/reports the same logical operation rather than creating another freeze/transition;
- CommandResult is final only when the relevant command semantics say so;
- command processing state is operational/application state, not canonical world state.

---

# 9. CanonicalTransitionRecord

Every committed canonical event batch belongs to exactly one CanonicalTransitionRecord.

```
CanonicalTransitionRecord
  transition_key: TransitionKey
  transition_kind

  session_id
  base_revision
  resulting_revision

  optional trigger_command_key
  optional resolution_id
  optional schedule_refs
  optional progression_refs

  before_state_hash
  canonical_event_batch_hash
  after_state_hash

  mechanics_version_manifest_ref

  committed_at   // operational
```

## TransitionKind

Initial set:

```
WINDOW_RESOLUTION
PROGRESSION
SCHEDULED_EFFECT
SESSION_LIFECYCLE
DEADLINE
ADMIN_COMPENSATION
```

Exact registry can expand without changing the rule that every canonical batch has one owning transition.

---

# 10. Stream revision convention

This convention is frozen by Domain Contracts v0.2.

- Initial canonical Session state is revision `0`.
- Each canonical Domain Event consumes exactly one StreamRevision.
- A transition against base revision `N` with `K` events assigns:

```
event[0].stream_revision = N + 1
event[i].stream_revision = N + 1 + i
resulting_revision = N + K
```

- BatchIndex is contiguous `0..K-1`.
- A no-op command emits no canonical transition/event batch and does not advance revision.
- A completed ResolutionWindow normally changes canonical progression/window state and therefore emits at least one event.

---

# 11. ResolutionWindow — canonical instance

ResolutionWindow represents canonical progression state, not operational resolver phases.

```
ResolutionWindow
  window_id
  definition_ref

  mode
  canonical_status

  opened_at_revision
  base_revision

  logical_open_tick
  optional target_close_logical_tick
  optional wall_deadline_descriptor

  decision_point_instance_ref

  action_slots: CanonicalMap<ActionSlotId, ActionSlot>

  close_policy
  timeout_policy
  disconnect_policy
  boundary_ordering_policy
  resolution_policy_ref
```

## WindowMode

```
SIMULTANEOUS
SECRET_SIMULTANEOUS
SEQUENTIAL
SINGLE_ACTOR
CONDITIONAL
```

Automatic canonical progression does not require a fake player ResolutionWindow.

## CanonicalWindowStatus

```
OPEN
RESOLVED
CANCELLED
INTERRUPTED
```

## Invariants

1. Simultaneous/secret-simultaneous windows open at a fixed base revision.
2. Canonical gameplay state remains at that base revision while the window is OPEN.
3. Window resolution/closure is committed in the transition that advances beyond the base revision.
4. One action-collecting canonical window per Session unless superseded by future ADR.
5. Wall deadline never mutates Window directly; it issues an EngineCommand.

---

# 12. WindowInputGate — authoritative pending input state

Operational/authoritative pending state outside canonical world history.

```
WindowInputGate
  window_id
  gate_status
  slot_states: CanonicalMap<ActionSlotId, SlotInputState>

  frozen_input_set_hash
  optional frozen_at
```

## GateStatus

```
ACCEPTING
FROZEN
CANCELLED
```

## SlotInputState

```
SlotInputState
  action_slot_id
  input_revision: SlotInputRevision
  optional current_submission_id
  finalized
```

## Rules

- Submit/Replace commands use compare-and-set against SlotInputRevision.
- Wall-clock `received_at` is never used to sort competing replacements.
- Freezing atomically closes the gate and records the FrozenInputSet.
- After FROZEN, no slot submission can change.
- Gate state is durable authoritative pending-input state but not canonical world history.

---

# 13. FrozenInputSet

```
FrozenInputSet
  window_id
  base_revision

  selected_submissions:
    CanonicalMap<ActionSlotId, SubmissionId>

  slot_input_revisions:
    CanonicalMap<ActionSlotId, SlotInputRevision>

  input_set_hash
```

This artifact is immutable once created and is referenced by ResolutionRecord.

---

# 14. ResolutionAttempt — operational/audit

```
ResolutionAttempt
  attempt_id
  window_id
  frozen_input_set_hash

  status:
    PREPARING
    RESOLVING
    COMMITTING
    COMMITTED
    ABORTED

  optional failure_code
  started_at
  optional finished_at
```

FROZEN/RESOLVING are not committed canonical Window statuses.

A crash can leave an attempt aborted/retryable while the canonical Window is still OPEN at its base revision and the input gate remains FROZEN.

---

# 15. ActionSlot

```
ActionSlot
  action_slot_id
  actor_ref
  participant_ref

  required
  allowed_action_type_refs: CanonicalSet<ActionTypeRef>

  replacement_policy
  optional default_action_template_ref

  pending_visibility_policy
```

## ReplacementPolicy

```
NO_REPLACEMENT
CAS_REPLACE_UNTIL_FREEZE
EXPLICIT_FINALIZE
```

No policy selects submissions by wall-clock timestamp.

## PendingVisibilityPolicy

At minimum supports:

```
PRIVATE_CONTENT
STATUS_ONLY_TO_OTHER_PARTICIPANT
PUBLIC_CONTENT
```

SECRET_SIMULTANEOUS requires private submission content until scenario policy releases it.

---

# 16. ActionSubmission

```
ActionSubmission
  submission_id

  session_id
  resolution_window_id
  action_slot_id

  action_type_id
  parameters
  selected_target_refs

  source_type:
    OPTION
    STRUCTURED_UI
    NATURAL_LANGUAGE_PARSED
    SCRIPTED_TEST
    SYNTHETIC_PLAYER

  optional source_player_view_key
  optional source_presentation_id

  created_by_command_key
  received_at   // operational
```

## Invariants

- immutable after creation;
- replacement creates a new SubmissionId;
- server binds Participant from Principal/ActionSlot;
- pending input is not canonical world state;
- pending content access follows PendingVisibilityPolicy.

---

# 17. ActionDefinition

Compiled immutable bundle definition.

```
ActionDefinition
  action_type_id
  action_definition_version

  parameter_schema_ref

  actor_constraints
  target_addressability_rules

  submission_eligibility_rules

  claim_derivation_rules
  resolution_requirement_rules
  read_write_footprint_rules

  timing_model
  resolver_policy_ref

  semantic_event_templates
  mutation_templates

  optional tags
```

## Submission eligibility

Rules that must hold against frozen base state and cannot be enabled by another same-window action.

Examples:

- correct ActionSlot;
- correct role/actor;
- parameter schema;
- target addressability;
- knowledge required to choose a target/action.

## Resolution requirements

World/resource/capability conditions evaluated during joint interaction resolution and allowed to be enabled/invalidated by other admitted actions.

Examples:

- resource availability;
- target remains operational;
- another action supplies a capability/resource;
- location/control state after declared interaction.

---

# 18. ExpandedAction

```
ExpandedAction
  action_instance_key

  submission_id
  action_slot_id
  action_type_id
  definition_ref

  actor_entity_id
  normalized_parameters
  resolved_target_refs

  derived_claims
  resolution_requirements
  read_write_footprint

  logical_interval
  resolver_policy_ref

  base_revision
```

Deterministic from:

- FrozenInputSet selected submission;
- ActionDefinition;
- frozen Session canonical state;
- MechanicsVersionManifest.

No ambient input.

---

# 19. AdmissionResult

```
AdmissionResult
  submission_id

  status:
    ADMITTED
    REJECTED_INVALID
    REJECTED_UNAUTHORIZED
    REJECTED_STALE
    REJECTED_NOT_ADDRESSABLE
    REJECTED_NOT_OFFERED
    REJECTED_ELIGIBILITY

  optional rule_ref
  optional detail_code
```

A world condition that could be changed by another action in the same window MUST NOT be modeled as an admission rejection.

---

# 20. ResolutionRequirement

Logical contract:

```
ResolutionRequirement
  requirement_key
  requirement_type
  target_ref
  predicate_ref
  satisfaction_policy
```

It is resolver input, not client input.

A failed ResolutionRequirement contributes to ActionOutcome/interaction transformation rather than retroactively invalidating submission authority.

Exact requirement type registry remains open.

---

# 21. ActionOutcome

```
ActionOutcome
  action_instance_key

  status:
    SUCCEEDED
    PARTIAL
    FAILED
    INTERRUPTED
    TRANSFORMED
    NO_EFFECT

  optional semantic_reason_code
  optional contribution_refs
```

Every ADMITTED action has exactly one ActionOutcome.

---

# 22. Claim

```
Claim
  claim_key_within_action

  target_ref
  scope
  mode

  optional quantity: CanonicalNumber
  optional logical_interval
```

## ClaimMode

```
READ
WRITE
CONSUME
RESERVE
PRODUCE
EXCLUSIVE
```

Rules:

- server-derived;
- scope is compiler-known, never arbitrary client JSON path;
- actions sharing a constrained claim domain are grouped even when no pair alone exceeds capacity;
- aggregate resource constraints are evaluated over the complete group.

---

# 23. InteractionEdge and InteractionGroup

```
InteractionEdge
  edge_key
  action_keys: CanonicalSet<ActionInstanceKey>
  interaction_type
  contributing_claim_refs
  optional rule_ref
```

For pairwise edges, `action_keys` contains exactly two actions.

## InteractionType

```
CONTENTION
EXCLUSION
INTERFERENCE
DEPENDENCY
COMPLEMENTARITY
ORDER_SENSITIVE
```

```
InteractionGroup
  group_key
  action_instance_keys: CanonicalSet<ActionInstanceKey>
  edge_keys: CanonicalSet<EdgeKey>
  selected_resolution_strategy_ref
```

Partition/grouping is deterministic.

Independent groups commute at final-state level.

---

# 24. NamedRandomDraw

```
NamedRandomDraw
  draw_key

  transition_key
  rule_ref

  rng_algorithm
  rng_version

  declared_context
  declared_context_hash

  distribution
  parameters
  result
```

## Derivation

A draw is derived only from:

```
session_seed // from SessionGenesis
rng_algorithm
rng_version
transition_key
rule_ref
draw_key
canonical(declared_context)
canonical(parameters)
```

It MUST NOT depend on:

- draw call order;
- list index;
- unrelated draw keys;
- full mutable ResolutionPlan hash.

Same `draw_key` reused with different declared context/parameters in one transition is an error.

---

# 25. ActionResolutionResult

Action-specific deterministic result before generic progression closure.

```
ActionResolutionResult
  resolution_id
  window_id
  base_revision
  frozen_input_set_hash

  admission_results:
    CanonicalMap<SubmissionId, AdmissionResult>

  expanded_actions:
    CanonicalMap<ActionInstanceKey, ExpandedAction>

  interaction_groups:
    CanonicalMap<GroupKey, InteractionGroup>

  named_random_draws:
    CanonicalMap<DrawKey, NamedRandomDraw>

  action_outcomes:
    CanonicalMap<ActionInstanceKey, ActionOutcome>

  action_event_templates_or_drafts

  action_resolution_hash
```

This is not yet the complete canonical transition because scheduler/progression may add canonical consequences.

---

# 26. BoundaryOrderingPolicy

Every ResolutionWindow definition declares how due logical scheduled effects at the frontier interact with its action resolution.

Initial semantic policies:

```
DUE_BEFORE_ACTIONS
ACTIONS_BEFORE_DUE
INTERRUPT_WINDOW
```

## Rules

- ordering is compiled scenario policy;
- no implicit timestamp tie-break exists;
- an interrupt closes/interrupts the Window canonically according to scenario rules;
- due effects and action effects are evaluated inside one Single Canonical Frontier operation.

---

# 27. TransitionPlan

Complete pre-commit canonical plan.

```
TransitionPlan
  transition_key
  transition_kind

  session_id
  base_revision

  optional resolution_id

  named_random_draws

  planned_events: OrderedList<CanonicalEventDraft>

  transient_after_state_content
  resulting_revision

  before_state_hash
  event_batch_hash
  after_state_hash

  required_outbox_work
```

## Invariants

- contains the complete canonical consequences for this frontier operation;
- includes scheduler/progression/window lifecycle consequences;
- validates fully before commit;
- no partial progression/event subset may commit;
- required outbox work is prepared before DB transaction commit.

---

# 28. ResolutionRecord

Resolution-specific immutable evidence referenced by a WINDOW_RESOLUTION CanonicalTransitionRecord.

```
ResolutionRecord
  resolution_id
  session_id
  window_id

  frozen_input_set_hash

  selected_submissions:
    CanonicalMap<ActionSlotId, SubmissionId>

  admission_results
  action_outcomes
  named_random_draws

  interaction_graph_hash
  action_resolution_hash

  mechanics_version_manifest_ref

  optional debug_artifact_ref
```

Commit revisions/hashes belong primarily to CanonicalTransitionRecord, avoiding duplicate transition ownership.

---

# 29. CanonicalEventContent

```
CanonicalEventContent
  session_id
  stream_revision

  event_batch_key
  batch_index

  event_family
  event_code
  event_schema_version

  logical_time

  correlation_ref
  causation_refs: CanonicalSet<Ref>

  actor_refs: CanonicalSet<Ref>
  subject_refs: CanonicalSet<Ref>

  payload
  mutations: OrderedList<Mutation>
```

## Identity

Semantic event identity:

```
(session_id, event_batch_key, batch_index)
```

For one canonical transition, `event_batch_key == transition_key`.

Operational DB IDs are not semantic identity.

---

# 30. EventRecordMetadata

```
EventRecordMetadata
  database_record_id
  recorded_at
  optional writer_instance
  optional debug_trace_ref
```

Excluded from canonical event hash/equality.

---

# 31. CanonicalStateContent and SessionStateProjection

## CanonicalStateContent

Hashable canonical gameplay/progression state only.

```
CanonicalStateContent
  lifecycle
  logical_time

  entities
  relationships
  epistemics

  goals
  commitments

  scheduler
  progression

  optional active_resolution_window

  scenario_metrics
```

Pending submissions, WindowInputGate, ResolutionAttempt, PresentationRecord and operational timestamps are excluded.

## SessionStateProjection

```
SessionStateProjection
  session_id
  stream_revision

  mechanics_version_manifest_ref
  mechanics_version_manifest_hash

  state_hash

  content: CanonicalStateContent
```

## State hash

```
state_hash =
  H(
    mechanics_version_manifest_hash
    || stream_revision
    || canonical(CanonicalStateContent)
  )
```

The hash field itself is not part of CanonicalStateContent.

---

# 32. State Mutation IR v0.2

```
Mutation =
  EntityCreate
  EntityDeactivate

  ComponentSet
  NumberAdjust
  CollectionAdd
  CollectionRemove

  FactAssert
  FactEndValidity

  ObservationRecordAdd
  KnowledgeRecordAdd
  KnowledgeRecordEnd
  BeliefRecordAdd
  BeliefRecordEnd
  SuspicionRecordAdd
  SuspicionRecordEnd
  CommunicationClaimAdd

  ScheduleAdd
  ScheduleCancel
  ScheduleFire

  DecisionPointInstanceActivate
  DecisionPointInstanceDeactivate

  WindowOpen
  WindowResolve
  WindowCancel
  WindowInterrupt

  SessionActivate
  SessionPause
  SessionComplete
  SessionAbandon

  ProgressionMarkerSet
```

Common envelope:

```
Mutation
  mutation_type
  target_ref
  payload
  mutation_schema_version
```

Every mutation type has a typed payload schema.

No arbitrary path mutation.

---

# 33. PredicateDefinition and PropositionValue

## PredicateDefinition

Compiled Scenario Bundle definition.

```
PredicateDefinition
  predicate_ref
  argument_schema
  value_schema
  temporal_semantics
```

## PropositionValue

Immutable runtime semantic value.

```
PropositionValue
  proposition_key

  predicate_ref
  arguments: OrderedList<CanonicalValue>
  value: CanonicalValue

  optional temporal_qualifier
```

## PropositionKey

Content-derived deterministically from canonical PropositionValue fields excluding proposition_key itself.

This supports propositions involving runtime-created entities/values without predeclaring every proposition instance.

---

# 34. TemporalQualifier

Optional explicit proposition semantics:

```
TIMELESS
AT_TICK(tick)
AS_OF_TICK(tick)
DURING_RANGE(start_tick, end_tick)
```

Exact authored syntax may differ; canonical meaning is deterministic.

A character acquiring knowledge at tick 20 is distinct from the proposition being semantically "true as of tick 10."

---

# 35. FactRecord

```
FactRecord
  fact_id
  proposition_key

  valid_from_logical_tick
  optional valid_until_logical_tick

  asserted_by_event_ref
  optional ended_by_event_ref
```

Ending validity never erases historical truth.

Corrections are explicit compensating/correction events.

---

# 36. ObservationRecord

```
ObservationRecord
  observation_id
  observer_entity_id

  observed_ref:
    PropositionKey
    | FactId
    | EvidenceRef
    | EventRef

  observed_at_logical_tick
  provenance_event_ref
```

Observation alone does not automatically grant KnowledgeRecord.

---

# 37. KnowledgeRecord

```
KnowledgeRecord
  knowledge_id
  knower_entity_id
  proposition_key

  acquired_at_logical_tick
  optional ended_at_logical_tick

  provenance_refs: CanonicalSet<Ref>
  granted_by_event_ref
```

Knowledge acquisition time does not redefine the PropositionValue's temporal qualifier.

Unknown = absence of relevant valid knowledge.

---

# 38. BeliefRecord and SuspicionRecord

```
BeliefRecord
  belief_id
  believer_entity_id
  proposition_key

  acquired_at_logical_tick
  optional ended_at_logical_tick

  optional confidence: CanonicalNumber
  provenance_refs
  optional supersedes_belief_ref
```

```
SuspicionRecord
  suspicion_id
  subject_entity_id
  proposition_key

  acquired_at_logical_tick
  optional ended_at_logical_tick

  optional confidence: CanonicalNumber
  provenance_refs
  optional supersedes_suspicion_ref
```

Belief/Suspicion do not imply Fact.

---

# 39. CommunicationClaim

```
CommunicationClaim
  communication_claim_id

  speaker_entity_id
  channel_ref

  optional addressed_to_refs: CanonicalSet<Ref>

  proposition_key

  made_at_logical_tick
  source_event_ref
```

Addressed audience is not equivalent to actual reception.

Actual reception is represented by ObservationRecord and subsequent epistemic events.

---

# 40. EvidenceRelation

```
EvidenceRelation
  evidence_ref
  proposition_key

  relation:
    SUPPORTS
    CONTRADICTS

  optional strength: CanonicalNumber
```

No automatic Bayesian/logical inference is implied.

---

# 41. ScheduledEffect

```
ScheduledEffect
  schedule_id

  status:
    PENDING
    FIRED
    CANCELLED

  source_ref
  created_at_logical_tick

  due:
    LOGICAL_TICK
    | CONDITION_REF

  due_value

  event_template_ref
  effect_parameters

  optional cancellation_rule_ref

  mechanics_version_manifest_ref
```

## Invariants

- due evaluation deterministic;
- PENDING can become FIRED or CANCELLED exactly once;
- firing/cancellation is canonical;
- interaction with an OPEN window obeys BoundaryOrderingPolicy;
- no scheduled effect independently commits through an open simultaneous window.

---

# 42. DecisionPointDefinition

Compiled definition:

```
DecisionPointDefinition
  decision_point_definition_ref

  activation_rule_ref
  deactivation_rule_ref

  action_slot_templates
  resolution_window_template_ref

  optional presentation_policy_ref
```

---

# 43. DecisionPointInstance

Canonical runtime progression instance.

```
DecisionPointInstance
  decision_point_instance_id
  definition_ref

  status:
    ACTIVE
    RESOLVED
    CANCELLED

  activated_at_revision
  activated_at_logical_tick

  optional resolution_window_id
  optional resolved_by_transition_key
```

A definition may produce multiple runtime instances over one Session.

Instance identity is deterministic or created as canonical transition output and then stored.

---

# 44. ProgressionEvaluation

```
ProgressionEvaluation
  transition_key

  starting_state_hash

  rounds: OrderedList<ProgressionRound>

  status:
    STABLE
    SESSION_ENDED
    STEP_BUDGET_EXHAUSTED

  resulting_state_hash
```

## ProgressionRound

```
ProgressionRound
  round_index
  eligible_transition_refs: CanonicalSet<Ref>
  selected_transition_refs: OrderedList<Ref>
  generated_effects
```

## Termination rules

1. Evaluate eligible compiled transitions from transient canonical state.
2. Resolve compatible candidates by declared composition.
3. Resolve mutually exclusive candidates by explicit priority or NamedRandomDraw.
4. Apply selected transition effects to transient state.
5. Repeat until:
   - no candidate remains;
   - Session ending/completion stops progression;
   - compiled progression-step budget is exhausted.
6. STEP_BUDGET_EXHAUSTED aborts the uncommitted canonical transition.
7. Compiler detects illegal/static progression cycles where feasible.

No implicit "first in array" fallback.

---

# 45. PlayerInteractionView

Deterministic audience-safe projection produced before Narrative Direction.

```
PlayerInteractionView
  player_view_key
  session_id
  participant_ref

  state_revision

  optional window_input_gate_hash
  optional own_slot_input_revision

  visible_world_projection
  visible_epistemic_projection

  optional decision_point_instance_ref
  optional resolution_window_ref

  interaction_surface
  own_pending_input_summary
  allowed_shared_readiness_status

  view_schema_version
```

## InteractionSurface

Mechanically authoritative-safe affordances:

```
InteractionSurface
  action_slots
  allowed_action_type_refs
  allowed_target_refs_or_target_query_results
  input_constraints
```

## Invariants

- derived deterministically from canonical state + authorized pending-input state + participant permissions/epistemics;
- PlayerViewKey changes when mechanically relevant authorized pending-input state changes even if StreamRevision does not;
- contains no information unauthorized for participant;
- Narrative Direction cannot add/remove legal mechanics from InteractionSurface;
- SECRET_SIMULTANEOUS never exposes other participant pending submission content.

PlayerViewKey is a content hash over the deterministic view.

---

# 46. PresentationPlan

Narrative/presentation artifact over a PlayerInteractionView.

```
PresentationPlan
  presentation_plan_id

  session_id
  audience_ref
  state_revision

  player_view_key

  related_event_refs
  related_progression_refs

  content_blocks
  presentation_hints

  presentation_policy_version
```

No `available_ui_actions` field exists here.

Mechanical affordances come from PlayerInteractionView.InteractionSurface.

---

# 47. PresentationRecord

```
PresentationRecord
  presentation_id
  session_id
  audience_ref

  state_revision
  player_view_key
  presentation_plan_hash

  generation_context: PresentationGenerationContext

  exact_delivered_payload
  output_hash

  generation_status
  delivery_status:
    PENDING
    DELIVERED
    FAILED
    SUPERSEDED

  generated_at
  optional delivered_at
```

Exact delivered payload may contain structured text/media/UI presentation metadata but cannot alter canonical mechanics.

---

# 48. TransactionalOutboxItem

```
TransactionalOutboxItem
  outbox_id
  session_id

  task_type
  deduplication_key

  payload
  execution_context

  status
  attempt_count

  created_at
  optional available_after
  optional completed_at
```

For BUILD_PRESENTATION:

- payload references state_revision/audience/player_view_key;
- execution_context contains immutable PresentationGenerationContext.
- delivery performs a DeliveryGuard check: if the targeted PlayerInteractionView is no longer applicable/current for that audience, the presentation is marked SUPERSEDED and is not delivered as current gameplay output.

## Invariants

- created in same DB transaction as canonical transition when required;
- retry uses same deduplication key and execution context;
- external side effects are idempotent;
- analytics tasks must not carry raw private world/free-text data by default;
- failure cannot alter canonical state except through a new EngineCommand.

---

# 49. Canonical commit contract

For a state-changing frontier operation:

```
prepare/freeze inputs
build TransitionPlan
validate
canonicalize/hash

BEGIN
  verify Session current revision == TransitionPlan.base_revision

  persist optional ResolutionRecord/evidence
  persist CanonicalTransitionRecord
  append ordered Domain Events
  update SessionStateProjection
  persist required Outbox items
COMMIT
```

All components commit atomically.

A revision mismatch aborts the uncommitted plan.

---

# 50. Session lifecycle

```
CREATED
WAITING_FOR_PARTICIPANTS
READY
ACTIVE
PAUSED
COMPLETED
ABANDONED
```

Proposed transitions:

```
CREATED -> WAITING_FOR_PARTICIPANTS -> READY -> ACTIVE

ACTIVE -> PAUSED
ACTIVE -> COMPLETED
ACTIVE -> ABANDONED

PAUSED -> ACTIVE
PAUSED -> ABANDONED
```

Commercial/account lifecycle remains outside this state machine.

---

# 51. Cross-contract invariants

## Authority

1. Browser cannot append canonical events.
2. Browser cannot author Claims/Mutation IR/canonical costs/priority.
3. Principal is server-bound.
4. Admin recovery is canonical compensation, not projection patching.

## Bootstrap/replay

5. Every Session has exactly one immutable SessionGenesis.
6. SessionGenesis.session_seed is the authoritative seed root used by canonical RNG derivation.
7. Revision-0 canonical state hash is reproducible from CompiledScenarioBundle + SessionGenesis.

## Command/idempotency

8. Same CommandKey + same semantic payload returns original result.
6. Same CommandKey + different payload is rejected.
7. System trigger retry uses same semantic CommandKey.

## Pending input/window

8. OPEN simultaneous Window holds canonical state at fixed base revision.
9. Slot replacement uses InputRevision CAS, never wall-time sorting.
10. FrozenInputSet is immutable.
11. Resolution operational phases do not advance canonical revision.
12. SECRET_SIMULTANEOUS pending contents cannot leak to other player view.

## Admission/resolution

13. Eligibility rules cannot depend on conditions intentionally satisfiable by other same-window actions.
14. ResolutionRequirements may be enabled/invalidated by joint interactions.
15. Every selected Submission has one AdmissionResult.
16. Every ADMITTED action has one ActionOutcome.
17. Aggregate resource constraints are group-level.
18. Independent groups commute at final-state level.

## Canonical transition/events/state

19. Every canonical event batch has exactly one CanonicalTransitionRecord.
20. Every canonical state change is attributable to ordered canonical events.
21. Event revisions follow the frozen convention.
22. Canonical sets/maps have deterministic serialization semantics.
23. Event batch + transition record + projection + required outbox commit atomically.
24. State hash has no self-reference.
25. Replay of canonical events reproduces SessionStateProjection.state_hash.

## Time/scheduler/progression

26. Wall clock never directly mutates historical mechanics.
27. BoundaryOrderingPolicy resolves due-effect/action ordering.
28. Progression has deterministic candidate selection and bounded convergence.
29. Progression non-convergence aborts before canonical commit.

## Random/numeric

30. Canonical numbers use exact integer/fixed-point semantics.
31. Every canonical random decision has NamedRandomDraw evidence.
32. Draw derivation excludes unrelated draw/order context.

## Epistemics

33. Runtime propositions use deterministic PropositionKey.
34. Knowledge has explicit provenance.
35. CommunicationClaim does not imply actual reception/knowledge.
36. Fact validity ending does not erase historical observations/knowledge.

## Presentation

37. PlayerInteractionView is audience-safe before Narrative Direction/Realization.
38. Mechanical InteractionSurface cannot be changed by Narrative Direction.
39. Presentation task retry preserves PresentationGenerationContext.
40. Delivered output is recoverable via PresentationRecord.
41. Human ActionSubmission may retain source PlayerView/Presentation provenance.
42. A stale presentation task cannot deliver itself as the current interaction after its PlayerInteractionView has been superseded.
43. A command crash after input freeze can resume from CommandProcessingRecord/FrozenInputSet without creating a second logical command or transition.

---

# 52. Red-team outcome against original 18 cases

v0.2 is designed to resolve the v0.1 failures as follows:

1. scarce battery — grouped aggregate capacity resolution;
2. sabotage vs transmit — interference/order-sensitive group;
3. independent actions — commutativity invariant;
4. hidden EntityId — addressability + InteractionSurface;
5. replacement near deadline — SlotInputRevision CAS + input gate freeze;
6. duplicate deadline — stable CommandKey;
7. scheduled effect due during open window — BoundaryOrderingPolicy;
8. three-way capacity — aggregate group;
9. same-window dependency — ResolutionRequirement, not admission predicate;
10. lie believed while canon false — distinct CommunicationClaim/Belief/Fact;
11. evidence changes belief — superseding belief records;
12. fact validity ends — historical records remain;
13. Realizer crash — transactional outbox;
14. outbox retry — dedup + immutable execution context;
15. unrelated RNG draw — narrow named derivation;
16. wording changes — presentation outside canonical hash;
17. old build replay — MechanicsVersionManifest + forensic replay inherited from ADR-011;
18. ending + progression trigger — deterministic rounds/composition/priority + termination budget.

---

# 53. Deliberately open details

Still not frozen:

- opaque ID encoding;
- exact Claim.scope representation;
- exact Mutation target representation;
- concrete event-family registry;
- fixed-point scales/units per scenario metric;
- PRNG implementation;
- resolution strategy parameter schemas;
- concrete Relationship/Goal/Commitment contracts;
- authoring DSL syntax;
- SQL tables/indexes;
- snapshot strategy;
- audit/presentation retention periods;
- exact progression step budget default;
- exact PlayerInteractionView storage/caching policy.

---

# 54. Acceptance gate

Acceptance review completed against the following gate:

1. verify it does not conflict with ADR-001..ADR-014;
2. verify the 18 synthetic cases are contractually unambiguous;
3. red-team:
   - zero-event/no-op transitions;
   - crash between input freeze and canonical commit;
   - repeated DecisionPoint instances;
   - communication interception;
   - stale presentation retry;
   - canonical collection hashing;
   - progression non-convergence;
4. decide whether any remaining open item is implementation-blocking.

v0.3 remains PROPOSED until explicitly accepted. SQL schema remains blocked.


---

# 55. Acceptance record

Domain Contracts v0.2 were explicitly accepted on 2026-10-02.

Acceptance evidence:
- `RED_TEAM_DOMAIN_CONTRACTS_v0.2.md`
- `POSTMORTEM_DOMAIN_CONTRACTS_v0.1.md`

These contracts are now binding implementation constraints under Architecture Baseline v0.3 and ADR-001 through ADR-014.

A later design that conflicts with these contracts must be identified as a contract change and cannot be silently treated as a persistence/implementation detail.


---

# 55. v0.3 change record

Domain Contracts v0.3 changes only the deterministic Session bootstrap contract relative to accepted v0.2.

Added:
- SessionGenesis;
- explicit durable ownership of session_seed;
- revision-0 bootstrap reconstruction rule;
- explicit prohibition on hidden/random/external bootstrap state;
- bootstrap/replay invariants.

All other accepted Domain Contracts v0.2 semantics remain unchanged unless explicitly restated here.

## Reason

Domain Model v0.1 postmortem found that ADR-011 requires Session seed retention and NamedRandomDraw uses `session_seed`, but Domain Contracts v0.2 did not assign that seed to an explicit durable owner.

## Acceptance dependency

If accepted, v0.3 supersedes v0.2 as the contract baseline without changing ADR-001 through ADR-014.
