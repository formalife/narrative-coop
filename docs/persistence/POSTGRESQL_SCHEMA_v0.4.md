# POSTGRESQL SCHEMA v0.4

**Status:** PROPOSED  
**Date:** 2026-10-02  
**Supersedes as working proposal:** POSTGRESQL_SCHEMA_v0.3  
**Governing architecture:** Architecture Baseline v0.3 — ACCEPTED  
**Governing contracts:** Domain Contracts v0.3 — ACCEPTED  
**Governing domain model:** Domain Model v0.2 — ACCEPTED  
**Governing persistence model:** Persistence Data Model v0.3 — ACCEPTED

---

# 1. Scope

This is the concrete PostgreSQL schema proposal before migration files.

It freezes, if accepted:

- table boundaries;
- core SQL column types;
- PK/FK/UNIQUE/CHECK structure;
- essential indexes;
- logical DB privilege classes;
- transaction/lock protocol;
- hash framing for persisted interaction artifacts;
- migration strategy.

It does not freeze:

- UUID generation algorithm;
- provider-specific role names;
- ADR-015 user identity;
- participant RLS policies;
- JSONB payload indexes;
- partitioning;
- snapshots;
- retention durations.

---

# 2. Internal schema boundary

```sql
CREATE SCHEMA engine;
REVOKE ALL ON SCHEMA engine FROM PUBLIC;
```

Authoritative tables live only in `engine`.

Browser-facing roles receive no direct table privileges.

The internal schema must not be exposed as a generic browser REST/table surface.

---

# 3. Logical DB privilege classes

Provider role names may differ, but deployment must preserve these capabilities.

## engine_owner / migrator

- owns schema/tables/functions;
- DDL;
- grants/default privileges;
- not used by normal application runtime.

## engine_publisher

- SELECT Scenario versions;
- INSERT new Scenario versions;
- UPDATE only publication-lifecycle/search metadata columns;
- cannot change canonical bundle bytes/hash after publish.

## engine_runtime

- SELECT required internal data;
- INSERT Session/Genesis/current runtime initial rows;
- UPDATE SessionRuntime through canonical transaction protocol;
- INSERT immutable canonical events/transitions/resolution evidence;
- INSERT/UPDATE command + pending-input rows;
- INSERT PlayerInteractionViews;
- INSERT PENDING PresentationRecords;
- INSERT Outbox work;
- no UPDATE/DELETE permission on immutable canonical-history relations.

## engine_worker

- SELECT Outbox / materialized PlayerViews / Presentation context;
- claim/update Outbox with fencing protocol;
- INSERT immutable PresentationPlans;
- UPDATE PresentationRecords only through the accepted generation/delivery state machine;
- no canonical event/runtime mutation.

## engine_ops_readonly

- SELECT explicitly granted internal operational/admin data;
- no DML.

No browser/client role maps directly to these internal authority roles.

---

# 4. Default privilege policy

Migration setup must prevent future objects from accidentally becoming public.

Conceptual baseline:

```sql
ALTER DEFAULT PRIVILEGES
  FOR ROLE <engine_owner_role>
  IN SCHEMA engine
  REVOKE ALL ON TABLES FROM PUBLIC;

ALTER DEFAULT PRIVILEGES
  FOR ROLE <engine_owner_role>
  IN SCHEMA engine
  REVOKE ALL ON SEQUENCES FROM PUBLIC;

ALTER DEFAULT PRIVILEGES
  FOR ROLE <engine_owner_role>
  IN SCHEMA engine
  REVOKE ALL ON FUNCTIONS FROM PUBLIC;
```

Exact deployment role substitution is provider configuration.

---

# 5. SQL type conventions

## UUID

PostgreSQL `uuid` for generated runtime instance identifiers.

Generation algorithm remains open.

## text

For semantic/stable keys and schema/version refs.

## bytea

For:
- hashes;
- canonical serialized bytes;
- Session seed.

## jsonb

For validated extensible structured values.

## bigint

For:
- StreamRevision;
- SlotInputRevision;
- LogicalTick;
- fencing generations.

## timestamptz

Operational timing only.

---

# 6. Status representation

Use `text + CHECK`, not PostgreSQL ENUM, for mutable application/domain statuses.

This keeps status evolution explicit but migration-friendly.

---

# 7. scenario_versions

```sql
CREATE TABLE engine.scenario_versions (
  scenario_id uuid NOT NULL,
  scenario_version text NOT NULL,

  bundle_hash bytea NOT NULL,
  bundle_schema_version text NOT NULL,

  compiled_bundle_canonical_bytes bytea NOT NULL,

  status text NOT NULL
    CHECK (status IN ('PUBLISHED', 'RETIRED')),

  published_at timestamptz NOT NULL,
  retired_at timestamptz NULL,

  searchable_metadata jsonb NULL,

  PRIMARY KEY (scenario_id, scenario_version),

  UNIQUE (bundle_hash),
  UNIQUE (scenario_id, scenario_version, bundle_hash),

  CHECK (
    (status = 'PUBLISHED' AND retired_at IS NULL)
    OR
    (status = 'RETIRED' AND retired_at IS NOT NULL)
  ),

  CHECK (
    retired_at IS NULL
    OR retired_at >= published_at
  )
);
```

## Privilege rule

The publisher role may UPDATE only:
- `status`;
- `retired_at`;
- `searchable_metadata`.

It cannot UPDATE canonical bundle/hash/schema-version columns.

---

# 8. sessions

```sql
CREATE TABLE engine.sessions (
  session_id uuid PRIMARY KEY,

  scenario_id uuid NOT NULL,
  scenario_version text NOT NULL,
  scenario_bundle_hash bytea NOT NULL,

  created_at timestamptz NOT NULL,

  FOREIGN KEY (
    scenario_id,
    scenario_version,
    scenario_bundle_hash
  )
  REFERENCES engine.scenario_versions (
    scenario_id,
    scenario_version,
    bundle_hash
  )
  ON UPDATE RESTRICT
  ON DELETE RESTRICT,

  UNIQUE (session_id, scenario_bundle_hash)
);
```

