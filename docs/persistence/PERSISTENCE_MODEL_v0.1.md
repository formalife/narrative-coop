# PERSISTENCE DATA MODEL v0.1

**Status:** PROPOSED  
**Date:** 2026-10-02  
**Governing architecture:** Architecture Baseline v0.3 — ACCEPTED  
**Governing contracts:** Domain Contracts v0.3 — ACCEPTED  
**Governing domain model:** Domain Model v0.2 — ACCEPTED  
**Database baseline:** PostgreSQL per ADR-003  
**Purpose:** define persistence ownership, relational boundaries, transaction boundaries and representation strategy before concrete SQL migrations.

---

# 1. Persistence principles

## 1.1 PostgreSQL is one persistence system, not one giant aggregate

Use one PostgreSQL database initially, but separate logical persistence families by ownership/lifecycle.

Do not create microservice/datastore boundaries from domain bounded contexts unless measured requirements justify them.

## 1.2 Normalize for invariants; JSON/bytes for scenario-defined structures

Prefer relational columns/constraints for:

- Session identity;
- revision/concurrency anchor;
- immutable scenario/session hashes;
- command idempotency;
- pending-input CAS;
- stream ordering;
- transition ownership;
- outbox leasing/status;
- uniqueness/lifecycle states that must survive process failure.

Prefer JSONB or canonical serialized bytes for:

- current canonical state content;
- scenario-defined component payloads;
- typed but extensible event payload/mutation structures;
- presentation payloads;
- resolution/debug evidence where relational querying is not a core invariant.

## 1.3 No persistence representation becomes a second domain truth

Examples:

- current Session projection is rebuildable from Genesis + events;
- transition records do not override event content;
- derived indexes do not override canonical state;
- event query columns do not override canonical event bytes;
- cached PlayerInteractionViews do not override current permissions/canonical state.

## 1.4 Append-only/immutable records remain immutable

Application code must never update/delete committed:

- scenario bundle bytes;
- SessionGenesis;
- canonical event content;
- canonical transition identity/hash evidence;
- delivered PresentationRecord payload;
- frozen selected-input evidence.

Administrative break-glass repair is outside ordinary domain writes and requires audit/runbook policy.

---

# 2. Persistence families

Proposed logical tables/relations:

## Scenario/version

- `scenario_versions`

## Session bootstrap/current canon

- `sessions`
- `session_genesis`
- `session_runtime`
- `session_events`
- `canonical_transitions`

## Commands/pending input

- `command_processing`
- `action_submissions`
- `window_input_gates`
- `window_slot_inputs`
- `frozen_input_sets`
- `resolution_attempts`

## Transition/resolution evidence

- `resolution_records`
- optional `resolution_debug_artifacts`

## Player interaction/presentation

- `player_interaction_views`
- `presentation_plans`
- `presentation_records`

## Reliability/operations

- `outbox`

Potential future/optional:
- presence store;
- secondary recap/admin projections;
- raw natural-language/parser audit artifacts;
- snapshots.

These names are logical proposals, not final SQL identifiers.

---

# 3. scenario_versions

Immutable published runtime bundle storage.

Logical fields:

```
scenario_id
scenario_version

bundle_hash
bundle_schema_version

compiled_bundle_canonical_bytes

status:
  PUBLISHED
  RETIRED

published_at
optional retired_at

optional metadata_json
```

## Keys/invariants

- unique `(scenario_id, scenario_version)`;
- unique `bundle_hash`;
- published canonical bytes never mutate;
- `bundle_hash` is computed from canonical serialized bundle bytes;
- RETIRED affects new Session creation only; existing pinned Sessions remain valid.

## Representation

Authoritative compiled bundle should preserve exact canonical serialized bytes (e.g. BYTEA) rather than relying on database JSON reserialization for content-addressed identity.

Small searchable metadata may be relational/JSONB but is not bundle truth.

---

# 4. sessions

Stable Session identity/metadata.

Logical fields:

```
session_id

scenario_id
scenario_version
scenario_bundle_hash

created_at
optional terminal_at
```

