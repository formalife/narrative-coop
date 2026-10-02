# POSTGRESQL SCHEMA v0.1

**Status:** PROPOSED  
**Date:** 2026-10-02  
**Governing architecture:** Architecture Baseline v0.3 — ACCEPTED  
**Governing contracts:** Domain Contracts v0.3 — ACCEPTED  
**Governing domain model:** Domain Model v0.2 — ACCEPTED  
**Governing persistence model:** Persistence Data Model v0.3 — ACCEPTED  
**Purpose:** concrete PostgreSQL schema/constraint/index/access/migration proposal before migration files or application code.

---

# 1. Scope and non-goals

This document proposes concrete PostgreSQL tables, column types, keys, constraints and indexes.

It does NOT yet create migration files.

It intentionally does not freeze:
- guest/account identity semantics (ADR-015 still PROPOSED);
- public/browser-facing RPC/API functions;
- final RLS participant policies;
- partitioning;
- historical snapshots;
- JSONB payload indexes without proven queries.

---

# 2. Database namespace and access baseline

Use a dedicated internal schema:

```sql
CREATE SCHEMA engine;
REVOKE ALL ON SCHEMA engine FROM PUBLIC;
```

All authoritative gameplay/persistence tables live under `engine`.

The application/backend role receives explicit privileges.

Browser/anon/authenticated client roles receive **no direct table privileges** on the internal schema.

Player access must occur through an authoritative server/API/RPC layer that returns PlayerInteractionView/presentation-safe data.

## Why not rely on RLS yet

PostgreSQL Row Level Security is available, but the actual participant/account identity binding is not accepted yet (ADR-015).

Therefore v0.1 does not invent user policies.

Future participant-facing SQL views/functions may use RLS once identity semantics are accepted.

Internal-authority protection is based first on:
- schema exposure boundary;
- SQL privileges;
- backend-only credentials.

---

# 3. SQL type conventions

## Runtime instance identifiers

Use PostgreSQL `uuid` for generated runtime identities:

- session_id;
- submission_id;
- window_id;
- resolution_id;
- attempt_id;
- presentation ids;
- outbox ids.

Generation algorithm is intentionally not frozen here.

Application code may generate UUIDs.

## Semantic/versioned keys

Use `text` for stable semantic identifiers:

- command_key;
- transition_key;
- action_slot_id;
- action_type_id;
- event family/code;
- schema/version refs;
- definition refs.

These are not assumed to be UUID-shaped.

## Hashes

Use `bytea`.

The exact hash algorithm is pinned in MechanicsVersionManifest, not encoded in SQL type.

## Canonical serialized content

Use `bytea`.

## Flexible validated structures

Use `jsonb`.

The application/compiler validates typed contracts before insertion.

## Canonical revisions/ticks

Use `bigint` with nonnegative checks where required.

## Operational timestamps

Use `timestamptz`.

Operational timestamps never define canonical gameplay ordering.

---

# 4. Status representation

Use `text + CHECK` rather than PostgreSQL ENUM for domain/application statuses in the initial schema.

Reason:
- status vocabularies may evolve;
- CHECK constraints are easier to replace in migrations than ENUM value lifecycle;
- application contracts remain the semantic source.

This does not mean arbitrary strings are allowed.

---

