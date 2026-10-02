# PERSISTENCE DATA MODEL v0.3

**Status:** PROPOSED  
**Date:** 2026-10-02  
**Supersedes as working proposal:** PERSISTENCE_MODEL_v0.2  
**Governing architecture:** Architecture Baseline v0.3 — ACCEPTED  
**Governing contracts:** Domain Contracts v0.3 — ACCEPTED  
**Governing domain model:** Domain Model v0.2 — ACCEPTED  
**Database baseline:** PostgreSQL per ADR-003

---

# 1. Persistence objective

Use one PostgreSQL system to preserve:

- deterministic Session history;
- synchronous current-state projection;
- command/input idempotency;
- pending-input concurrency;
- resolver/replay evidence;
- reliable post-commit work;

without turning persistence representation into a second domain model.

No SQL migration is authorized until this persistence model is accepted.

---

# 2. Core representation rule

Normalize when the database must enforce:

- identity;
- uniqueness;
- ordering;
- CAS/revision;
- lifecycle/lease;
- foreign ownership;
- idempotency.

Use JSONB for typed extensible structures whose schema is enforced by application/compiler contracts.

Use exact canonical serialized bytes for content-addressed/replay material whose exact committed representation must be retained.

---

# 3. Logical persistence families

## Immutable definitions/bootstrap

- `scenario_versions`
- `sessions`
- `session_genesis`

## Canonical history/current state

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

## Transition evidence

- `resolution_records`
- optional `resolution_debug_artifacts`

## Interaction/presentation

- `player_interaction_views`
- `presentation_plans`
- `presentation_records`

## Reliability

- `outbox`

Optional secondary read models, presence and analytics remain outside canonical state.

Logical relation != mandatory dedicated physical table if a later schema can preserve identical invariants more simply.

---

# 4. Global transactional lock order

When one transaction locks multiple concurrency-bearing rows, acquire locks only in this relative order:

```
1. session_runtime
2. command_processing
3. window_input_gates
4. window_slot_inputs in canonical action_slot_id order
```

A transaction may use a subset but MUST NOT reverse the relative order.

Examples:

### Submit/Replace Action

```
command_processing
-> gate
-> target slot
```

### Freeze/Close Window

```
session_runtime
-> command_processing
-> gate
-> all slots in canonical slot order
```

### Canonical Window resolution commit

```
session_runtime
-> command_processing
-> gate
```

Purpose:

- avoid deadlock cycles;
- serialize freeze against canonical closure;
- ensure frozen input is attached to the correct canonical base revision/window.

Do not hold these locks during LLM/network/media/external-provider calls.

---

# 5. scenario_versions

Immutable published compiled Scenario Bundle.

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

optional searchable_metadata_jsonb
```

Invariants:

- unique `(scenario_id, scenario_version)`;
- unique `bundle_hash`;
- canonical bytes immutable after publish;
- RETIRED prevents new use but never invalidates old Session replay;
- searchable metadata is not bundle truth.

---

# 6. sessions

Stable Session identity.

```
session_id

scenario_id
scenario_version
scenario_bundle_hash

created_at
optional terminal_at
```

No mutable gameplay state lives here.

Identity/account ownership fields remain open pending ADR-015.

---

# 7. session_genesis

Exactly one immutable bootstrap record per Session.

```
session_id

mechanics_manifest_canonical_bytes
mechanics_manifest_hash

scenario_bundle_hash
session_seed

initial_canonical_state_hash

created_at
```

Invariants:

- one row per Session;
- immutable;
- seed backend-private and durable;
- scenario bundle hash matches Session and manifest;
- revision-0 state hash reproducible from bundle + Genesis;
- created_at operational only.

No hidden random setup.

Random initial setup occurs through revision-1+ canonical transitions.

---

# 8. session_runtime

One mutable row per Session.

This row is:

- current SessionStateProjection storage;
- Single Canonical Frontier lock/CAS anchor.

Logical fields:

```
session_id