No terminal lifecycle field is duplicated here.

Terminal state is canonical Session state/event history.

---

# 9. session_genesis

```sql
CREATE TABLE engine.session_genesis (
  session_id uuid PRIMARY KEY,

  mechanics_manifest_canonical_bytes bytea NOT NULL,
  mechanics_manifest_hash bytea NOT NULL,

  scenario_bundle_hash bytea NOT NULL,

  session_seed bytea NOT NULL,

  initial_canonical_state_hash bytea NOT NULL,

  created_at timestamptz NOT NULL,

  FOREIGN KEY (session_id, scenario_bundle_hash)
    REFERENCES engine.sessions (
      session_id,
      scenario_bundle_hash
    )
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  UNIQUE (
    session_id,
    mechanics_manifest_hash
  )
);
```

Runtime role receives INSERT/SELECT, not UPDATE/DELETE.

---

# 10. session_runtime

```sql
CREATE TABLE engine.session_runtime (
  session_id uuid PRIMARY KEY,

  current_revision bigint NOT NULL
    CHECK (current_revision >= 0),

  state_hash bytea NOT NULL,
  mechanics_manifest_hash bytea NOT NULL,

  canonical_state_content jsonb NOT NULL
    CHECK (
      jsonb_typeof(canonical_state_content) = 'object'
    ),

  updated_at timestamptz NOT NULL,

  FOREIGN KEY (
    session_id,
    mechanics_manifest_hash
  )
  REFERENCES engine.session_genesis (
    session_id,
    mechanics_manifest_hash
  )
  ON UPDATE RESTRICT
  ON DELETE RESTRICT
);
```

This is the mutable current projection + canonical frontier lock anchor.

No duplicated canonical lifecycle fields in the initial schema.

---

# 11. command_processing

```sql
CREATE TABLE engine.command_processing (
  session_id uuid NOT NULL,
  command_key text NOT NULL,

  command_type text NOT NULL,

  semantic_payload_hash bytea NOT NULL,
  semantic_payload jsonb NOT NULL,

  principal_type text NOT NULL,
  principal_data jsonb NOT NULL,

  processing_status text NOT NULL
    CHECK (
      processing_status IN (
        'RECEIVED',
        'ACCEPTED_PENDING',
        'PROCESSING',
        'COMPLETED',
        'REJECTED',
        'FAILED_RETRYABLE',
        'FAILED_TERMINAL'
      )
    ),

  frozen_input_set_hash bytea NULL,
  transition_key text NULL,

  command_result jsonb NULL,

  processing_owner text NULL,
  processing_lease_until timestamptz NULL,
  retry_after timestamptz NULL,

  processing_generation bigint NOT NULL DEFAULT 0
    CHECK (processing_generation >= 0),

  processing_attempt_count integer NOT NULL DEFAULT 0
    CHECK (processing_attempt_count >= 0),

  first_received_at timestamptz NOT NULL,
  last_updated_at timestamptz NOT NULL,

  PRIMARY KEY (session_id, command_key),

  FOREIGN KEY (session_id)
    REFERENCES engine.sessions(session_id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  CHECK (
    processing_status <> 'PROCESSING'
    OR (
      processing_owner IS NOT NULL
      AND processing_lease_until IS NOT NULL
      AND processing_generation > 0
    )
  ),

  CHECK (
    processing_status <> 'FAILED_RETRYABLE'
    OR retry_after IS NOT NULL
  )
);
```

Retry scheduling and processing lease are separate semantics.

---

# 12. command retry index

```sql
CREATE INDEX ix_command_retryable
ON engine.command_processing (
  retry_after,
  session_id,
  command_key
)
WHERE processing_status = 'FAILED_RETRYABLE';
```

PROCESSING lease monitoring:

```sql
CREATE INDEX ix_command_processing_lease
ON engine.command_processing (
  processing_lease_until,
  session_id,
  command_key
)
WHERE processing_status = 'PROCESSING';
```

---

# 13. action_submissions

```sql
CREATE TABLE engine.action_submissions (
  submission_id uuid PRIMARY KEY,

  session_id uuid NOT NULL,
  window_id uuid NOT NULL,
  action_slot_id text NOT NULL,

  action_type_id text NOT NULL,

  parameters jsonb NOT NULL
    CHECK (jsonb_typeof(parameters) = 'object'),

  selected_target_refs jsonb NOT NULL
    CHECK (jsonb_typeof(selected_target_refs) = 'array'),

  submission_content_hash bytea NOT NULL,

  source_type text NOT NULL
    CHECK (
      source_type IN (
        'OPTION',
        'STRUCTURED_UI',
        'NATURAL_LANGUAGE_PARSED',
        'SCRIPTED_TEST',
        'SYNTHETIC_PLAYER'
      )
    ),

  source_player_view_key bytea NULL,
  source_presentation_id uuid NULL,

  created_by_command_key text NOT NULL,

  received_at timestamptz NOT NULL,

  FOREIGN KEY (session_id)
    REFERENCES engine.sessions(session_id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  UNIQUE (
    submission_id,
    session_id,
    window_id,
    action_slot_id
  )
);
```

Mechanical submission hash input is frozen by Persistence v0.3.

---

# 14. window_input_gates

```sql
CREATE TABLE engine.window_input_gates (
  session_id uuid NOT NULL,
  window_id uuid NOT NULL,

  base_revision bigint NOT NULL
    CHECK (base_revision >= 0),

  gate_status text NOT NULL
    CHECK (
      gate_status IN (
        'ACCEPTING',
        'FROZEN',
        'CANCELLED',
        'CLOSED'
      )
    ),

  frozen_input_set_hash bytea NULL,
  freeze_owner_command_key text NULL,
  planned_transition_key text NULL,

  created_at timestamptz NOT NULL,
  updated_at timestamptz NOT NULL,
  frozen_at timestamptz NULL,
  closed_at timestamptz NULL,

  PRIMARY KEY (session_id, window_id),

  FOREIGN KEY (session_id)
    REFERENCES engine.sessions(session_id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  CHECK (
    gate_status <> 'FROZEN'
    OR (
      frozen_input_set_hash IS NOT NULL
      AND freeze_owner_command_key IS NOT NULL
      AND frozen_at IS NOT NULL
    )
  ),

  CHECK (
    gate_status NOT IN ('CANCELLED', 'CLOSED')
    OR closed_at IS NOT NULL
  )
);
```