# 5. scenario_versions

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
  )
);
```

## Immutability

Application role may:
- INSERT new published versions;
- transition status PUBLISHED -> RETIRED.

It may not modify:
- bundle_hash;
- bundle_schema_version;
- canonical bytes.

Prefer enforcing immutable-column behavior in repository/application code initially; a narrow DB trigger can be added later if red-team justifies it.

---

# 6. sessions

```sql
CREATE TABLE engine.sessions (
  session_id uuid PRIMARY KEY,

  scenario_id uuid NOT NULL,
  scenario_version text NOT NULL,
  scenario_bundle_hash bytea NOT NULL,

  created_at timestamptz NOT NULL,
  terminal_at timestamptz NULL,

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

No account/owner FK is introduced before ADR-015.

---

# 7. session_genesis

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
    REFERENCES engine.sessions (session_id, scenario_bundle_hash)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  UNIQUE (session_id, mechanics_manifest_hash)
);
```

This row is immutable after creation.

---

# 8. session_runtime

```sql
CREATE TABLE engine.session_runtime (
  session_id uuid PRIMARY KEY,

  current_revision bigint NOT NULL
    CHECK (current_revision >= 0),

  state_hash bytea NOT NULL,
  mechanics_manifest_hash bytea NOT NULL,

  canonical_state_content jsonb NOT NULL
    CHECK (jsonb_typeof(canonical_state_content) = 'object'),

  updated_at timestamptz NOT NULL,

  FOREIGN KEY (session_id, mechanics_manifest_hash)
    REFERENCES engine.session_genesis (
      session_id,
      mechanics_manifest_hash
    )
    ON UPDATE RESTRICT
    ON DELETE RESTRICT
);
```

No duplicated lifecycle/logical-time convenience columns in v0.1.

---

# 9. command_processing

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
  )
);
```

No FK is created from `transition_key` to canonical transitions because command records and transition evidence have different retention/lifecycle semantics.

---

# 10. action_submissions

```sql
CREATE TABLE engine.action_submissions (
  submission_id uuid PRIMARY KEY,

  session_id uuid NOT NULL,
  window_id uuid NOT NULL,
  action_slot_id text NOT NULL,

  action_type_id text NOT NULL,

  parameters jsonb NOT NULL,
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

Selected submission integrity is verified by rehashing the structured mechanical content.

---

# 11. window_input_gates

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

Candidate database enforcement:

```sql
CREATE UNIQUE INDEX ux_one_nonterminal_gate_per_session
ON engine.window_input_gates(session_id)
WHERE gate_status IN ('ACCEPTING', 'FROZEN');
```

This directly encodes the accepted "one active action-input gate per Session" invariant.

---

# 12. window_slot_inputs

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

  FOREIGN KEY (session_id, window_id)
    REFERENCES engine.window_input_gates(session_id, window_id)
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

The second FK is valid only when `current_submission_id` is non-null under PostgreSQL's normal nullable FK semantics.

---

# 13. frozen_input_sets

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

  FOREIGN KEY (session_id, window_id)
    REFERENCES engine.window_input_gates(session_id, window_id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  UNIQUE (session_id, input_set_hash)
);
```

No FK is attempted for each SubmissionId nested inside `selected_entries`; verifier logic validates those structured refs and content hashes.

---

# 14. resolution_attempts

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

Operational/retention-managed.

---

# 15. resolution_records

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

  FOREIGN KEY (session_id)
    REFERENCES engine.sessions(session_id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  UNIQUE (session_id, window_id)
);
```

One committed resolution evidence record per resolved Window.

---

# 16. canonical_transitions

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

  CHECK (resulting_revision = base_revision + event_count),

  UNIQUE (session_id, base_revision),
  UNIQUE (session_id, resulting_revision)
);
```

The two unique revision constraints encode one canonical transition from each base revision and one transition ending at each resulting revision.

No hard FK to optional command/resolution evidence is required because those records may have different long-term retention policies.

---

# 17. session_events

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

  PRIMARY KEY (session_id, stream_revision),

  UNIQUE (
    session_id,
    transition_key,
    batch_index
  ),

  FOREIGN KEY (session_id, transition_key)
    REFERENCES engine.canonical_transitions(
      session_id,
      transition_key
    )
    DEFERRABLE INITIALLY DEFERRED
    ON UPDATE RESTRICT
    ON DELETE RESTRICT
);
```

The deferred FK allows event/transition insertion order inside the atomic commit transaction without weakening final referential integrity.

Cross-row assertions such as:
- exact event_count;
- contiguous batch_index range;
- revision interval matching transition;

remain commit-protocol/verifier responsibilities in v0.1 rather than complex database triggers.

---

# 18. player_interaction_views

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
    CHECK (jsonb_typeof(view_content) = 'object'),

  view_content_hash bytea NOT NULL,

  created_at timestamptz NOT NULL,

  PRIMARY KEY (session_id, player_view_key),

  FOREIGN KEY (session_id)
    REFERENCES engine.sessions(session_id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  CHECK (player_view_key = view_content_hash)
);
```

