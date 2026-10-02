# DOMAIN CONTRACTS v0.1

**Status:** PROPOSED  
**Date:** 2026-10-02  
**Governing architecture:** Architecture Baseline v0.3 — ACCEPTED  
**Purpose:** concrete runtime/domain contracts before database schema or production code.

---

# 1. Contract design principles

These contracts must preserve the accepted architecture.

## 1.1 Authority

- Client input is never canonical authority by itself.
- Principal identity is server-bound.
- Claims, effects, costs and resolver policy are server/scenario-derived.
- Canonical state changes only through committed event batches.

## 1.2 Determinism

Canonical contracts avoid:
- ambient wall clock;
- unseeded randomness;
- unconstrained floating point;
- implicit platform ordering;
- arbitrary runtime code.

## 1.3 Versionability

Every externally persisted or replay-relevant contract has an explicit schema/contract version.

## 1.4 Separation

Keep separate:
- command/input;
- pending submissions;
- expanded resolution inputs;
- canonical event content;
- operational metadata;
- current projections;
- presentation/audit artifacts.

## 1.5 No SQL assumptions

These are logical contracts.

Field grouping does not imply one table/document per contract.

---

# 2. Shared scalar/value types

The concrete programming-language representation remains implementation work, but semantics are defined here.

```
type SessionId = OpaqueId
type ParticipantId = OpaqueId
type EntityId = OpaqueId
type WindowId = OpaqueId
type ActionSlotId = OpaqueId
type SubmissionId = OpaqueId
type CommandId = OpaqueId
type ResolutionId = OpaqueId
type ScenarioId = OpaqueId
type PropositionId = OpaqueId
type ScheduleId = OpaqueId
type PresentationId = OpaqueId

type StreamRevision = NonNegativeInteger
type BatchIndex = NonNegativeInteger
type LogicalTick = Integer
type SchemaVersion = NonEmptyString
type ContentHash = NonEmptyString
```

## 2.1 Canonical numeric value

Canonical gameplay numbers MUST use an explicit representation.

```
CanonicalNumber =
  | IntegerValue
  | FixedPointValue

IntegerValue:
  integer: exact integer
  optional unit

FixedPointValue:
  scaled_integer: exact integer
  scale: positive integer
  optional unit
```

Example:

```
{ scaled_integer: 735, scale: 1000, unit: "trust" }
```

represents 0.735 without binary floating-point semantics.

No NaN/Infinity.

---

# 3. VersionManifest

Pinned to a Session.

```
VersionManifest
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

  director_version
  realizer_version
```

Narrative prompt/template/model versions may additionally be stored per PresentationRecord because they can vary independently.

## Invariant

A Session never silently changes its VersionManifest.

Any supported live upgrade policy would require an explicit future ADR.

---

# 4. Principal

```
Principal =
  | ParticipantPrincipal
  | SystemPrincipal
  | AdminRecoveryPrincipal

ParticipantPrincipal
  participant_id

SystemPrincipal
  system_role:
    SCHEDULER
    DEADLINE_SERVICE
    PROGRESSION_ENGINE
    INTERNAL_WORKER

AdminRecoveryPrincipal
  operator_ref
  recovery_reason
```

## Invariants

- ParticipantPrincipal is derived from authenticated server context.
- Client payload cannot override Principal.
- AdminRecovery commands require explicit audit trail and constrained command types.

---

# 5. EngineCommand

```
EngineCommand
  command_id: CommandId
  command_contract_version
  command_type
  session_id
  principal

  optional expected_revision
  optional correlation_ref
  optional causation_ref

  payload

  received_at   // operational only
```

Initial command families:

```
SubmitAction
ReplaceAction
CloseResolutionWindow
DeadlineElapsed
ResumeSession
ApplyAutomaticProgression
```

Exact list remains extensible.

## CommandResult

```
CommandResult
  command_id
  status:
    APPLIED
    ACCEPTED_PENDING
    REJECTED
    DUPLICATE

  optional rejection_code
  optional resulting_revision
  optional created_refs[]
```