One nonterminal input gate:

```sql
CREATE UNIQUE INDEX ux_one_nonterminal_gate_per_session
ON engine.window_input_gates(session_id)
WHERE gate_status IN ('ACCEPTING', 'FROZEN');
```

---

# 15. window_slot_inputs

```sql
CREATE TABLE engine.window_slot_inputs (
  session_id uuid NOT NULL,
  window_id uuid NOT NULL,
  action_slot_id text NOT NULL,

  slot_input_revision bigint NOT NULL DEFAULT 0
    CHECK (slot_input_revision >= 0),

  current_submission_id uuid NULL,

  finalized boolean NOT NULL DEFAULT false,

  updated_at timestamptz NOT NULL,

  PRIMARY KEY (
    session_id,
    window_id,
    action_slot_id
  ),

  FOREIGN KEY (
    session_id,
    window_id
  )
    REFERENCES engine.window_input_gates(
      session_id,
      window_id
    )
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  FOREIGN KEY (
    current_submission_id,
    session_id,
    window_id,
    action_slot_id
  )
    REFERENCES engine.action_submissions(
      submission_id,
      session_id,
      window_id,
      action_slot_id
    )
    ON UPDATE RESTRICT
    ON DELETE RESTRICT
);
```

---

# 16. frozen_input_sets

```sql
CREATE TABLE engine.frozen_input_sets (
  session_id uuid NOT NULL,
  window_id uuid NOT NULL,

  base_revision bigint NOT NULL
    CHECK (base_revision >= 0),

  freeze_owner_command_key text NOT NULL,

  input_set_hash bytea NOT NULL,

  selected_entries jsonb NOT NULL
    CHECK (jsonb_typeof(selected_entries) = 'array'),

  created_at timestamptz NOT NULL,

  PRIMARY KEY (session_id, window_id),

  FOREIGN KEY (
    session_id,
    window_id
  )
    REFERENCES engine.window_input_gates(
      session_id,
      window_id
    )
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  UNIQUE (
    session_id,
    window_id,
    input_set_hash
  )
);
```

---

# 17. resolution_attempts

```sql
CREATE TABLE engine.resolution_attempts (
  attempt_id uuid PRIMARY KEY,

  session_id uuid NOT NULL,
  window_id uuid NOT NULL,
  frozen_input_set_hash bytea NOT NULL,

  status text NOT NULL
    CHECK (
      status IN (
        'PREPARING',
        'RESOLVING',
        'COMMITTING',
        'COMMITTED',
        'ABORTED'
      )
    ),

  worker_ref text NULL,
  failure_code text NULL,

  started_at timestamptz NOT NULL,
  finished_at timestamptz NULL,

  FOREIGN KEY (session_id)
    REFERENCES engine.sessions(session_id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT
);
```

Operational only.

---

# 18. resolution_records

```sql
CREATE TABLE engine.resolution_records (
  resolution_id uuid PRIMARY KEY,

  session_id uuid NOT NULL,
  window_id uuid NOT NULL,

  frozen_input_set_hash bytea NOT NULL,

  selected_submissions jsonb NOT NULL,
  admission_results jsonb NOT NULL,
  action_outcomes jsonb NOT NULL,
  named_random_draws jsonb NOT NULL,

  interaction_graph_hash bytea NOT NULL,
  action_resolution_hash bytea NOT NULL,

  mechanics_manifest_hash bytea NOT NULL,

  created_at timestamptz NOT NULL,

  FOREIGN KEY (
    session_id,
    window_id,
    frozen_input_set_hash
  )
    REFERENCES engine.frozen_input_sets(
      session_id,
      window_id,
      input_set_hash
    )
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  UNIQUE (session_id, window_id)
);
```

Resolution evidence is now relationally bound to the exact frozen input hash.

---

# 19. canonical_transitions

```sql
CREATE TABLE engine.canonical_transitions (
  session_id uuid NOT NULL,
  transition_key text NOT NULL,

  transition_kind text NOT NULL
    CHECK (
      transition_kind IN (
        'WINDOW_RESOLUTION',
        'PROGRESSION',
        'SCHEDULED_EFFECT',
        'SESSION_LIFECYCLE',
        'DEADLINE',
        'ADMIN_COMPENSATION'
      )
    ),

  base_revision bigint NOT NULL
    CHECK (base_revision >= 0),

  resulting_revision bigint NOT NULL,

  event_count integer NOT NULL
    CHECK (event_count > 0),

  trigger_command_key text NULL,
  resolution_id uuid NULL,

  before_state_hash bytea NOT NULL,
  event_batch_hash bytea NOT NULL,
  after_state_hash bytea NOT NULL,

  mechanics_manifest_hash bytea NOT NULL,

  committed_at timestamptz NOT NULL,

  PRIMARY KEY (session_id, transition_key),

  FOREIGN KEY (session_id)
    REFERENCES engine.sessions(session_id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  CHECK (
    resulting_revision = base_revision + event_count
  ),

  UNIQUE (session_id, base_revision),
  UNIQUE (session_id, resulting_revision)
);
```

Optional evidence refs stay soft because evidence retention can differ from canonical history retention.

---

# 20. session_events