current_revision
state_hash
mechanics_manifest_hash

canonical_state_content_jsonb

updated_at
```

## Deliberately excluded

Do not initially duplicate canonical JSON semantics into mutable convenience columns such as:

- lifecycle_state;
- logical_tick;
- ending_ref;
- active_window_id.

If later query performance requires these, add derived/generated/secondary projection fields that are explicitly non-authoritative and rebuildable.

## Invariants

- current_revision == latest committed Session event revision;
- state_hash matches manifest hash + revision + canonical state content;
- normal canonical commit replaces complete validated after-state;
- browser/admin domain endpoints do not partially patch JSONB;
- row is locked or compare-and-set for every canonical transition.

Whole-row JSONB update cost is an accepted MVP trade-off subject to measurement.

---

# 9. session_events

Append-only canonical event stream.

```
session_id
stream_revision

transition_key
batch_index

event_family_index
event_code_index
event_schema_version_index
logical_tick_index

canonical_event_bytes
canonical_event_hash

recorded_at
```

## Keys

- primary/unique `(session_id, stream_revision)`;
- unique `(session_id, transition_key, batch_index)`.

## Authority

Only:

- `canonical_event_bytes`;
- their cryptographic/content hash;

define canonical semantic event content.

Columns suffixed `_index` are query acceleration metadata.

They are non-authoritative.

## Insert validation

Writer derives both canonical bytes and index metadata from the same validated in-memory CanonicalEventContent.

## Repair validation

Maintenance/replay tooling can:

1. parse canonical bytes;
2. verify canonical_event_hash;
3. recompute index metadata;
4. detect/repair non-authoritative index-column drift.

Canonical bytes are never reconstructed from index columns.

---

# 10. canonical_transitions

One immutable row per non-empty canonical batch.

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

Invariants:

- unique `(session_id, transition_key)`;
- event_count > 0;
- resulting_revision = base_revision + event_count;
- owned event rows exactly occupy the resulting contiguous revision interval;
- after_state_hash equals committed session_runtime hash;
- immutable.

No-op commands create no transition row.

---

# 10A. Event and batch hash formulas

For persistence verification:

```
canonical_event_hash =
  H(canonical_event_bytes)
```

```
event_batch_hash =
  H(
    canonical(
      OrderedList<CanonicalEventContent>
      sorted by batch_index
    )
  )
```

Do not compute a batch hash by raw concatenation of variable-length hashes/byte strings without canonical framing.

The hash algorithm and canonicalization version come from the pinned MechanicsVersionManifest.

CanonicalTransitionRecord.event_batch_hash must equal the recomputed ordered batch hash.

---

# 11. command_processing

Durable command idempotency/resume record.

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

processing_owner
optional processing_lease_until
processing_generation
processing_attempt_count

first_received_at
last_updated_at
```

## Semantic idempotency

Primary/unique:
`(session_id, command_key)`

- same key + same semantic hash -> resume/return same logical result;
- same key + changed semantic hash -> reject;
- duplicate delivery does not create a new logical operation.

## Lease

PROCESSING may be reclaimed after `processing_lease_until` expires.

Every claim/takeover increments `processing_generation`.

Takeover:

- keeps same CommandKey;
- keeps same FrozenInputSet if already frozen;
- increments processing generation and attempt count;
- cannot create a second canonical transition.

Any lease renewal, command finalization, or canonical commit owned by the command MUST compare the worker's expected `processing_generation`.

A stale worker with an older generation cannot update command result/status or attach a canonical transition.

Lease timing/generation are operational fencing data and cannot affect gameplay result semantics.

---

# 12. action_submissions

Immutable pending structured input.

```
submission_id

session_id
window_id
action_slot_id

action_type_id

parameters_jsonb
selected_target_refs_jsonb

submission_content_hash

source_type
optional source_player_view_key
optional source_presentation_id

created_by_command_key

received_at
```

Rules:

- `submission_content_hash` is computed from canonicalized mechanical submission content: Session/Window/Slot identity, ActionType, parameters and selected targets; it excludes operational timestamps and optional presentation provenance unless explicitly mechanical;
- replacement inserts another row;
- world resources not reserved merely by insertion;
- access follows pending visibility policy;
- selected submission content is retained for resolver verification support window;
- raw natural-language source text is separate optional audit data.

---

# 13. window_input_gates

One pending-input gate per canonical ResolutionWindow instance.

```
session_id
window_id

base_revision

gate_status:
  ACCEPTING
  FROZEN
  CANCELLED
  CLOSED

optional frozen_input_set_hash
optional freeze_owner_command_key
optional planned_transition_key

created_at
updated_at
optional frozen_at
optional closed_at
```

Invariants:

- cannot transition back to ACCEPTING;
- base revision/window identity verified against session_runtime at freeze;
- at most one nonterminal action-input gate per Session;
- FROZEN gate has a freeze owner;
- normal canonical transition cannot advance behind a FROZEN gate except explicit recovery/interrupt semantics.

A partial unique index is a candidate DDL mechanism for the one-active-gate invariant.

---

# 14. window_slot_inputs

Pending CAS state per ActionSlot.

```
session_id
window_id
action_slot_id

slot_input_revision

optional current_submission_id
finalized

updated_at
```

Key:
`(session_id, window_id, action_slot_id)`

Submit/Replace transaction:

1. claim semantic command idempotency;
2. lock gate;
3. require gate ACCEPTING;
4. lock target slot;
5. verify expected SlotInputRevision;
6. insert immutable ActionSubmission;
7. update current submission;
8. increment slot revision;
9. commit.

No wall-clock timestamp chooses the winner.

---

# 15. frozen_input_sets

Immutable selected-input snapshot.

```
session_id
window_id

base_revision

freeze_owner_command_key

input_set_hash

selected_entries_jsonb


created_at
```

Invariants:

- one per Window;
- immutable;
- each `selected_entries_jsonb` entry contains at least:
  - action_slot_id;
  - slot_input_revision;
  - submission_id;
  - submission_content_hash;
- `input_set_hash` is computed over Window/base identity plus the canonically ordered selected entries;
- resolver verification checks each persisted ActionSubmission rehashes to its selected submission_content_hash;
- selected ActionSubmission rows remain available for supported verification period;
- a competing close command seeing existing frozen set cannot create another resolution owner.

---

# 16. Freeze transaction

Freeze is intentionally separate from canonical commit.

Transaction lock order:

```
session_runtime
-> command_processing
-> window_input_gates
-> all window_slot_inputs in canonical slot order
```

Steps:

1. verify Session runtime revision == Window/gate base revision;
2. verify active canonical Window id matches gate;
3. verify Session not terminal;
4. require gate ACCEPTING;
5. deterministically choose selected submissions/defaults;
6. verify/capture each selected ActionSubmission content hash;
7. create FrozenInputSet whose hash includes selected content hashes;
8. set gate FROZEN + owner CommandKey;
9. attach frozen hash to CommandProcessingRecord;
10. set command PROCESSING/lease/generation;
11. commit.

No StreamRevision advances.

After commit the Session frontier is logically occupied by the frozen resolution until it commits, explicitly aborts or follows recovery policy.

---

# 17. resolution_attempts

Operational attempt telemetry.

```
attempt_id

session_id
window_id
frozen_input_set_hash

status

optional worker_ref
optional failure_code

started_at
optional finished_at
```

Not canonical.

May be compacted under operational retention.

CommandProcessingRecord owns logical resumability.

---

# 18. resolution_records

Immutable mechanical resolution evidence.

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

Selected input + random evidence retained according to resolver verification compatibility window.

World state is never rebuilt from ResolutionRecord instead of events.

---

# 19. resolution_debug_artifacts — optional

```
resolution_id
artifact_type
content_jsonb_or_bytes
content_hash
created_at
retention_class
```