## Invariants

1. `command_id` is idempotent.
2. Duplicate processing returns the historical result and does not duplicate effects.
3. `received_at` cannot affect canonical rule outcomes except through an explicitly transformed command such as DeadlineElapsed.
4. Expected-revision mismatch is a controlled rejection/conflict, not an implicit retry with changed semantics.

---

# 6. ResolutionWindow

```
ResolutionWindow
  window_id
  mode
  status

  opened_at_revision
  base_revision

  logical_open_tick
  optional logical_close_tick
  optional wall_deadline

  action_slots[]
  close_policy
  disconnect_policy
  timeout_policy
  resolution_policy_ref
```

## WindowMode

```
SIMULTANEOUS
SECRET_SIMULTANEOUS
SEQUENTIAL
SINGLE_ACTOR
CONDITIONAL
AUTOMATIC
```

## WindowStatus

```
OPEN
FROZEN
RESOLVING
RESOLVED
CANCELLED
INTERRUPTED
```

## State transitions

```
OPEN
  -> FROZEN
  -> CANCELLED
  -> INTERRUPTED

FROZEN
  -> RESOLVING

RESOLVING
  -> RESOLVED

INTERRUPTED
  -> optional future new window through canonical progression
```

No transition returns a Window to OPEN.

## Invariants

1. For simultaneous modes, `base_revision == opened_at_revision`.
2. Canonical gameplay state does not advance while an OPEN simultaneous window collects submissions.
3. After FROZEN, selected submissions cannot change.
4. One active canonical action-collecting window per Session unless a future ADR explicitly introduces nested windows.
5. Wall deadline causes a command; it does not mutate status directly.

---

# 7. ActionSlot

```
ActionSlot
  action_slot_id
  actor_ref
  participant_ref
  required: boolean
  allowed_action_type_refs[]
  replacement_policy
  optional default_action_template_ref
```

## ReplacementPolicy

Initial proposal:

```
NO_REPLACEMENT
LATEST_VALID_SUBMISSION
EXPLICIT_FINALIZE
```

Exact policy vocabulary remains PROPOSED.

---

# 8. ActionSubmission

```
ActionSubmission
  submission_id
  session_id
  resolution_window_id
  action_slot_id

  action_type_id
  parameters
  selected_target_refs[]

  source:
    OPTION
    STRUCTURED_UI
    NATURAL_LANGUAGE_PARSED
    SCRIPTED_TEST
    SYNTHETIC_PLAYER

  received_at   // operational
```

The participant is derived from the EngineCommand Principal and ActionSlot binding rather than trusted as a free payload field.

## Invariants

1. Submission is immutable after creation.
2. Replacement creates a new SubmissionId.
3. Submission is pending input, not canonical world history.
4. Final selected submission per ActionSlot is frozen into ResolutionRecord.

---

# 9. ActionDefinition

Contained in CompiledScenarioBundle.

```
ActionDefinition
  action_type_id
  action_definition_version

  parameter_schema_ref
  actor_constraints
  target_rules

  admission_predicates[]
  claim_derivation_rules[]
  read_write_footprint_rules[]

  timing_model
  resolver_policy_ref

  semantic_event_templates[]
  mutation_templates[]

  optional tags[]
```

## Invariants

- Definition is immutable for one Scenario Bundle hash.
- All referenced rules are pure/compiled.
- Mutation templates can only target allowed state domains.
- Tags do not dispatch mechanics.

---

# 10. ExpandedAction

```
ExpandedAction
  action_instance_key
  submission_id
  action_slot_id
  action_type_id
  definition_ref

  actor_entity_id
  normalized_parameters
  resolved_target_refs[]

  derived_claims[]
  read_write_footprint
  logical_interval

  resolver_policy_ref
  base_revision
```

## Invariants

1. Deterministic function of:
   - frozen submission;
   - ActionDefinition;
   - frozen SessionState;
   - pinned VersionManifest.