```sql
CREATE TABLE engine.session_events (
  session_id uuid NOT NULL,

  stream_revision bigint NOT NULL
    CHECK (stream_revision > 0),

  transition_key text NOT NULL,
  batch_index integer NOT NULL
    CHECK (batch_index >= 0),

  event_family_index text NOT NULL,
  event_code_index text NOT NULL,
  event_schema_version_index text NOT NULL,
  logical_tick_index bigint NOT NULL,

  canonical_event_bytes bytea NOT NULL,
  canonical_event_hash bytea NOT NULL,

  recorded_at timestamptz NOT NULL,

  PRIMARY KEY (
    session_id,
    stream_revision
  ),

  UNIQUE (
    session_id,
    transition_key,
    batch_index
  ),

  FOREIGN KEY (
    session_id,
    transition_key
  )
    REFERENCES engine.canonical_transitions(
      session_id,
      transition_key
    )
    DEFERRABLE INITIALLY DEFERRED
    ON UPDATE RESTRICT
    ON DELETE RESTRICT
);
```

Index columns are non-authoritative query metadata.

Canonical bytes/hash are immutable authority.

---

# 21. canonical event/batch hashes

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

Algorithm/canonicalizer come from MechanicsVersionManifest.

---

# 22. PlayerInteractionView hash framing

Exact hash input:

```
PlayerViewHashInput = {
  session_id,
  participant_ref,
  state_revision,

  window_id,
  window_input_gate_hash,
  own_slot_input_revision,

  view_schema_version,

  view_content
}
```

```
player_view_key =
  H(canonical(PlayerViewHashInput))
```

Fields with absent values are represented according to the project canonical serialization contract.

Created-at timestamps are excluded.

---

# 23. player_interaction_views

```sql
CREATE TABLE engine.player_interaction_views (
  session_id uuid NOT NULL,
  player_view_key bytea NOT NULL,

  participant_ref uuid NOT NULL,

  state_revision bigint NOT NULL
    CHECK (state_revision >= 0),

  window_id uuid NULL,
  window_input_gate_hash bytea NULL,

  own_slot_input_revision bigint NULL
    CHECK (
      own_slot_input_revision IS NULL
      OR own_slot_input_revision >= 0
    ),

  view_schema_version text NOT NULL,

  view_content jsonb NOT NULL
    CHECK (
      jsonb_typeof(view_content) = 'object'
    ),

  created_at timestamptz NOT NULL,

  PRIMARY KEY (
    session_id,
    player_view_key
  ),

  FOREIGN KEY (session_id)
    REFERENCES engine.sessions(session_id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT
);
```

The application/verifier recomputes PlayerViewKey from the complete framing object.

No misleading SQL CHECK claims to hash only the JSON fragment.

Rows are immutable once inserted.

---

# 24. PresentationPlan hash framing

```
PresentationPlanHashInput = {
  session_id,
  participant_ref,
  state_revision,
  player_view_key,

  plan_version,
  presentation_policy_version,

  plan_content
}
```

```
plan_hash =
  H(canonical(PresentationPlanHashInput))
```

Random `presentation_plan_id` and operational timestamps are excluded from content hash.

---

# 25. presentation_plans

```sql
CREATE TABLE engine.presentation_plans (
  presentation_plan_id uuid PRIMARY KEY,

  session_id uuid NOT NULL,
  participant_ref uuid NOT NULL,

  state_revision bigint NOT NULL
    CHECK (state_revision >= 0),

  player_view_key bytea NOT NULL,

  plan_version text NOT NULL,
  presentation_policy_version text NOT NULL,

  plan_content jsonb NOT NULL
    CHECK (
      jsonb_typeof(plan_content) = 'object'
    ),

  plan_hash bytea NOT NULL,

  created_at timestamptz NOT NULL,

  FOREIGN KEY (
    session_id,
    player_view_key
  )
    REFERENCES engine.player_interaction_views(
      session_id,
      player_view_key
    )
    ON UPDATE RESTRICT
    ON DELETE RESTRICT
);
```

Immutable after insert.

---

# 26. Delivered presentation hash framing

```
DeliveredPresentationHashInput = {
  rendered_payload
}
```

`rendered_payload` contains immutable PresentationAssetRefs for any external media.

```
output_hash =
  H(canonical(DeliveredPresentationHashInput))
```

The Presentation row identity, audience, plan hash and generation context remain separate causal metadata.

---

# 27. outbox

Defined before PresentationRecord so PresentationRecord can hold a soft source-build identifier without circular FK requirements.

```sql
CREATE TABLE engine.outbox (
  outbox_id uuid PRIMARY KEY,

  session_id uuid NOT NULL,
  transition_key text NULL,

  task_type text NOT NULL,
  deduplication_key text NOT NULL,

  player_view_key bytea NULL,
  presentation_id uuid NULL,

  payload jsonb NOT NULL,
  payload_hash bytea NOT NULL,

  execution_context jsonb NOT NULL,
  execution_context_hash bytea NOT NULL,

  status text NOT NULL
    CHECK (
      status IN (
        'PENDING',
        'PROCESSING',
        'COMPLETED',
        'FAILED_RETRYABLE',
        'FAILED_TERMINAL'
      )
    ),

  attempt_count integer NOT NULL DEFAULT 0
    CHECK (attempt_count >= 0),

  available_after timestamptz NOT NULL,

  lease_owner text NULL,
  lease_until timestamptz NULL,

  lease_generation bigint NOT NULL DEFAULT 0
    CHECK (lease_generation >= 0),

  last_error_code text NULL,

  created_at timestamptz NOT NULL,
  completed_at timestamptz NULL,

  FOREIGN KEY (session_id)
    REFERENCES engine.sessions(session_id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  FOREIGN KEY (
    session_id,
    player_view_key
  )
    REFERENCES engine.player_interaction_views(
      session_id,
      player_view_key
    )
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  UNIQUE (
    session_id,
    task_type,
    deduplication_key
  ),

  CHECK (
    task_type NOT IN (
      'BUILD_PRESENTATION',
      'DELIVER_PRESENTATION'
    )
    OR (
      player_view_key IS NOT NULL
      AND presentation_id IS NOT NULL
    )
  ),

  CHECK (
    status <> 'PROCESSING'
    OR (
      lease_owner IS NOT NULL
      AND lease_until IS NOT NULL
      AND lease_generation > 0
    )
  ),

  CHECK (
    status <> 'COMPLETED'
    OR completed_at IS NOT NULL
  )
);
```