For:

- ExpandedActions;
- full InteractionGraph;
- verbose plan traces.

Not required for state replay.

---

# 20. player_interaction_views

Immutable materialized PlayerInteractionView when an exact historical source view is required.

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
view_content_hash

created_at
```

Invariant:

```
player_view_key == H(canonical(view_content))
```

## Persist only when required

Materialize when:

- async presentation generation references the view;
- delivered presentation forensic evidence requires it;
- explicit playtest/debug mode requests retention.

Ordinary UI reads may compute views on demand.

## Security

Rows are not directly enumerable/readable by other participants merely by knowing a key.

Authorization remains participant/session based via authoritative API/RPC.

---

# 21. presentation_plans

Immutable structured narrative plan.

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

Invariant:

```
plan_hash == H(canonical(plan_content + relevant plan identity/version fields))
```

Narrative Direction cannot modify the InteractionSurface from the source PlayerInteractionView.

---

# 22. immutable presentation asset reference

Delivered gameplay presentation may reference external media only through immutable/versioned identity.

```
PresentationAssetRef
  asset_key
  content_hash
  media_type
  immutable_object_version_or_key
  optional generation_metadata_ref
```

A mutable bare URL is not sufficient historical evidence.

Physical media bytes may remain in object storage such as R2.

---

# 23. presentation_records

Immutable delivered-player evidence.

```
presentation_id

session_id
participant_ref

state_revision
player_view_key
presentation_plan_hash

generation_context_jsonb
generation_context_hash

source_build_outbox_id
optional accepted_build_lease_generation

exact_delivered_payload_jsonb_or_text
output_hash

generation_status
delivery_status

generated_at
optional delivered_at
```

Rules:

- delivered payload and referenced PresentationAssetRefs become immutable evidence;
- same generation task retries with identical generation context;
- DeliveryGuard prevents a superseded PlayerView from being delivered as current gameplay;
- generated but superseded output may be retained as non-delivered debug evidence.

---

# 24. outbox

Reliable at-least-once task queue inside PostgreSQL.

```
outbox_id

session_id
optional transition_key

task_type
deduplication_key

payload_jsonb
payload_hash

execution_context_jsonb
execution_context_hash

status:
  PENDING
  PROCESSING
  COMPLETED
  FAILED_RETRYABLE
  FAILED_TERMINAL

attempt_count

available_after

optional lease_owner
optional lease_until
lease_generation
optional last_error_code