2. `action_instance_key` is deterministic or persisted as frozen resolution evidence.
3. No ambient data may influence expansion.

---

# 11. AdmissionResult

```
AdmissionResult
  submission_id
  status

  optional rejection_code
  optional rule_ref
  optional detail_code
```

## Status

```
ADMITTED
REJECTED_INVALID
REJECTED_UNAUTHORIZED
REJECTED_STALE
REJECTED_NOT_ADDRESSABLE
REJECTED_PRECONDITION
```

A rejected submission does not receive an ActionOutcome.

---

# 12. ActionOutcome

```
ActionOutcome
  action_instance_key
  status

  optional semantic_reason_code
  optional contribution_refs[]
```

## Status

```
SUCCEEDED
PARTIAL
FAILED
INTERRUPTED
TRANSFORMED
NO_EFFECT
```

## Invariant

Every ADMITTED ExpandedAction gets exactly one ActionOutcome in the ResolutionPlan.

---

# 13. Claim

```
Claim
  claim_id_within_action
  target_ref
  scope
  mode

  optional quantity
  optional unit
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

## Scope

Exact syntax remains open.

Semantically it must identify one of:

- entire Entity;
- specific Component;
- specific declared Component field/domain;
- named ResourcePool;
- named Channel;
- named Location capacity/control domain;
- other compiler-known claimable domain.

Arbitrary client-supplied JSON paths are forbidden.

## Invariants

- Claims are server-derived.
- Quantity uses CanonicalNumber.
- Compatibility/interactions are versioned.
- Aggregate contention is evaluated across the whole interaction group, not only pairwise.

---

# 14. InteractionEdge

```
InteractionEdge
  left_action_key
  right_action_key
  interaction_type

  optional contributing_claim_refs[]
  optional rule_ref
```

## InteractionType

```
CONTENTION
EXCLUSION
INTERFERENCE
DEPENDENCY
COMPLEMENTARITY
ORDER_SENSITIVE
```

Multiple interaction reasons may exist between the same pair.

---

# 15. InteractionGroup

```
InteractionGroup
  group_key
  action_instance_keys[]
  edge_refs[]
  selected_resolution_strategy_ref
```

## Invariant

The partition into groups is deterministic for the same ExpandedAction set + bundle/version.

Independent groups must commute at final-state level.

---

# 16. NamedRandomDraw

```
NamedRandomDraw
  draw_key

  rule_ref
  resolution_or_progression_ref

  rng_algorithm
  rng_version
  derivation_context_hash

  distribution
  parameters

  result
```

## Invariants

1. Draw result is reproducible from pinned seed/version/context.
2. Unrelated draw insertion does not perturb this draw.
3. Every canonical random choice has an explicit NamedRandomDraw record/evidence.
4. No ambient RNG exists in canonical code.

---

# 17. ResolutionPlan

Pre-commit deterministic artifact.

```
ResolutionPlan
  resolution_id
  session_id
  window_id
  base_revision

  selected_submission_ids[]
  admission_results[]
  expanded_actions[]
  interaction_groups[]

  named_random_draws[]
  action_outcomes[]

  planned_events[]
  planned_progression_transitions[]

  before_state_hash
  optional planned_after_state_hash
```

## Invariants

- one AdmissionResult per selected submission;
- one ActionOutcome per ADMITTED action;
- all interactions resolved;
- planned mutations validate against schemas;
- no canonical commit has happened yet.

ResolutionPlan itself may be stored wholly or represented through ResolutionRecord + hashes depending on persistence design.

---

# 18. ResolutionRecord

Immutable audit/replay evidence.

```
ResolutionRecord
  resolution_id
  session_id
  window_id

  base_revision
  committed_from_revision
  committed_through_revision

  selected_submission_ids[]
  admission_results[]
  action_outcomes[]
  named_random_draws[]

  interaction_graph_hash
  resolution_plan_hash

  before_state_hash
  canonical_event_batch_hash
  after_state_hash

  version_manifest_ref
  committed_at   // operational