For non-presentation tasks, PlayerViewKey/PresentationId may be null.

---

# 28. presentation_records

One stable PresentationRecord is allocated before external generation.

```sql
CREATE TABLE engine.presentation_records (
  presentation_id uuid PRIMARY KEY,

  session_id uuid NOT NULL,
  participant_ref uuid NOT NULL,

  state_revision bigint NOT NULL
    CHECK (state_revision >= 0),

  player_view_key bytea NOT NULL,

  presentation_plan_id uuid NULL,
  presentation_plan_hash bytea NULL,

  generation_context jsonb NOT NULL,
  generation_context_hash bytea NOT NULL,

  source_build_outbox_id uuid NOT NULL,

  accepted_build_lease_generation bigint NULL
    CHECK (
      accepted_build_lease_generation IS NULL
      OR accepted_build_lease_generation > 0
    ),

  rendered_payload jsonb NULL,
  output_hash bytea NULL,

  generation_status text NOT NULL
    CHECK (
      generation_status IN (
        'PENDING',
        'GENERATED',
        'FAILED'
      )
    ),

  delivery_status text NOT NULL
    CHECK (
      delivery_status IN (
        'PENDING',
        'DELIVERED',
        'FAILED',
        'SUPERSEDED'
      )
    ),

  generated_at timestamptz NULL,
  delivered_at timestamptz NULL,

  FOREIGN KEY (
    session_id,
    player_view_key
  )
    REFERENCES engine.player_interaction_views(
      session_id,
      player_view_key
    )
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  FOREIGN KEY (presentation_plan_id)
    REFERENCES engine.presentation_plans(
      presentation_plan_id
    )
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  -- PENDING generation has no partial generated artifact.
  CHECK (
    generation_status <> 'PENDING'
    OR (
      presentation_plan_id IS NULL
      AND presentation_plan_hash IS NULL
      AND accepted_build_lease_generation IS NULL
      AND rendered_payload IS NULL
      AND output_hash IS NULL
      AND generated_at IS NULL
    )
  ),

  -- GENERATED appears atomically complete.
  CHECK (
    generation_status <> 'GENERATED'
    OR (
      presentation_plan_id IS NOT NULL
      AND presentation_plan_hash IS NOT NULL
      AND accepted_build_lease_generation IS NOT NULL
      AND rendered_payload IS NOT NULL
      AND output_hash IS NOT NULL
      AND generated_at IS NOT NULL
    )
  ),

  -- Terminal generation failure has no generated artifact.
  CHECK (
    generation_status <> 'FAILED'
    OR (
      presentation_plan_id IS NULL
      AND presentation_plan_hash IS NULL
      AND accepted_build_lease_generation IS NULL
      AND rendered_payload IS NULL
      AND output_hash IS NULL
      AND generated_at IS NULL
      AND delivery_status = 'FAILED'
    )
  ),

  -- Delivery requires a generated artifact.
  CHECK (
    delivery_status <> 'DELIVERED'
    OR generation_status = 'GENERATED'
  ),

  CHECK (
    delivery_status <> 'FAILED'
    OR generation_status IN ('GENERATED', 'FAILED')
  ),

  CHECK (
    delivery_status <> 'SUPERSEDED'
    OR generation_status IN ('PENDING', 'GENERATED')
  ),

  -- delivered_at exists exactly for actual delivery.
  CHECK (
    (delivery_status = 'DELIVERED' AND delivered_at IS NOT NULL)
    OR
    (delivery_status <> 'DELIVERED' AND delivered_at IS NULL)
  )
);
```

`source_build_outbox_id` remains a soft historical identifier because completed Outbox rows may later be compacted.

---

# 28A. Outbox -> PresentationRecord integrity

After `presentation_records` exists:

```sql
ALTER TABLE engine.outbox
ADD CONSTRAINT fk_outbox_presentation
FOREIGN KEY (presentation_id)
REFERENCES engine.presentation_records(presentation_id)
DEFERRABLE INITIALLY DEFERRED
ON UPDATE RESTRICT
ON DELETE RESTRICT;
```

Non-presentation tasks keep `presentation_id = NULL`.

For BUILD/DELIVER tasks the existing Outbox CHECK requires a PresentationId and PlayerViewKey.

The FK is deferred so the PENDING PresentationRecord and BUILD Outbox task can be inserted in either order within the same transaction.

---

# 29. PresentationRecord lifecycle enforcement

One narrow trigger protects temporal immutability.

It contains no game/narrative decision logic.

## Immutable from creation

These fields never change after INSERT:

- presentation_id;
- session_id;
- participant_ref;
- state_revision;
- player_view_key;
- generation_context;
- generation_context_hash;
- source_build_outbox_id.

## State rules

### Generation PENDING

Allowed:

- stay PENDING with no generated-result field changes;
- transition atomically to GENERATED;
- transition to FAILED;
- become delivery SUPERSEDED while remaining generation PENDING.

### Generation GENERATED

Generation/source/output fields are frozen.

Only delivery state can move from PENDING to:
- DELIVERED;
- FAILED;
- SUPERSEDED.

### Terminal delivery

DELIVERED / FAILED / SUPERSEDED rows are fully immutable.

## Trigger