created_at
optional completed_at
```

## Semantic task idempotency

For the chosen deduplication scope:

- same dedup key + same payload/context hashes -> same logical task;
- same dedup key + changed payload/context -> fail as `OUTBOX_IDEMPOTENCY_KEY_REUSE`.

## Claim/lease/fencing protocol

Short DB transaction:

1. select ready PENDING/expired-retry rows with row lock;
2. skip rows already locked by another claimant;
3. set PROCESSING, lease owner/until;
4. increment `lease_generation` and attempt count;
5. commit.

External work occurs outside the DB transaction.

Any result acceptance/completion MUST compare the worker's expected `lease_generation`.

A stale worker from an older generation cannot persist authoritative task result or mark the item completed.

Expired PROCESSING leases can be reclaimed with a new generation.

No PostgreSQL row lock is held while calling an LLM, realtime provider, analytics service or object storage.

---

# 24A. Presentation generation/delivery pipeline

Presentation work is split into two idempotent task stages.

## BUILD_PRESENTATION

Outbox payload contains stable:

- `presentation_id`;
- participant/audience;
- PlayerViewKey;
- state revision;
- PresentationGenerationContext hash/context.

The PlayerInteractionView content is already materialized/immutable.

Worker:

1. claims BUILD task and obtains `lease_generation`;
2. loads exact PlayerInteractionView + generation context;
3. runs Narrative Direction/Realization outside DB transaction;
4. starts result transaction;
5. locks/checks BUILD outbox row;
6. verifies expected `lease_generation` still current;
7. verifies output hashes and referenced assets;
8. creates/finalizes PresentationRecord generated payload for stable `presentation_id`;
9. records accepted build generation;
10. inserts idempotent `DELIVER_PRESENTATION` outbox task;
11. marks BUILD completed;
12. commits.

If the worker lost its fencing generation while generation was running, its output is discarded.

Multiple external LLM/media calls may still occur after crashes/lease expiry, but only one fenced result becomes the persisted presentation artifact.

## DELIVER_PRESENTATION

Uses stable `presentation_id` as delivery dedup identity.

Worker:

1. claims/fences delivery task;
2. runs DeliveryGuard against current PlayerInteractionView applicability;
3. if stale, marks PresentationRecord SUPERSEDED and completes without current delivery;
4. otherwise delivers using `presentation_id` as client/API idempotency identity;
5. records delivered status;
6. completes outbox item with matching generation.

If delivery transport retries after uncertainty, repeated `presentation_id` is treated idempotently by the player-facing delivery path.

---

# 25. Session creation transaction

One DB transaction:

1. resolve immutable scenario version;
2. create Session;
3. create SessionGenesis;
4. deterministically construct revision-0 CanonicalStateContent;
5. verify initial hash;
6. create session_runtime at revision 0;
7. commit.

Any failure rolls back all bootstrap rows.

No Domain Event at revision 0.

---

# 26. Canonical transition commit transaction

TransitionPlan is fully computed/validated before opening the commit transaction.

Lock order:

```
session_runtime
-> command_processing if command-owned
-> window_input_gate if relevant
```

Steps:

1. lock session_runtime;
2. verify current_revision == base_revision;
3. verify current state hash == before-state hash;
4. lock command record where relevant and verify expected `processing_generation`;
5. lock relevant gate where relevant;
6. verify gate/freeze owner consistency;
7. insert immutable ResolutionRecord if applicable;
8. insert immutable CanonicalTransitionRecord;
9. append ordered Session events;
10. replace session_runtime after-state/revision/hash;
11. close/cancel/finalize pending gate/slot state if Window ends;
12. mark command terminal result if command-owned;
13. insert required Outbox items with hashes/dedup keys;
14. commit.

The transaction never waits on external provider work.

---

# 27. Terminal Session pending-input invariant

If committed after-state is terminal:

- no gate may remain ACCEPTING or FROZEN;
- active Window must be resolved/cancelled/interrupted in the canonical event batch;
- relevant gate becomes CLOSED/CANCELLED in the same DB transaction;
- slot input state becomes non-writable;
- subsequent normal gameplay Submit/Replace commands reject.

This persistence invariant mirrors the accepted Domain Model.

---

# 28. Crash after freeze before commit

Expected durable intermediate state:

```
session_runtime: unchanged at base revision
canonical Window: still OPEN
gate: FROZEN
FrozenInputSet: exists
freeze_owner_command_key: set
CommandProcessingRecord: PROCESSING with lease + processing_generation
canonical transition: absent
```

Recovery:

- same CommandKey or authorized recovery worker claims expired processing lease;
- reuses same FrozenInputSet;
- recomputes/resumes deterministic resolution;
- commits at same base revision.

If base revision changed exceptionally, normal resolution aborts and explicit recovery semantics apply.

---

# 29. No-op command

A NO_OP:

- stores/returns CommandResult;
- creates no CanonicalTransitionRecord;
- appends no event;
- advances no revision;
- does not change session_runtime state/hash.

---

# 30. Event index-metadata repair policy

The following are non-authoritative acceleration fields:

- event_family_index;
- event_code_index;
- event_schema_version_index;
- logical_tick_index.

Provide consistency tooling that can scan:

```
canonical_event_bytes
-> parse
-> verify event hash
-> recompute index metadata
-> compare/repair index metadata
```

Canonical bytes and hashes are never overwritten by this repair.

---

# 31. Referential integrity

Use relational foreign keys for stable owner relations when they do not distort event/history design.

Candidates:

- Session -> ScenarioVersion;
- Genesis -> Session;
- Runtime -> Session;
- event/transition -> Session;
- submission/gate/input -> Session;
- resolution/presentation/outbox -> Session.

Do not force database FKs for opaque refs embedded in canonical event bytes/JSONB.

Those are validated through domain/compiler/replay rules.

---

# 32. JSONB policy

Use JSONB for extensible typed structures.

Rules:

1. every replay-relevant JSONB contract carries/derives schema version;
2. application/schema validation occurs before write;
3. canonical hashes use application canonical serialization, not PostgreSQL output serialization;
4. no blanket GIN indexes;
5. add JSONB indexes only for measured query paths;
6. current canonical state JSONB is replaced by validated whole after-state in normal gameplay commit.

PostgreSQL provides JSONB containment/path querying and GIN operator classes, but those capabilities do not justify indexing every payload. 

---

# 33. Canonical bytes policy

Exact canonical serialized bytes are required at least for:

- CompiledScenarioBundle;
- MechanicsVersionManifest;
- CanonicalEventContent.

Store the declared canonicalization/hash version through MechanicsVersionManifest.

Canonical bytes are immutable after committed/published.

Noncanonical derived artifacts can use JSONB + hash unless exact-byte retention becomes required.

---

# 34. Access/security zones

## Backend-only authoritative writes

- session_genesis;
- session_runtime;
- session_events;
- canonical_transitions;
- command_processing;
- gates/slot inputs/frozen sets;
- resolution evidence;
- outbox.

## Player API

Players receive authorized PlayerInteractionView/presentation data.

Browser cannot query full canonical state or another player's pending/view data through generic table access.

## Analytics

Only deliberate sanitized properties/events.

Concrete RLS identities/policies wait for ADR-015 and implementation schema.

---

# 35. Initial index candidates

Not final DDL.

## Required integrity/order

- unique Session event revision;
- unique Session transition/batch index;
- unique Session transition key;
- unique Session command key;
- unique Window/Slot;
- one FrozenInputSet per Window;
- unique PlayerViewKey;
- unique Scenario version/hash;
- unique Outbox dedup key in defined scope.

## Operational query indexes

- submissions by Session/Window/Slot;
- outbox ready work by status/available_after;
- presentations by Session/Participant/time;
- transitions by ResolutionId.

## Partial unique candidate

At most one nonterminal gate per Session.

PostgreSQL partial UNIQUE indexes can enforce uniqueness over a qualifying subset; exact DDL/predicate is deferred to schema design.

---

# 36. Worker/recovery monitoring

Operational alerts should detect:

- PROCESSING commands past lease;
- FROZEN gates without active/recoverable command owner;
- PROCESSING outbox items past lease;
- repeated retry storms;
- session_runtime revision/hash mismatch with latest transition;
- event index metadata mismatch;
- orphan pending gate after terminal Session.

Monitoring is operational; it does not change canonical mechanics.

---

# 37. Partitioning

No initial partitioning.

Revisit event/audit/presentation partitioning after measured:

- row count;
- maintenance/vacuum pressure;
- replay scan patterns;
- backup/archive needs.

Do not design uniqueness/idempotency around hypothetical future partitions now.

---

# 38. Snapshots

No historical snapshot table initially.

`session_runtime` is current synchronous projection, not historical authority.

Future replay snapshots are optimization only and require:

- revision;
- state hash;
- bundle/manifest hash;
- validation against event replay.

Genesis + events always remain sufficient.

---

# 39. Retention classes

## Mechanical reconstruction

Retain:
- immutable Scenario Bundle bytes;
- SessionGenesis;
- Session events.

## Mechanical verification

Retain for supported compatibility window:
- selected ActionSubmissions;
- FrozenInputSet;
- ResolutionRecord;
- required NamedRandomDraw evidence;
- CanonicalTransitionRecord;
- relevant build/version artifacts.

## Player-experience forensic

Delivered PresentationRecord and immutable asset refs under product/privacy retention policy.

## Optional audit

Shorter retention allowed:
- replaced/nonselected submissions;
- raw natural-language input;
- parser candidates;
- verbose resolution artifacts;
- worker attempt details.

## Operational

Completed Outbox/expired leases can be compacted after reliability/dedup retention window.

---

# 40. Backup/restore/rebuild hierarchy

Minimum mechanical restore:

```
scenario_versions
+ sessions/session_genesis
+ session_events
= reconstruct session_runtime
```

Then rebuild:

- derived indexes;
- optional read projections.

For resolver verification/forensic workflows also restore retained:

- selected submissions/FrozenInputSet;
- ResolutionRecords;
- transitions/random evidence;
- PresentationRecords as required.

Outbox completed history and PresenceState are not required for canonical state restoration.

---

# 41. Persistence invariants

1. exactly one Genesis per Session;
2. exactly one runtime row per Session;
3. canonical revision advances only through canonical commit;
4. event revisions contiguous;
5. committed event bytes immutable;
6. transition range/hash agrees with event rows;
7. runtime after hash equals transition after hash;
8. all multi-row concurrency transactions respect global lock order;
9. freeze verifies runtime base revision/window before committing FROZEN state;
10. FROZEN gate has one owner CommandKey;
11. selected input immutable;
12. command PROCESSING state is recoverable via lease and fenced by processing_generation;
13. stale command-processing generation cannot finalize/attach transition;
14. same CommandKey cannot apply twice;
15. every selected ActionSubmission has immutable submission_content_hash;
16. FrozenInputSet hash commits to selected submission content hashes;
17. outbox claim is lease-based/recoverable and fenced by lease_generation;
18. stale outbox generation cannot finalize authoritative task result;
19. same outbox dedup key cannot change payload/context semantics;
20. required outbox insert is atomic with canonical transition;
21. event index metadata is non-authoritative/rebuildable;
22. current state convenience duplicates are avoided by default;
23. materialized PlayerView/plan/output hash verifies stored content;
24. presentation build/delivery use stable presentation_id and fenced task generations;
25. delivered external assets use immutable/versioned identity;
26. terminal Session leaves no writable/frozen pending input gate;
27. selected submissions remain available for supported resolver verification;
28. derived indexes/caches can be deleted without breaking state reconstruction;
29. direct browser mutation of internal/canonical tables is forbidden.

---

# 42. v0.2 red-team gate

Before acceptance verify:

1. canonical worker race;
2. freeze vs replacement;
3. freeze vs terminal/admin transition;
4. deadline vs explicit close;
5. worker dies after freeze;
6. command worker lease expires while original worker wakes up late;
7. stale command worker tries canonical commit after takeover;
8. outbox worker dies before side effect;
9. outbox worker dies after external generation but before result persistence;
10. stale outbox worker tries to persist output after lease takeover;
11. duplicate presentation delivery;
12. dedup key payload mismatch;
13. selected ActionSubmission row content corruption;
14. event index metadata corruption;
15. current projection corruption/rebuild;
16. materialized PlayerView hash mismatch;
17. mutable media URL replaced after delivery;
18. selected submission retention deletion attempt;
19. terminal transition with frozen gate;
20. large JSONB current state;
21. event growth without partitioning;
22. full restore from Genesis + events;
23. manual admin mutation attempt.

No SQL migration before this gate passes.


---

# 43. v0.3 change record

Persistence Model v0.3 adds:

- command-processing fencing generation;
- outbox lease fencing generation;
- ActionSubmission mechanical content hash;
- FrozenInputSet commitment to selected submission hashes;
- stable presentation task/presentation identity;
- fenced presentation-result acceptance;
- explicit BUILD_PRESENTATION -> DELIVER_PRESENTATION split;
- presentation-idempotent delivery semantics.

No accepted architecture, contract or domain-model decision is changed.