```

Optional debug detail can be stored separately if large.

---

# 19. CanonicalEventContent

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
  causation_refs[]

  actor_refs[]
  subject_refs[]

  payload
  mutations[]
```

## Event identity

Semantic identity proposal:

```
(session_id, event_batch_key, batch_index)
```

Database surrogate IDs may exist but do not define semantic equality.

## Invariants

1. StreamRevision strictly increases according to the chosen revision convention.
2. Batch indexes are contiguous from 0.
3. Event content is deterministic/hashable.
4. No `recorded_at` inside CanonicalEventContent.

---

# 20. EventRecordMetadata

```
EventRecordMetadata
  database_record_id
  recorded_at
  optional writer_instance
  optional debug_trace_ref
```

Excluded from semantic event hash/replay equality.

---

# 21. State Mutation IR v0.1 proposal

Exact target encoding remains open; operation semantics are proposed here.

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
  ObservationRecordMutation
  KnowledgeAdd
  KnowledgeEnd
  BeliefAdd
  BeliefEnd
  SuspicionAdd
  SuspicionEnd
  CommunicationClaimRecord

  ScheduleAdd
  ScheduleCancel

  DecisionPointActivate
  DecisionPointDeactivate
  WindowOpen
  WindowClose
  SessionComplete
  SessionPause
  ProgressionMarkerSet
```

## Common mutation envelope

```
Mutation
  mutation_type
  target_ref
  payload
  mutation_schema_version
```

## Invariants

- No arbitrary path mutation.
- Every mutation type has a typed payload schema.
- Every target domain is compiler/runtime validated.
- Mutations in an EventBatch are atomic.
- A mutation cannot bypass accepted engine invariants.

---

# 22. Proposition

```
Proposition
  proposition_id
  predicate
  arguments[]
  value
  proposition_schema_version
```

Value may be boolean, enum, integer/fixed-point, entity reference or another explicitly schema-allowed canonical scalar.

## Invariant

Proposition identity/value semantics are stable within the pinned Scenario Bundle.

---

# 23. FactRecord

```
FactRecord
  fact_id
  proposition_id

  valid_from_logical_tick
  optional valid_until_logical_tick

  asserted_by_event_ref
  optional ended_by_event_ref
```

## Invariants

- Closing validity does not erase the historical FactRecord.
- Corrections use explicit compensating/correction semantics.
- Do not create FactRecords for every ordinary Component value unless epistemic reference requires it.

---

# 24. ObservationRecord

```
ObservationRecord
  observation_id
  observer_entity_id

  observed_ref:
    proposition_id | evidence_ref | event_ref

  observed_at_logical_tick
  provenance_event_ref
```

Observation does not automatically equal knowledge unless deterministic rules grant knowledge.

---

# 25. KnowledgeRecord

```
KnowledgeRecord
  knowledge_id
  knower_entity_id
  proposition_id

  valid_from_logical_tick
  optional valid_until_logical_tick

  provenance_refs[]
  granted_by_event_ref
```

Unknown = absence of a currently valid KnowledgeRecord.

---

# 26. BeliefRecord

```
BeliefRecord
  belief_id
  believer_entity_id
  proposition_id

  valid_from_logical_tick
  optional valid_until_logical_tick

  optional confidence: CanonicalNumber
  provenance_refs[]
```

Belief is independent of canonical truth.

---

# 27. SuspicionRecord

```
SuspicionRecord
  suspicion_id
  subject_entity_id
  proposition_id

  valid_from_logical_tick
  optional valid_until_logical_tick

  optional confidence: CanonicalNumber
  provenance_refs[]
```

The exact product distinction between Belief and Suspicion must remain semantically meaningful in scenario rules; do not store both if a scenario never uses the distinction.

---

# 28. CommunicationClaim

```
CommunicationClaim
  communication_claim_id

  speaker_entity_id
  recipient_refs[]
  channel_ref

  proposition_id

  made_at_logical_tick
  source_event_ref