This is not the mutable canonical state row.

## Invariants

- references one immutable scenario version/hash;
- Session identity never changes scenario bundle;
- no gameplay revision/state content here.

Account/participant ownership fields remain out of scope until ADR-015 is accepted.

---

# 5. session_genesis

Exactly one immutable bootstrap row per Session.

Logical fields:

```
session_id

mechanics_manifest_canonical_bytes
mechanics_manifest_hash

scenario_bundle_hash

session_seed

initial_canonical_state_hash

created_at
```

## Keys/invariants

- primary/unique `session_id`;
- scenario bundle hash matches Session/scenario version;
- immutable after creation;
- seed cannot be null/rotated;
- initial state hash is revision-0 state hash;
- `created_at` excluded from mechanics hash.

## Representation

Store exact manifest canonical bytes plus hash.

Do not rely on reconstructing old MechanicsVersionManifest from mutable deployment configuration.

---

# 6. session_runtime

One mutable synchronous current-state row per Session.

This is both:

- current SessionStateProjection storage;
- concurrency/Single Canonical Frontier anchor.

Logical fields:

```
session_id

current_revision
state_hash

lifecycle_state
logical_tick

canonical_state_content_jsonb

mechanics_manifest_hash

updated_at
```

## Why one row

Avoid splitting current revision/hash/state across multiple authoritative mutable rows.

A canonical transition locks/compares this row and replaces current projection atomically with event append.

## Invariants

- `current_revision` equals last canonical event revision;
- `state_hash` matches canonical serialization of revision + manifest hash + CanonicalStateContent;
- lifecycle/logical_tick columns are extracted convenience/query fields and MUST equal the JSONB content;
- extracted convenience columns never override JSONB semantic state;
- no partial JSONB patch from browser/admin domain API;
- normal canonical update writes the complete validated after-state projection.

## Trade-off

Whole-state JSONB updates can create write amplification/bloat.

Accepted for MVP-scale short two-player Sessions; revisit only with measurement.

---

# 7. session_events

Append-only canonical Session event stream.

Logical fields:

```
session_id
stream_revision

transition_key
batch_index

event_family
event_code
event_schema_version
logical_tick

canonical_event_bytes
canonical_event_hash

recorded_at
```

## Primary/order keys

- primary/unique `(session_id, stream_revision)`;
- unique `(session_id, transition_key, batch_index)`.

## Authority

`canonical_event_bytes` are the authoritative immutable CanonicalEventContent representation.

Columns such as family/code/logical_tick are extracted query/index metadata and must match canonical bytes at insertion time.

They are not an alternate event payload.

## Why bytes instead of only JSONB

The accepted replay/version policy requires immutable historical canonical representation.

JSONB is useful for querying logical JSON values but does not preserve original serialized bytes.

Canonical bytes + content hash preserve exactly what was hashed/committed.

## Query policy

Do not design product features that query arbitrary event payload JSON directly as the main read path.

Use current/secondary projections for product views.

---

# 8. canonical_transitions

One immutable record per committed non-empty canonical event batch.

Logical fields:

```
session_id
transition_key

transition_kind

base_revision
resulting_revision
event_count

optional trigger_command_key
optional resolution_id

before_state_hash
event_batch_hash
after_state_hash

mechanics_manifest_hash

committed_at
```

## Keys/invariants

- unique `(session_id, transition_key)`;
- base/resulting revisions match owned event range;
- `resulting_revision = base_revision + event_count`;
- event_count > 0;
- before hash equals previous Session state hash;
- after hash equals committed session_runtime state hash;
- immutable after commit.

Transition record is causal/hash evidence.

World truth remains event content.

---

# 9. command_processing

Durable idempotency and resumable application-command state.

Logical fields:

```
session_id
command_key

command_type
semantic_payload_hash
semantic_payload_jsonb

principal_type
principal_ref_or_json

processing_status

optional frozen_input_set_hash
optional transition_key

optional command_result_jsonb

first_received_at
last_updated_at
```

## Key

Primary/unique:
`(session_id, command_key)`

## Idempotency