The application computes the canonical hash.

The SQL equality check only ensures the two persisted values agree; PostgreSQL does not compute the canonical JSON hash.

---

# 19. presentation_plans

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
    CHECK (jsonb_typeof(plan_content) = 'object'),

  plan_hash bytea NOT NULL,

  created_at timestamptz NOT NULL,

  FOREIGN KEY (session_id, player_view_key)
    REFERENCES engine.player_interaction_views(
      session_id,
      player_view_key
    )
    ON UPDATE RESTRICT
    ON DELETE RESTRICT
);
```

Plan hash validation is application/verifier responsibility.

---

# 20. presentation_records

```sql
CREATE TABLE engine.presentation_records (
  presentation_id uuid PRIMARY KEY,

  session_id uuid NOT NULL,
  participant_ref uuid NOT NULL,

  state_revision bigint NOT NULL
    CHECK (state_revision >= 0),

  player_view_key bytea NOT NULL,
  presentation_plan_id uuid NULL,
  presentation_plan_hash bytea NOT NULL,

  generation_context jsonb NOT NULL,
  generation_context_hash bytea NOT NULL,

  source_build_outbox_id uuid NOT NULL,
  accepted_build_lease_generation bigint NULL
    CHECK (
      accepted_build_lease_generation IS NULL
      OR accepted_build_lease_generation > 0
    ),

  exact_delivered_payload jsonb NULL,
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

  FOREIGN KEY (session_id, player_view_key)
    REFERENCES engine.player_interaction_views(
      session_id,
      player_view_key
    )
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  FOREIGN KEY (presentation_plan_id)
    REFERENCES engine.presentation_plans(presentation_plan_id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  CHECK (
    delivery_status <> 'DELIVERED'
    OR (
      exact_delivered_payload IS NOT NULL
      AND output_hash IS NOT NULL
      AND delivered_at IS NOT NULL
    )
  )
);
```

Media inside `exact_delivered_payload` references immutable PresentationAssetRef identity/hash, never mutable URL alone.

---

# 21. outbox

```sql
CREATE TABLE engine.outbox (
  outbox_id uuid PRIMARY KEY,

  session_id uuid NOT NULL,
  transition_key text NULL,

  task_type text NOT NULL,
  deduplication_key text NOT NULL,

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

  UNIQUE (
    session_id,
    task_type,
    deduplication_key
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

No FK from `transition_key` because some operational tasks can have distinct retention/recovery lifecycle and no hard reference is required for correctness.

---

# 22. Outbox indexes

Ready work:

```sql
CREATE INDEX ix_outbox_ready
ON engine.outbox (
  available_after,
  outbox_id
)
WHERE status IN ('PENDING', 'FAILED_RETRYABLE');
```

Expired PROCESSING leases:

```sql
CREATE INDEX ix_outbox_processing_lease
ON engine.outbox (
  lease_until,
  outbox_id
)
WHERE status = 'PROCESSING';
```

Do **not** put `now()` inside a partial-index predicate.

Worker query compares `available_after/lease_until` to current time at execution.

---

# 23. Supporting indexes

```sql
CREATE INDEX ix_action_submissions_window_slot
ON engine.action_submissions (
  session_id,
  window_id,
  action_slot_id,
  received_at
);

CREATE INDEX ix_command_processing_status_lease
ON engine.command_processing (
  processing_status,
  processing_lease_until
);

CREATE INDEX ix_resolution_attempts_window
ON engine.resolution_attempts (
  session_id,
  window_id,
  started_at
);

CREATE INDEX ix_canonical_transitions_resolution
ON engine.canonical_transitions(resolution_id)
WHERE resolution_id IS NOT NULL;

CREATE INDEX ix_presentations_participant
ON engine.presentation_records (
  session_id,
  participant_ref,
  generated_at
);
```

No GIN JSONB indexes in initial DDL.

Add only when a real query pattern needs one.

---

# 24. Schema privileges

Initial principle:

```sql
REVOKE ALL ON ALL TABLES IN SCHEMA engine FROM PUBLIC;
REVOKE ALL ON ALL SEQUENCES IN SCHEMA engine FROM PUBLIC;
```

Concrete backend/migration/worker roles depend on deployment provider.

Required separation:

- migration owner: DDL capability;
- application authoritative role: required DML;
- worker role: only required outbox/presentation/operational DML;
- browser-facing roles: no direct engine table privileges.

Do not make the application runtime the migration/table-owner role in production.

---

# 25. RLS policy

No participant RLS policies are frozen in v0.1 because ADR-015 is still open.

For internal engine tables:

- direct client privileges are absent;
- internal schema should not be exposed as a generic client REST surface.

If provider configuration exposes the schema regardless, enable RLS with default-deny as defense-in-depth before production exposure.

Future user/player-facing views/RPC tables may use RLS after identity semantics are accepted.

---

# 26. Transaction protocol — Session creation

Single transaction:

1. INSERT Session;
2. INSERT SessionGenesis;
3. construct/verify revision-0 state in application;
4. INSERT SessionRuntime revision 0;
5. COMMIT.

Failure anywhere rolls back bootstrap.

No revision-0 event row.

---

# 27. Transaction protocol — Submit/Replace Action

Required logical lock sequence:

1. claim/check `command_processing`;
2. lock `window_input_gates` row FOR UPDATE;
3. require ACCEPTING;
4. lock target `window_slot_inputs` row FOR UPDATE;
5. compare expected SlotInputRevision;
6. INSERT immutable ActionSubmission;
7. UPDATE slot current submission + revision;
8. finalize CommandResult;
9. COMMIT.

No canonical StreamRevision change.

---

# 28. Transaction protocol — Freeze

Lock sequence:

1. `session_runtime FOR UPDATE`;
2. command row;
3. gate row;
4. all slot rows ordered by `action_slot_id`.

Validate:

- runtime revision == gate base revision;
- canonical current-state Window id == gate window;
- Session is not terminal;
- gate ACCEPTING.

Then:

- choose selected submissions/defaults;
- verify each Submission content hash;
- INSERT FrozenInputSet;
- gate -> FROZEN;
- set freeze owner;
- command -> PROCESSING with incremented generation/lease;
- COMMIT.

No canonical revision change.

---

# 29. Transaction protocol — Canonical transition commit

TransitionPlan is computed/validated before transaction.

Lock sequence:

1. `session_runtime FOR UPDATE`;
2. command row if command-owned;
3. gate row if relevant.

Validate:

- current revision/hash;
- expected command processing generation;
- frozen owner/hash;
- terminal/gate invariants.

Then:

1. INSERT ResolutionRecord if applicable;
2. INSERT CanonicalTransitionRecord;
3. INSERT ordered SessionEvent rows;
4. UPDATE SessionRuntime after-state/hash/revision;
5. close/cancel pending gate/slots if relevant;
6. terminalize command result;
7. INSERT required Outbox items;
8. COMMIT.

The deferred event->transition FK is checked at transaction end.

---

# 30. Transaction protocol — Outbox claim

Claim transaction:

```sql
SELECT outbox_id
FROM engine.outbox
WHERE (
    status IN ('PENDING', 'FAILED_RETRYABLE')
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

Then update selected rows:

- status PROCESSING;
- lease_owner;
- lease_until;
- lease_generation = lease_generation + 1;
- attempt_count = attempt_count + 1.

Commit quickly.

External processing occurs after commit.

Result/finalization transaction must compare expected `lease_generation`.

---

# 31. Immutable data enforcement strategy

v0.1 does NOT add generic "prevent UPDATE" triggers everywhere.

Instead:

- application repository exposes no update/delete operation for immutable relations;
- privileges can distinguish writers where practical;
- integration tests assert immutability paths;
- production audit/monitoring detects unexpected mutation;
- break-glass mutation is not a normal app capability.

Potential future:
narrow immutability triggers can be added if the threat/operational model justifies their complexity.

---

# 32. Cross-row invariants not delegated to CHECK

PostgreSQL CHECK constraints are row-local.

The following remain application transaction + replay/verifier assertions:

- transition owns exactly `event_count` rows;
- event revisions exactly cover transition interval;
- batch_index is exactly `0..K-1`;
- canonical event bytes match extracted index metadata;
- state hash matches canonical state bytes/value;
- FrozenInputSet nested refs/hash match submissions;
- canonical current-state Window matches gate Window;
- terminal canonical state matches pending gate closure.

Avoid complex trigger systems before demonstrated need.

---

# 33. Migration strategy

After schema acceptance, migrations should be forward-only and version-controlled.

Initial plan:

```
supabase/migrations/
  <timestamp>_engine_schema.sql
  <timestamp>_engine_constraints.sql
  <timestamp>_engine_indexes.sql
  <timestamp>_engine_privileges.sql
```

Exact split can change.

## Rules

1. migration files are canonical GitHub artifacts;
2. local/staging migration test before production;
3. production migrations are never edited after application;
4. destructive changes use expand -> migrate/backfill -> contract;
5. event canonical bytes are never rewritten merely because current schema changes;
6. large future index builds may use `CREATE INDEX CONCURRENTLY` in a dedicated nontransactional migration/runbook when needed;
7. initial empty-database indexes do not need concurrent creation;
8. future large-table constraints may use staged validation patterns rather than long blocking rewrites where supported.

---

# 34. Rollback philosophy

Do not depend on reverse/down migrations for canonical production history.

For a faulty migration:

- fix forward with a new migration;
- restore from backup only for catastrophic operational failure;
- never "rollback" by deleting/reinterpreting committed gameplay events.

Application releases must remain compatible with the database migration sequencing used during deploy.

---

# 35. Schema verification tests required before migration acceptance

## Structural

1. all PK/FK/UNIQUE/CHECK constraints create on supported PostgreSQL;
2. partial active-gate unique index works;
3. nullable composite FK for current submission behaves as intended;
4. deferred event->transition FK works in both insertion orders;
5. internal schema has no PUBLIC table privilege.

## Concurrency

6. two transition transactions cannot both commit from same base revision;
7. Submit/Replace vs Freeze linearizes;
8. two Freeze commands produce one FrozenInputSet;
9. processing-generation stale worker cannot finalize command;
10. outbox stale generation cannot finalize work;
11. terminal transition leaves no active gate.

## Replay/integrity

12. Genesis + events reconstruct current state hash;
13. event metadata corruption is detectable/repairable;
14. selected submission content corruption is detected by FrozenInputSet verification;
15. batch hash reproduces;
16. view/plan/presentation hashes verify.

## Recovery

17. crash after freeze resumes;
18. crash inside canonical transaction rolls back all;
19. crash after outbox external side effect leads to idempotent retry;
20. DB restore with events but without derived data rebuilds successfully.

---

# 36. Open schema decisions

Still PROPOSED/not frozen:

- UUID generation algorithm;
- exact database/application role names;
- Supabase exposed-schema configuration;
- ADR-015 player identity and RLS policies;
- hash algorithm byte length;
- any JSONB GIN/path indexes;
- partitioning;
- snapshot tables;
- retention cleanup jobs;
- whether optional debug/audit relations share PostgreSQL or object storage at scale.

---

# 37. Acceptance gate

Before POSTGRESQL_SCHEMA v0.1 can be accepted:

1. execute a DDL-level logical review;
2. red-team PostgreSQL constraint behavior;
3. check migration/privilege/RLS assumptions;
4. find any conflict with Persistence v0.3;
5. produce postmortem and v0.2 if required.

No actual migration file should be committed before this gate passes.