```

A CommunicationClaim never automatically becomes a Fact or KnowledgeRecord.

---

# 29. EvidenceRelation

```
EvidenceRelation
  evidence_ref
  proposition_id

  relation:
    SUPPORTS
    CONTRADICTS

  optional strength: CanonicalNumber
```

No automatic Bayesian/logical inference is implied.

Scenario rules decide what evidence does to knowledge/belief/suspicion.

---

# 30. ScheduledEffect

```
ScheduledEffect
  schedule_id
  status

  source_ref
  created_at_logical_tick

  due:
    logical_tick
    | deterministic_condition_ref

  event_template_ref
  effect_parameters

  optional cancellation_rule_ref

  version_manifest_ref
```

## Status

```
PENDING
FIRED
CANCELLED
```

## Invariants

- Fire/cancel is canonical.
- Due evaluation is deterministic.
- A fired schedule cannot fire again.
- Interaction with open ResolutionWindow follows the accepted Single Canonical Frontier policy.

---

# 31. DecisionPointDefinition

Compiled scenario contract.

```
DecisionPointDefinition
  decision_point_id

  activation_rule_ref
  deactivation_rule_ref

  action_slot_templates[]
  resolution_window_template

  optional presentation_ref
```

DecisionPoint activation is canonical Scenario Progression.

---

# 32. ProgressionTransition

```
ProgressionTransition
  transition_key
  trigger_ref

  from_progression_state
  to_progression_state

  generated_event_templates[]
  generated_window_templates[]
  optional ending_ref

  priority
  optional random_policy_ref
```

## Invariants

- Compiled/pure.
- Multiple candidates must deterministically compose or select.
- Narrative Direction cannot create ProgressionTransitions.

---

# 33. PresentationPlan

Player-safe structured artifact.

```
PresentationPlan
  presentation_plan_id
  session_id
  audience_ref

  state_revision
  related_event_refs[]
  related_progression_refs[]

  content_blocks[]
  available_ui_actions[]
  optional localization_context

  plan_version
```

## Critical invariant

Everything in the PresentationPlan must already be authorized for that audience before Narrative Realization receives it.

The Realizer never receives full hidden SessionState "for convenience."

---

# 34. PresentationRecord

```
PresentationRecord
  presentation_id
  session_id
  audience_ref

  state_revision
  presentation_plan_hash

  realizer_version
  template_version
  optional provider
  optional model
  optional prompt_version

  exact_delivered_output
  output_hash

  generation_status
  delivery_status

  generated_at   // operational
  optional delivered_at
```

PresentationRecord is immutable after a delivered output is finalized, except for append-only delivery status/audit mechanics defined later.

---

# 35. TransactionalOutboxItem

```
TransactionalOutboxItem
  outbox_id
  session_id

  task_type
  deduplication_key
  payload

  status
  attempt_count

  created_at
  optional available_after
  optional completed_at
```

## Initial TaskType examples

```
BUILD_PRESENTATION
REALTIME_STATE_ADVANCED
UPDATE_SECONDARY_PROJECTION
EMIT_ANALYTICS
TRIGGER_MEDIA_JOB
```

## Invariants

- Created in same DB transaction as the canonical commit that requires it.
- Consumer operation is idempotent by deduplication key.
- Failure/retry cannot alter canonical world state outside another EngineCommand/canonical transition.

---

# 36. SessionState projection

Logical shape, not storage format:

```
SessionState
  session_id
  stream_revision
  state_hash

  version_manifest

  lifecycle
  logical_time

  entities
  relationships
  epistemics

  goals
  commitments

  scheduler
  progression

  optional active_resolution_window_ref

  scenario_metrics