- same key + same semantic hash returns/continues same operation;
- same key + different semantic hash rejects;
- processing state survives process crash;
- mutable until terminal command status;
- not canonical world history.

## Raw natural-language input

Do not store large/raw free-text automatically inside semantic command payload.

Parsed structured semantics belong here; raw input, if retained, belongs to a separate optional audit artifact with privacy/retention controls.

---

# 10. action_submissions

Immutable pending input records.

Logical fields:

```
submission_id

session_id
window_id
action_slot_id

action_type_id

parameters_jsonb
selected_target_refs_jsonb

source_type
optional source_player_view_key
optional source_presentation_id

created_by_command_key

received_at
```

## Invariants

- submission immutable;
- replacement creates a new row;
- does not reserve/mutate world resources;
- access governed by pending visibility policy;
- browser can submit only through authoritative command/API path.

## Retention

Selected/frozen submissions required by supported resolver verification policy.

Replaced/nonselected submissions may follow shorter privacy retention if not required for forensic policy.

---

# 11. window_input_gates

One durable gate per canonical ResolutionWindow instance.

Logical fields:

```
session_id
window_id

base_revision

gate_status:
  ACCEPTING
  FROZEN
  CANCELLED

optional frozen_input_set_hash

created_at
updated_at
optional frozen_at
```

## Invariants

- gate created after/with observed canonical WindowOpen;
- base_revision matches canonical Window instance;
- once FROZEN/CANCELLED, never returns to ACCEPTING;
- one currently active accepting/frozen action gate per Session under accepted architecture.

A partial unique constraint/index is a candidate mechanism for enforcing one active gate per Session.

---

# 12. window_slot_inputs

CAS state for each ActionSlot.

Logical fields:

```
session_id
window_id
action_slot_id

slot_input_revision

optional current_submission_id
finalized

updated_at
```

## Key

`(session_id, window_id, action_slot_id)`

## Concurrency protocol

Replacement/submit transaction:

1. lock/check parent gate is ACCEPTING;
2. verify expected SlotInputRevision;
3. insert immutable ActionSubmission;
4. update slot current_submission_id;
5. increment SlotInputRevision;
6. update any derived interaction-view invalidation data if used;
7. commit.

Lock ordering must be consistent:
- gate first;
- target slot second.

This reduces deadlock risk against freeze operations.

---

# 13. frozen_input_sets

Immutable selected-input snapshot after gate freeze.

Logical fields:

```
session_id
window_id

base_revision

input_set_hash

selected_submissions_jsonb
slot_input_revisions_jsonb

created_at
```

## Invariants

- at most one FrozenInputSet per Window;
- immutable;
- hash calculated from canonicalized selected slot/submission/revision mapping;
- ResolutionRecord references this hash/record.

## Freeze transaction

The input freeze is a noncanonical application transaction:

1. lock gate;
2. verify ACCEPTING;
3. lock all slot rows in deterministic slot-id order;
4. select final submissions/defaults according to policy;
5. create FrozenInputSet;
6. set gate FROZEN;
7. attach hash to CommandProcessingRecord;
8. commit.

No canonical StreamRevision advances.

---

# 14. resolution_attempts

Operational resolver-attempt telemetry/resume evidence.

Logical fields:

```
attempt_id

session_id
window_id
frozen_input_set_hash

status
optional failure_code

started_at
optional finished_at
```

May be pruned under operational retention.

Not required for state replay.

---

# 15. resolution_records

Immutable resolution evidence.

Logical fields:

```
resolution_id

session_id
window_id
frozen_input_set_hash

selected_submissions_jsonb
admission_results_jsonb
action_outcomes_jsonb
named_random_draws_jsonb

interaction_graph_hash
action_resolution_hash

mechanics_manifest_hash

created_at
```

## Invariants

- immutable once referenced by committed CanonicalTransitionRecord;
- random draw evidence required by verification policy retained;
- no world state is derived from this instead of canonical events.

Large optional debug graph/expanded-action detail can be separated to a debug artifact table/blob so core evidence stays bounded.

---