```sql
CREATE FUNCTION engine.guard_presentation_record_update()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  -- Core identity and generation context are immutable from creation.
  IF ROW(
    NEW.presentation_id,
    NEW.session_id,
    NEW.participant_ref,
    NEW.state_revision,
    NEW.player_view_key,
    NEW.generation_context,
    NEW.generation_context_hash,
    NEW.source_build_outbox_id
  )
  IS DISTINCT FROM
  ROW(
    OLD.presentation_id,
    OLD.session_id,
    OLD.participant_ref,
    OLD.state_revision,
    OLD.player_view_key,
    OLD.generation_context,
    OLD.generation_context_hash,
    OLD.source_build_outbox_id
  ) THEN
    RAISE EXCEPTION
      'presentation_record % identity/context is immutable',
      OLD.presentation_id;
  END IF;

  -- A terminal delivery artifact is fully immutable.
  IF OLD.delivery_status IN ('DELIVERED', 'FAILED', 'SUPERSEDED') THEN
    RAISE EXCEPTION
      'presentation_record % is terminal and immutable',
      OLD.presentation_id;
  END IF;

  IF OLD.generation_status = 'PENDING' THEN
    IF NEW.generation_status = 'PENDING' THEN
      -- No partial generated result may appear while still PENDING.
      IF ROW(
        NEW.presentation_plan_id,
        NEW.presentation_plan_hash,
        NEW.accepted_build_lease_generation,
        NEW.rendered_payload,
        NEW.output_hash,
        NEW.generated_at
      )
      IS DISTINCT FROM
      ROW(
        OLD.presentation_plan_id,
        OLD.presentation_plan_hash,
        OLD.accepted_build_lease_generation,
        OLD.rendered_payload,
        OLD.output_hash,
        OLD.generated_at
      ) THEN
        RAISE EXCEPTION
          'pending presentation % cannot contain partial generated result',
          OLD.presentation_id;
      END IF;

      IF NEW.delivery_status NOT IN ('PENDING', 'SUPERSEDED') THEN
        RAISE EXCEPTION
          'pending generation % cannot be delivered/failed as delivery',
          OLD.presentation_id;
      END IF;

    ELSIF NEW.generation_status = 'GENERATED' THEN
      -- BUILD and DELIVER are separate tasks.
      IF NEW.delivery_status <> 'PENDING' THEN
        RAISE EXCEPTION
          'newly generated presentation % must await delivery',
          OLD.presentation_id;
      END IF;

      -- Row CHECK constraints require the generated result fields atomically.

    ELSIF NEW.generation_status = 'FAILED' THEN
      IF NEW.delivery_status <> 'FAILED' THEN
        RAISE EXCEPTION
          'terminal generation failure % must also mark delivery FAILED',
          OLD.presentation_id;
      END IF;

      -- Row CHECK constraints require generated-result fields to remain NULL.

    ELSE
      RAISE EXCEPTION
        'invalid generation transition for presentation %',
        OLD.presentation_id;
    END IF;

    RETURN NEW;
  END IF;

  IF OLD.generation_status = 'GENERATED' THEN
    -- Generated source artifact can never be replaced.
    IF ROW(
      NEW.presentation_plan_id,
      NEW.presentation_plan_hash,
      NEW.accepted_build_lease_generation,
      NEW.rendered_payload,
      NEW.output_hash,
      NEW.generation_status,
      NEW.generated_at
    )
    IS DISTINCT FROM
    ROW(
      OLD.presentation_plan_id,
      OLD.presentation_plan_hash,
      OLD.accepted_build_lease_generation,
      OLD.rendered_payload,
      OLD.output_hash,
      OLD.generation_status,
      OLD.generated_at
    ) THEN
      RAISE EXCEPTION
        'generated presentation % payload/plan is immutable',
        OLD.presentation_id;
    END IF;

    IF OLD.delivery_status = 'PENDING'
       AND NEW.delivery_status NOT IN (
         'PENDING',
         'DELIVERED',
         'FAILED',
         'SUPERSEDED'
       ) THEN
      RAISE EXCEPTION
        'invalid delivery transition for presentation %',
        OLD.presentation_id;
    END IF;

    RETURN NEW;
  END IF;

  -- generation_status FAILED is terminal and should already imply delivery FAILED;
  -- the terminal delivery guard above normally catches this state.
  RAISE EXCEPTION
    'presentation_record % is in an unsupported update state',
    OLD.presentation_id;
END;
$$;

CREATE TRIGGER trg_guard_presentation_record_update
BEFORE UPDATE ON engine.presentation_records
FOR EACH ROW
EXECUTE FUNCTION engine.guard_presentation_record_update();
```

## Fencing requirement

The trigger is not a lease/fencing substitute.

BUILD result acceptance still occurs in one transaction that:

1. locks/checks BUILD Outbox;
2. verifies expected `lease_generation`;
3. INSERTs immutable PresentationPlan;
4. UPDATEs PresentationRecord PENDING -> GENERATED;
5. INSERTs DELIVER task;
6. completes BUILD task;
7. COMMITs.

A stale generation cannot persist a result.

Delivery similarly verifies its Outbox fencing generation before PENDING -> terminal delivery status.

---

# 30. Outbox indexes

```sql
CREATE INDEX ix_outbox_ready
ON engine.outbox (
  available_after,
  outbox_id
)
WHERE status IN (
  'PENDING',
  'FAILED_RETRYABLE'
);

CREATE INDEX ix_outbox_processing_lease
ON engine.outbox (
  lease_until,
  outbox_id
)
WHERE status = 'PROCESSING';
```

Do not use `now()` in index predicates.

---

# 31. Supporting indexes

```sql
CREATE INDEX ix_action_submissions_window_slot
ON engine.action_submissions (
  session_id,
  window_id,
  action_slot_id,
  received_at
);

CREATE INDEX ix_resolution_attempts_window
ON engine.resolution_attempts (
  session_id,
  window_id,
  started_at
);

CREATE INDEX ix_canonical_transitions_resolution
ON engine.canonical_transitions(
  resolution_id
)
WHERE resolution_id IS NOT NULL;

CREATE INDEX ix_presentations_participant
ON engine.presentation_records (
  session_id,
  participant_ref,
  generated_at
);
```

No JSONB GIN index in initial schema.

---

# 32. Atomic canonical presentation handoff

If a canonical transition requires a new async player presentation, the TransitionPlan/application prepares before commit:

- PlayerInteractionView(s) derived from planned after-state + planned pending-state finalization;
- PlayerViewKey(s);
- stable PresentationId(s);
- PresentationGenerationContext(s);
- BUILD_PRESENTATION outbox payload/context/hashes.

Inside the canonical DB transaction, after canonical state/gate updates and before COMMIT:

1. INSERT immutable PlayerInteractionView if not already materialized by key;
2. INSERT PENDING PresentationRecord with stable PresentationId and generation context;
3. INSERT BUILD_PRESENTATION Outbox item referencing:
   - same Session;
   - PlayerViewKey;
   - PresentationId;
4. deferred Outbox -> PresentationRecord FK validates at COMMIT;
5. COMMIT together with events/state/required outbox.

Therefore:

> no BUILD_PRESENTATION task can commit without its exact source PlayerInteractionView and stable PresentationRecord identity.

If two logically identical PlayerViews are materialized, PK/hash identity makes insertion idempotent.

---

# 33. Pending-input-triggered async presentation

A pending ActionSubmission change may alter PlayerInteractionView without changing StreamRevision.

If the product requires async generated presentation for such a pending-only change:

- compute/materialize new PlayerInteractionView;
- allocate stable PresentationId;
- insert PENDING PresentationRecord;
- insert BUILD outbox task;

in the same pending-input transaction that updates the slot state.

Do not enqueue async generation before the source view is durable.

If UI can render pending-state changes deterministically without generation, no presentation task is required.

---

# 34. Outbox claim/fencing

Claim:

```sql
SELECT outbox_id
FROM engine.outbox
WHERE (
    status IN (
      'PENDING',
      'FAILED_RETRYABLE'
    )
    AND available_after <= current_timestamp
  )
  OR (
    status = 'PROCESSING'
    AND lease_until < current_timestamp
  )
ORDER BY available_after, outbox_id
FOR UPDATE SKIP LOCKED
LIMIT :batch_size;
```

Claim update:
- status PROCESSING;
- lease_owner;
- lease_until;
- lease_generation += 1;
- attempt_count += 1.

Commit before external work.

Fenced completion:

```sql
UPDATE engine.outbox
SET
  status = 'COMPLETED',
  completed_at = current_timestamp
WHERE outbox_id = :outbox_id
  AND status = 'PROCESSING'
  AND lease_owner = :worker
  AND lease_generation = :expected_generation;
```

Affected row count MUST be exactly 1.

Equivalent generation fencing is required for FAILED_RETRYABLE/FAILED_TERMINAL transitions.

---

# 35. Command processing fencing

Worker claim/takeover increments `processing_generation`.

Any renewal/finalization/canonical commit owned by the command uses:

```sql
... WHERE
  session_id = :session_id
  AND command_key = :command_key
  AND processing_status = 'PROCESSING'
  AND processing_owner = :worker
  AND processing_generation = :expected_generation
```

Affected row count MUST be exactly 1.

Stale workers cannot overwrite the current logical command result.

---

# 36. Canonical transaction lock order

Relative lock order remains:

```
1. session_runtime
2. command_processing
3. window_input_gates
4. window_slot_inputs ordered by action_slot_id
```

Never reverse the order inside a multi-row transaction.

No external provider call while holding database locks.

---

# 37. Session creation transaction

One transaction:

1. INSERT Session;
2. INSERT SessionGenesis;
3. construct/recheck revision-0 canonical state in application;
4. INSERT SessionRuntime revision 0;
5. COMMIT.

No hidden bootstrap event/randomization.

---

# 38. Submit/Replace transaction

1. establish semantic CommandKey/idempotency;
2. lock command record as required;
3. lock gate;
4. require ACCEPTING;
5. lock slot;
6. compare SlotInputRevision;
7. INSERT ActionSubmission;
8. UPDATE slot + revision;
9. optionally atomically materialize/enqueue pending-state presentation per section 33;
10. finalize command result;
11. COMMIT.

No StreamRevision change.

---

# 39. Freeze transaction

Lock:
`session_runtime -> command -> gate -> slots ordered`.

Verify:
- current canonical revision;
- active canonical Window;
- nonterminal Session;
- gate ACCEPTING.

Then:
- select/freeze submissions;
- verify content hashes;
- INSERT FrozenInputSet;
- gate -> FROZEN + owner;
- command -> PROCESSING generation/lease;
- COMMIT.

No StreamRevision change.

---

# 40. Canonical transition commit

Precompute/validate TransitionPlan and required post-commit presentation source views.

Transaction:

1. lock SessionRuntime;
2. verify revision/state hash;
3. lock/check command generation if relevant;
4. lock/check gate/freeze owner if relevant;
5. INSERT ResolutionRecord if applicable;
6. INSERT CanonicalTransitionRecord;
7. INSERT ordered SessionEvents;
8. UPDATE SessionRuntime after-state/hash/revision;
9. close/cancel gate/slots if required;
10. finalize command;
11. INSERT materialized PlayerInteractionViews required by async presentations;
12. INSERT PENDING PresentationRecords with stable IDs;
13. INSERT required Outbox rows;
14. COMMIT.

All-or-nothing.

---

# 41. Immutability privilege matrix

At minimum:

| Relation | runtime INSERT | runtime UPDATE | runtime DELETE | worker UPDATE |
|---|---:|---:|---:|---:|
| scenario_versions | no* | no* | no | no |
| sessions | yes | no | no | no |
| session_genesis | yes | no | no | no |
| session_runtime | yes | yes | no | no |
| session_events | yes | no | no | no |
| canonical_transitions | yes | no | no | no |
| resolution_records | yes | no | no | no |
| action_submissions | yes | no | no | no |
| frozen_input_sets | yes | no | no | no |
| player_interaction_views | yes | no | no | no |
| presentation_plans | no | no | no | insert only |
| presentation_records | yes pending | no normal generation | no | state-machine UPDATE |
| outbox | yes | no normal worker path | no | fenced UPDATE |

* Scenario publishing uses separate publisher role.

Command/gate/slot/attempt tables are mutable according to their lifecycle.

Concrete GRANT statements are generated only after deployment role mapping is known.

---

# 42. RLS/access stance

Internal engine tables do not depend on participant RLS for correctness.

Primary controls:

- internal schema not exposed to browser API;
- no client USAGE/grants;
- least-privilege backend roles.

If provider configuration exposes internal tables, enable RLS/default-deny before exposure.

Participant-specific RLS/views/functions are designed after ADR-015.