```

## Invariants

- Fully rebuildable from pinned initial state + canonical event stream.
- Never treated as historical authority over conflicting events.
- No presentation prose inside canonical SessionState.

---

# 37. Session lifecycle proposal

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
CREATED
  -> WAITING_FOR_PARTICIPANTS
  -> READY
  -> ACTIVE

ACTIVE
  -> PAUSED
  -> COMPLETED
  -> ABANDONED

PAUSED
  -> ACTIVE
  -> ABANDONED
```

Exact commercial/account lifecycle remains separate.

---

# 38. Cross-contract invariants

These are implementation-blocking invariants.

## Authority

1. No browser request directly appends Domain Events.
2. No browser request directly supplies Claim or Mutation IR.
3. Principal is never trusted from free client payload.

## Command/idempotency

4. Same CommandId cannot produce canonical effects twice.
5. Same selected SubmissionId cannot become two action instances in one Resolution.

## Window/frontier

6. Canonical gameplay revision stays fixed while an OPEN simultaneous window collects submissions.
7. FROZEN window input set is immutable.
8. One canonical frontier operation advances a Session at a time.

## Resolution

9. Every selected submission has one AdmissionResult.
10. Every ADMITTED action has exactly one ActionOutcome.
11. Interaction grouping is deterministic.
12. Aggregate resource constraints are checked across the relevant group.
13. Independent groups commute at final-state level.

## Events/state

14. Every canonical state change is attributable to canonical event content.
15. Event batch + projection update + ResolutionRecord + required outbox writes commit atomically.
16. Operational metadata does not affect semantic event-batch hash.
17. Current SessionState hash matches canonical replay.

## Time/random/numbers

18. Wall clock does not directly drive historical mechanics.
19. Logical time is deterministic integer/fixed representation.
20. Canonical numeric values never depend on platform float tolerance.
21. Every canonical random decision has named deterministic evidence.

## Epistemics

22. Knowledge cannot appear without explicit deterministic provenance.
23. CommunicationClaim does not imply Fact.
24. Unknown is not fabricated as a positive belief state.
25. Historical fact validity is not silently erased.

## Presentation

26. PresentationPlan is audience-safe before realization.
27. Realizer failure cannot roll back or corrupt canonical gameplay state.
28. Delivered presentation that can influence player behavior is recoverable from PresentationRecord.

---

# 39. Deliberately open contract details

The following are not yet frozen by this document:

- Opaque ID encoding (UUID/ULID/deterministic composite, etc.);
- exact stream-revision convention;
- exact Claim.scope syntax;
- exact Mutation target encoding;
- exact fixed-point scales/units per scenario metric;
- exact PRNG;
- exact event-family registry;
- exact resolver strategy parameter schemas;
- exact Relationship/Goal/Commitment contracts;
- exact Scenario source DSL;
- SQL representation/indexes;
- snapshot strategy;
- retention policy for noncanonical audit artifacts.

---

# 40. Required red-team before v0.2 contracts

Before these contracts are accepted, test them against at least the following synthetic cases:

1. both players consume the same scarce battery;
2. one sabotages a radio while the other transmits;
3. two independent actions resolve in any input order;
4. player submits a hidden EntityId manually;
5. player replaces an action immediately before deadline;
6. deadline command is delivered twice;
7. scheduled logical effect becomes due while window is open;
8. three actions collectively exceed capacity though each pair fits;
9. one action depends on another in the same simultaneous window;
10. one player lies; recipient believes it; canon remains false;
11. later evidence changes belief without changing historical communication claim;
12. fact validity ends while historical observation remains;
13. Realizer crashes after canonical commit;
14. outbox consumer retries twice;
15. unrelated RNG draw is added to a rule;
16. presentation wording changes but world replay hash remains unchanged;
17. engine build upgrades while an old Session must be replayed;
18. ending trigger and another progression trigger become true together.

---

# 41. Next artifact after contract acceptance

After red-team and acceptance of Domain Contracts:

1. `DOMAIN_MODEL_v0.1.md`
2. concrete persistence model / SQL schema proposal;
3. repository package dependency architecture;
4. resolver/test harness skeleton.

No production implementation should precede contract review.