# 16. resolution_debug_artifacts — optional

Potential fields:

```
resolution_id
artifact_type
content_jsonb_or_bytes
content_hash
created_at
retention_class
```

Examples:

- ExpandedActions;
- full InteractionGraph;
- transient plan diagnostics.

Not required for basic state replay.

Can use shorter retention if forensic policy permits.

---

# 17. player_interaction_views

Immutable materialized view content when persistence is required.

Logical fields:

```
player_view_key

session_id
participant_ref
state_revision

optional window_id
optional window_input_gate_hash
optional own_slot_input_revision

view_schema_version

view_content_jsonb
created_at
```

## Materialization policy

Do NOT persist every computed view automatically.

Persist a PlayerInteractionView when:

- an async Presentation generation task references it;
- a delivered PresentationRecord must preserve the exact source view;
- debugging/playtest policy explicitly requests it.

Otherwise it may be computed on demand.

## Why persistence is needed for async presentation

A PlayerInteractionView can change while canonical StreamRevision stays constant because pending input changes.

An outbox worker cannot reliably reconstruct an old view later from only current Session state.

Therefore any async task keyed to a specific PlayerViewKey must have immutable source view content available.

---

# 18. presentation_plans

Immutable structured plans produced from one PlayerInteractionView.

Logical fields:

```
presentation_plan_id

session_id
participant_ref

state_revision
player_view_key

plan_version
presentation_policy_version

plan_content_jsonb
plan_hash

created_at
```

Narrative Direction may create multiple plans experimentally only if delivery policy clearly identifies which one is actually presented.

A plan is not world state.

---

# 19. presentation_records

Player-facing delivered/generation evidence.

Logical fields:

```
presentation_id

session_id
participant_ref

state_revision
player_view_key
presentation_plan_hash

generation_context_jsonb
generation_context_hash

exact_delivered_payload
output_hash

generation_status
delivery_status

generated_at
optional delivered_at
```

## Invariants

- delivered payload becomes immutable interaction evidence;
- retry generation uses same pinned GenerationContext;
- DeliveryGuard compares target PlayerViewKey/current applicability before current gameplay delivery;
- SUPERSEDED output may be retained for debug but is not presented as current.

Large media bytes live outside PostgreSQL; record immutable asset references/hashes.

---

# 20. outbox

Transactional post-commit reliability queue.

Logical fields:

```
outbox_id

session_id
optional transition_key

task_type

deduplication_key
payload_jsonb
execution_context_jsonb

status:
  PENDING
  PROCESSING
  COMPLETED
  FAILED_RETRYABLE
  FAILED_TERMINAL

attempt_count

available_after
created_at
optional locked_at
optional worker_ref
optional completed_at
```

## Invariants

- unique deduplication key within appropriate task scope;
- items required by canonical transition inserted in same transaction;
- consumers idempotent;
- retry preserves execution context;
- outbox processing cannot directly mutate canonical Session state except by issuing a new EngineCommand.

## Worker candidate

PostgreSQL row locking with a short lease/transaction is sufficient initially.

A worker may select ready rows using row-level locking and skip rows already claimed by other workers.

No broker is required by this model.

---

# 21. Session creation transaction

Proposed logical transaction:

1. verify immutable scenario version exists;
2. create `sessions`;
3. create immutable `session_genesis` with Session seed and MechanicsVersionManifest;
4. deterministically construct revision-0 CanonicalStateContent;
5. verify `initial_canonical_state_hash`;
6. create `session_runtime` at revision 0;
7. commit.

No Domain Event exists at revision 0.

Any random initialization or dynamic participant binding occurs through subsequent canonical transitions.

---

# 22. Canonical transition commit transaction

Precondition:
TransitionPlan built/validated outside DB transaction.

Transaction:

1. lock/read `session_runtime`;
2. verify `current_revision == base_revision`;
3. verify current state hash == TransitionPlan.before_state_hash;
4. insert optional immutable `resolution_records`;
5. insert `canonical_transitions`;
6. append `session_events` in batch-index order;
7. replace `session_runtime` projection with validated after-state;
8. finalize/cancel relevant WindowInputGate if transition closes the Window;
9. update CommandProcessingRecord to terminal/applied result if command-owned;
10. insert required Outbox items;
11. commit.