---

# 43. Cross-row invariants

Still enforced by application transaction + verifier rather than complex triggers:

- transition event count/range;
- exact batch-index sequence;
- event index metadata equality with canonical bytes;
- state hash equality;
- FrozenInputSet nested selected-entry hashes;
- canonical Window/gate relationship;
- terminal state/gate closure;
- presentation hash recomputation.

This is deliberate.

PostgreSQL CHECK constraints remain for row-local invariants.

---

# 44. Migration strategy

After schema acceptance:

```
db/migrations/
  <timestamp>_engine_schema.sql
  <timestamp>_engine_constraints.sql
  <timestamp>_engine_indexes.sql
  <timestamp>_engine_privileges.sql
```

Rules:

1. the canonical migration archive is provider-neutral under `db/migrations/`;
2. provider-specific deployment tooling may consume/mirror this archive but does not become the technical source of truth;
3. forward-only canonical migrations;
4. never edit an applied migration;
5. expand -> backfill/migrate -> contract for destructive evolution;
6. do not rewrite historical canonical event bytes for current schema evolution;
7. empty-initial-schema indexes can be built normally;
8. future large index builds may use dedicated `CREATE INDEX CONCURRENTLY` operational migrations;
9. provider/staging migration dry-run before production;
10. runtime app role is never migration/table-owner role.

---

# 44A. Deployment target prerequisite

The schema is intentionally provider-neutral.

Before executable migrations are considered implementation-ready, record and test against the selected deployment target:

- PostgreSQL major version;
- hosting/provider;
- migration owner role;
- runtime role mapping;
- worker role mapping;
- browser/API exposed-schema configuration;
- allowed extensions.

The current schema deliberately avoids depending on optional extensions.

If Supabase is later selected, its migration tooling may point at or consume the canonical `db/migrations/` files, but GitHub's provider-neutral SQL archive remains authoritative.

---

# 45. Restore hierarchy

Canonical state recovery needs:

```
scenario_versions
+ sessions/session_genesis
+ session_events
```

Then rebuild:
- SessionRuntime;
- derived indexes;
- secondary projections.

Resolver verification additionally restores selected input/evidence.

Outbox/Presence are not canonical restore requirements.

---

# 46. PostgreSQL-specific assumptions

The schema relies on widely supported PostgreSQL features:

- transactions;
- row locks;
- `FOR UPDATE SKIP LOCKED`;
- composite/deferred foreign keys;
- partial unique indexes;
- JSONB;
- column-level grants/default privileges.

No specialized extension is required by v0.2.

---

# 47. v0.4 acceptance tests

## DDL/constraints

1. create all relations/constraints/indexes on target PostgreSQL version;
2. composite Scenario version/hash FK works;
3. nullable current-submission composite FK works;
4. exact FrozenInputSet composite FK works;
5. deferred event->transition FK works;
6. partial active-gate unique index works;
7. publisher cannot update canonical Scenario bytes;
8. runtime cannot update/delete event/genesis/transition rows.

## Concurrency

9. two canonical commits from same revision -> one succeeds;
10. ReplaceAction vs Freeze linearizes;
11. two freeze commands -> one FrozenInputSet;
12. stale command generation cannot finalize;
13. stale outbox generation cannot finalize;
14. retryable command respects retry_after;
15. terminal transition atomically closes pending gate.

## Presentation

16. BUILD task cannot exist without source PlayerView/Pending PresentationRecord in canonical workflow;
17. PlayerViewKey recomputes from complete framing;
18. PlanHash recomputes;
19. generated result is accepted once;
20. stale BUILD worker output discarded;
21. delivery retry uses same PresentationId;
22. mutable external media URL alone cannot appear as historical asset identity.
23. PresentationRecord trigger rejects identity/context mutation while PENDING.
24. PENDING record cannot contain partial generated fields.
25. PENDING -> GENERATED writes all generated fields atomically and leaves delivery PENDING.
26. generation FAILED has no generated artifact and delivery FAILED.
27. generated payload/plan fields cannot change.
28. terminal delivery record cannot change.
29. delivered_at exists iff delivery_status=DELIVERED.
30. BUILD/DELIVER Outbox presentation_id must reference an existing PresentationRecord at transaction commit.

## Replay/integrity

31. Genesis + events rebuild state hash;
32. event metadata drift detected/repairable;
33. selected submission corruption detected;
34. event batch hash reproduces;
35. DB restore without derived rows rebuilds.

## Access/migrations

36. PUBLIC has no engine schema/table access;
37. future default privileges do not expose objects;
38. migration owner is distinct from runtime role;
39. runtime cannot UPDATE/DELETE immutable canonical relations;
40. worker cannot mutate canonical Session tables;
41. publisher cannot change canonical Scenario bytes/hash;
42. RLS absence on internal tables does not make them client-readable;
43. schema DDL passes on the explicitly selected deployment PostgreSQL major/provider before migration implementation is marked ready.

---

# 48. Acceptance gate

Red-team v0.4 before generating migration SQL.

If no material schema defect remains:
- mark schema accepted;
- generate first migration files;
- create repository package skeleton for contracts/domain/persistence;
- build DB integration tests around the accepted invariants.

If material defects remain:
- produce PostgreSQL Schema v0.3 first.


---

# 49. v0.3 change record

PostgreSQL Schema v0.3 adds:

- database-enforced PresentationRecord lifecycle immutability;
- deferred Outbox -> PresentationRecord referential integrity;
- tighter generation/delivery checks;
- provider-neutral canonical migration path;
- explicit deployment target/version prerequisite;
- privilege-matrix verification requirements.

No accepted architecture/domain/persistence decision changes.


---

# 50. v0.4 change record

PostgreSQL Schema v0.4 tightens only PresentationRecord persistence semantics:

- identity/context immutable from creation;
- no partial generated result while PENDING;
- atomic PENDING -> GENERATED;
- terminal generation-failure representation;
- exact delivered_at state equivalence;
- immutable generated artifact and terminal delivery rows.

No architecture/domain/persistence decision changes.