All canonical events/state/transition/outbox work either commits together or not at all.

---

# 23. Crash after freeze before canonical commit

Durable state after freeze:

- Session remains at original canonical revision;
- Window remains canonically OPEN;
- WindowInputGate is FROZEN;
- FrozenInputSet exists;
- CommandProcessingRecord references frozen hash;
- no canonical transition/event has committed.

Retry:

1. load same CommandKey;
2. load FrozenInputSet;
3. resume/recompute deterministic resolution under same base state;
4. attempt canonical commit.

If Session revision no longer equals base revision due to exceptional administrative/recovery activity, abort normal commit and require explicit recovery policy.

---

# 24. No-op command

A semantic NO_OP command:

- updates CommandProcessingRecord/CommandResult;
- creates no CanonicalTransitionRecord;
- creates no Domain Event;
- does not change session_runtime revision/hash;
- may produce a noncanonical response/presentation only if explicitly needed.

---

# 25. Referential-integrity policy

Use database foreign keys for stable relational ownership where practical:

Examples:
- Session -> scenario version;
- Genesis -> Session;
- events/transitions/submissions/gates -> Session;
- slot inputs -> gate/window identity;
- ResolutionRecord -> Session;
- PresentationRecord -> Session.

Do not attempt relational foreign keys into arbitrary canonical event/payload references stored inside canonical bytes/JSON.

Those are validated by compiler/domain/replay validators.

---

# 26. JSONB policy

JSONB is appropriate for:

- current CanonicalStateContent projection;
- command structured payload/result;
- ActionSubmission parameters;
- selected-input maps;
- resolution evidence;
- presentation/view structures;
- outbox payload/context.

Rules:

1. every JSONB domain object has an application/schema version where replay/persistence requires it;
2. database JSONB shape alone is not validation;
3. validate against typed/schema contracts before write;
4. do not create blanket GIN indexes on every JSONB column;
5. add payload indexes only for demonstrated query patterns;
6. canonical hashes are computed from the declared application canonical serialization, not PostgreSQL JSONB textual output.

---

# 27. Canonical bytes policy

Use exact canonical serialized bytes for content whose identity/hash/version contract requires preserving the committed representation.

At minimum candidates:

- CompiledScenarioBundle canonical bytes;
- MechanicsVersionManifest canonical bytes;
- CanonicalEventContent canonical bytes.

Potentially also persist canonical bytes for:
- FrozenInputSet;
- PresentationPlan;
- PlayerInteractionView;

only if exact-byte forensic requirements justify it.

Otherwise JSONB + declared hash is sufficient for noncanonical/derived artifacts.

---

# 28. Database access/security zones

## Internal-authoritative

Direct browser writes forbidden:

- session_genesis;
- session_runtime;
- session_events;
- canonical_transitions;
- command_processing;
- gates/slot inputs;
- frozen sets;
- resolution evidence;
- outbox.

## Player-facing

Browser receives only API/RPC projections such as PlayerInteractionView/presentation data authorized for its Participant.

Do not expose canonical world-state tables as generic browser-readable resources.

## Analytics

Analytics receives deliberate sanitized events/properties, not raw canonical/private tables.

Concrete RLS/API policies depend on the later identity decision (ADR-015) and implementation stack.

---

# 29. Index strategy — initial candidates, not frozen DDL

## Essential uniqueness/order

- `session_events(session_id, stream_revision)` unique/primary;
- `session_events(session_id, transition_key, batch_index)` unique;
- `canonical_transitions(session_id, transition_key)` unique;
- `command_processing(session_id, command_key)` unique;
- `window_slot_inputs(session_id, window_id, action_slot_id)` unique;
- one FrozenInputSet per Window;
- `outbox(deduplication_key)` unique within chosen scope;
- `player_interaction_views(player_view_key)` unique;
- scenario version/hash uniqueness.

## Likely operational indexes

- ActionSubmissions by Session/Window/Slot;
- Outbox by pending/retry status + `available_after`;
- Presentations by Session/Participant/delivery time;
- Transitions by ResolutionId where present.

## Partial uniqueness candidate

Enforce at most one nonterminal action-input gate per Session with a partial unique index over active gate statuses.

Exact predicate/status implementation belongs to SQL schema review.

---

# 30. Partitioning policy

Do NOT partition tables initially.

Potential future partition candidates:

- session_events;
- command/audit history;
- presentations/outbox archive.

Revisit only after measured table size/maintenance/query requirements.

Partitioning changes operational complexity and can constrain uniqueness/index design.

---

# 31. Snapshot policy

No explicit historical snapshot table in v0.1 persistence model.

Current `session_runtime` is a synchronous current projection, not a historical snapshot.

Future replay snapshots may be introduced only as optimization if replay time/retention measurements justify them.

A snapshot never becomes more authoritative than Genesis + event stream.

---

# 32. Deletion/retention classes

## Mechanical minimum

Retain under historical compatibility policy:

- scenario canonical bundle;
- SessionGenesis;
- canonical Session events.

## Verification evidence

Retain according to resolver/replay support policy:

- FrozenInputSet selected inputs;
- ResolutionRecord;
- CanonicalTransitionRecord;
- NamedRandomDraw evidence.

## Player experience

Delivered PresentationRecord retention subject to privacy/product policy.

## Optional audit

May use shorter retention:

- replaced/nonselected submissions;
- raw natural-language artifacts;
- debug resolution artifacts;
- worker logs/attempts.

## Operational

Completed Outbox rows may be compacted/archive/deleted after dedup/reliability window, provided historical transition correctness does not depend on them.

---

# 33. Persistence invariants

1. One immutable Genesis per Session.
2. One mutable session_runtime row per Session.
3. Session current revision is advanced only inside canonical commit.
4. Event stream revisions are contiguous under accepted convention.
5. Event rows never mutate after commit.
6. Transition record event range/hash agrees with event rows.
7. Current projection hash agrees with committed transition after-state hash.
8. Gate freeze never advances StreamRevision.
9. Slot replacement uses CAS revision and cannot pass a frozen gate.
10. FrozenInputSet immutable.
11. Same CommandKey cannot apply twice.
12. Same CommandKey with changed semantic payload rejected.
13. Required outbox work is atomic with canonical transition.
14. Delivered presentation references immutable source PlayerView/plan/generation context.
15. Derived indexes/caches can be deleted without breaking state reconstruction.
16. Browser cannot write canonical/internal tables directly.
17. No table outside event stream/genesis can silently become canonical historical authority.

---

# 34. Persistence red-team cases

Before acceptance, test:

1. two canonical transition workers race for same Session;
2. freeze transaction races with ReplaceAction;
3. deadline and explicit-finalize commands race;
4. crash after Genesis insert but before runtime row creation;
5. crash after event append but before state projection/outbox update;
6. duplicate event batch insert retry;
7. session_runtime JSONB row becomes large;
8. current-state convenience columns drift from JSONB;
9. canonical event bytes disagree with extracted metadata columns;
10. PlayerInteractionView async task runs after pending input changed;
11. outbox worker crashes after external side effect but before marking completed;
12. same deduplication key reused with changed outbox payload;
13. scenario bundle is RETIRED while old Session replays;
14. optional nonselected submissions deleted before forensic investigation;
15. event table grows large without partitions;
16. admin attempts to patch current state directly;
17. database restore recovers event stream but loses derived indexes;
18. command row remains PROCESSING indefinitely after worker death;
19. Session becomes terminal while gate remains FROZEN;
20. PresentationRecord/media points to mutable external asset.

---

# 35. Next gate

Perform persistence-model red-team/postmortem.

If v0.1 survives:
- accept persistence model;
- design concrete PostgreSQL schema/DDL proposal.

If material flaws are found:
- produce PERSISTENCE_MODEL_v0.2 before any migration files.

No production SQL migration should be created from v0.1 without this review.
