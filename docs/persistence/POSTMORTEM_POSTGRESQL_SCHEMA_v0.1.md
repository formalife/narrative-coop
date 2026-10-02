# POSTMORTEM — PostgreSQL Schema v0.1

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/persistence/POSTGRESQL_SCHEMA_v0.1.md`  
**Governing architecture/contracts/domain/persistence model:** ACCEPTED  
**Verdict:** MODIFY

## Executive verdict

The v0.1 schema maps the accepted Persistence Model to PostgreSQL with the right overall shape, but the DDL review found several persistence-level gaps that should be corrected before any migration file exists.

No accepted ADR, Domain Contract, Domain Model, or Persistence Model decision needs to change.

The most important issues are:

1. async presentation tasks do not yet guarantee that the exact PlayerInteractionView they reference is durably materialized in the same atomic commit that creates the task;
2. append-only/immutable history is still protected mainly by application conventions rather than a database privilege matrix;
3. hash identity for PlayerInteractionView/PresentationPlan is ambiguous because relational fields and JSONB payload fields are split without one explicit canonical hash framing;
4. selected resolution evidence can be strengthened with a relational FK to the exact FrozenInputSet hash;
5. `sessions.terminal_at` duplicates canonical lifecycle semantics without being explicitly marked non-authoritative;
6. retryable commands lack a dedicated retry scheduling field;
7. default privileges for future objects in `engine` are not specified.

The correct action is a targeted PostgreSQL Schema v0.2.

---

# 1. What v0.1 got right

Preserve:

- dedicated internal `engine` schema;
- UUID runtime-instance ids and semantic text keys;
- `bytea` for hashes/canonical bytes;
- `jsonb` for typed scenario-defined structures;
- text + CHECK rather than PostgreSQL ENUM;
- one `session_runtime` concurrency anchor;
- composite FK from Session to exact Scenario version/hash;
- exact SessionGenesis manifest/hash/seed storage;
- partial UNIQUE active-gate invariant;
- composite FK from slot current_submission to matching Window/Slot submission;
- DEFERRABLE event -> transition FK;
- explicit StreamRevision and transition uniqueness;
- no initial JSONB GIN indexes;
- no initial partitioning/snapshot tables;
- forward-only migration philosophy;
- `FOR UPDATE SKIP LOCKED` outbox claim model.

---

# 2. Findings

## PG-01 — PlayerInteractionView and BUILD_PRESENTATION task are not atomically coupled

### Problem

Persistence v0.3 requires an async presentation worker to use the exact historical PlayerInteractionView identified by PlayerViewKey.

Schema v0.1 provides:

- `player_interaction_views`;
- `outbox`;

but the canonical transition transaction only inserts outbox items and does not require the referenced PlayerInteractionView to be inserted first/in the same transaction.

Failure mode:

1. canonical transition commits;
2. BUILD_PRESENTATION outbox item commits;
3. process dies before materializing the corresponding PlayerInteractionView;
4. worker wakes and cannot reconstruct the historical view if pending-input state has already changed.

### Correction

For any async presentation task created by a canonical transition:

- construct PlayerInteractionView from the planned after-state + planned pending-input finalization before commit;
- INSERT the materialized PlayerInteractionView in the same DB transaction;
- then INSERT BUILD_PRESENTATION outbox task referencing its PlayerViewKey.

For presentation tasks caused by noncanonical pending-input changes, materialize the view and enqueue the task in the same pending-input transaction if async generation is required.

Add a deferred/normal relational FK from outbox presentation payload is not practical because the PlayerViewKey is nested JSON.

Instead store explicit nullable columns on outbox for presentation tasks:

- `player_view_key`;
- `presentation_id`;

and FK `(session_id, player_view_key)` to `player_interaction_views`.

Non-presentation tasks leave these null.

---

## PG-02 — Immutable history should be enforced by database privileges

### Problem

v0.1 says application code does not expose UPDATE/DELETE operations for immutable tables, but a compromised/buggy runtime DB role could still execute them if granted broad DML.

Immutable relations include:

- `scenario_versions` canonical bytes/hash fields;
- `session_genesis`;
- `session_events`;
- `canonical_transitions`;
- committed `resolution_records`;
- delivered presentation payloads.

### Correction

Define logical database roles/privilege classes:

### engine_owner / migrator

Owns schema/DDL.

### engine_app

May:
- SELECT required internal tables;
- INSERT immutable gameplay records;
- UPDATE only mutable runtime/input/command rows.

Must NOT receive UPDATE/DELETE on:
- session_genesis;
- session_events;
- canonical_transitions;
- resolution_records.

### engine_worker

May:
- claim/update outbox;
- insert/update presentation workflow rows under fenced protocol;
- read materialized PlayerViews.

No direct canonical event/runtime mutation.

Actual provider role names may differ; migrations map deployment roles to this privilege matrix.

This turns immutability from convention into least privilege.

---

## PG-03 — Scenario-version immutability is mixed with retirement state

### Problem

`scenario_versions` contains both immutable canonical bundle data and mutable retirement metadata in the same row.

A role allowed to UPDATE status could accidentally update canonical bytes/hash too.

### Correction

Either:

A. use column-level UPDATE grants so publisher role can update only:
- status;
- retired_at;
- searchable_metadata;

or

B. split mutable publication lifecycle metadata.

Prefer A initially to avoid another table.

Also add:

```
CHECK (retired_at IS NULL OR retired_at >= published_at)
```

---

## PG-04 — PlayerViewKey hash framing is ambiguous

### Problem

Schema v0.1 stores:

- relational fields such as session_id, participant_ref, state_revision;
- `view_content` JSONB;
- `view_content_hash`;
- CHECK `player_view_key = view_content_hash`.

But the accepted contract defines PlayerViewKey as identity of the complete deterministic PlayerInteractionView.

If `view_content` excludes relational identity/context fields, this CHECK hashes only part of the view.

If `view_content` duplicates those fields, there are two copies that can drift.

### Correction

Define one exact hash framing:

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
player_view_key = H(canonical(PlayerViewHashInput))
```

Remove redundant `view_content_hash`.

The application/verifier computes/rechecks PlayerViewKey from relational columns + JSONB.

Same principle for PresentationPlan:

```
plan_hash = H(canonical(full plan identity/version/content))
```

Do not claim SQL alone verifies these canonical hashes.

---

## PG-05 — ResolutionRecord should bind to exact FrozenInputSet

### Problem

ResolutionRecord stores:

- session_id;
- window_id;
- frozen_input_set_hash;

but there is no relational constraint that the referenced Window actually has that exact FrozenInputSet.

### Correction

Add:

```
UNIQUE (session_id, window_id, input_set_hash)
```

to `frozen_input_sets`.

Then add composite FK from `resolution_records`:

```
(session_id, window_id, frozen_input_set_hash)
  -> frozen_input_sets(session_id, window_id, input_set_hash)
```

This is safe because both are mechanical verification evidence with aligned retention expectations.

---

## PG-06 — sessions.terminal_at risks a second lifecycle truth

### Problem

The accepted Domain Model says terminal Session lifecycle is canonical state/event history.

A mutable `sessions.terminal_at` column can be interpreted as authoritative or drift from canonical state.

### Correction

Remove `terminal_at` from the initial authoritative `sessions` table.

If needed later for operational querying/retention, add it to a secondary/admin projection or explicitly derived metadata table.

---

## PG-07 — Retryable command processing lacks retry scheduling

### Problem

`command_processing` has `FAILED_RETRYABLE` but no `available_after/retry_after`.

Using the expired processing lease as retry scheduling overloads lease semantics.

### Correction

Add:

```
retry_after timestamptz NULL
```

Rules:

- FAILED_RETRYABLE may set retry_after;
- a command retry worker checks retry_after separately from processing lease;
- PROCESSING uses processing_lease_until;
- retry timing is operational and cannot change canonical semantics.

Add an operational index for retryable commands if automatic retry is implemented.

---

## PG-08 — Future-object default privileges are not specified

### Problem

`REVOKE ALL ON ALL TABLES IN SCHEMA engine FROM PUBLIC` only applies to existing relations.

Future migrations should not depend on remembered manual revokes.

### Correction

The schema/migration baseline must include appropriate `ALTER DEFAULT PRIVILEGES` for the migration owner so future tables/sequences/functions do not gain unintended access.

Exact runtime role names remain deployment-specific.

---

## PG-09 — PresentationRecord exact delivered artifact needs explicit hash framing

### Problem

`exact_delivered_payload jsonb` is acceptable, but `output_hash` is not defined precisely.

### Correction

Define:

```
DeliveredPresentationArtifact = {
  presentation_id,
  audience/session identity,
  exact_delivered_payload,
  immutable PresentationAssetRefs
}
```

```
output_hash = H(canonical(DeliveredPresentationArtifact))
```

JSONB may store the structured payload because arrays/strings/values are preserved semantically; hash verification always uses the application canonicalizer.

If future output contains binary/non-JSON material, store immutable asset refs/hashes rather than raw external URLs.

---

## PG-10 — Presentation generation rows need stable preallocation semantics

### Problem

BUILD_PRESENTATION should have one stable `presentation_id` before the external LLM/media call, but schema v0.1 does not say when the PresentationRecord row is created.

### Correction

At task enqueue time:

- allocate stable `presentation_id`;
- store it on outbox;
- optionally create a PENDING PresentationRecord immediately, or allow fenced BUILD worker to create it with a uniqueness guarantee on presentation_id.

Preferred: create PENDING PresentationRecord together with PlayerView + BUILD outbox task.

Benefits:
- one stable artifact identity exists before external work;
- duplicate workers compete to fill/finalize the same row;
- DELIVER task can reference the stable presentation id.

---

## PG-11 — Outbox result fencing should be expressible as SQL compare-and-set

### Problem

The model says a worker must compare lease_generation, but schema review should make the write shape explicit.

### Correction

Completion/update operations use a predicate equivalent to:

```sql
UPDATE engine.outbox
SET ...
WHERE outbox_id = :id
  AND status = 'PROCESSING'
  AND lease_generation = :expected_generation
  AND lease_owner = :expected_owner;
```

Require affected-row count == 1.

Same for command-processing lease/finalization.

This is implementation SQL behavior, not a trigger.

---

## PG-12 — RLS is intentionally deferred, but internal schema isolation must be provider-tested

### Problem

"Not exposed" is a deployment assumption, not a PostgreSQL table property.

### Correction

Before production:

- verify provider API exposed-schema configuration;
- verify browser roles have no USAGE on `engine`;
- verify no direct table grants;
- if any internal schema is exposed, enable RLS/default-deny as defense-in-depth.

Identity-specific policies still wait for ADR-015.

Current PostgreSQL RLS uses default deny when enabled with no applicable policy, but table owners typically bypass RLS unless configured otherwise; privilege/schema isolation therefore remains primary for internal authoritative tables.

---

# 3. DDL behavior review

## Composite nullable FK

The `window_slot_inputs.current_submission_id` composite FK is acceptable with PostgreSQL's default MATCH SIMPLE behavior: when current_submission_id is NULL, the row does not require a submission target.

## Partial unique active-gate index

Appropriate.

PostgreSQL supports partial UNIQUE indexes over qualifying rows.

## Deferred event -> transition FK

Appropriate.

It preserves end-of-transaction referential integrity without forcing a specific insert order.

## Cross-row event-count/range checks

Leaving them to the atomic commit protocol + verifier is preferable to adding complex constraint triggers now.

PostgreSQL CHECK constraints are row-local and cannot directly express these multi-row invariants.

## Outbox ready indexes

Appropriate.

Do not use volatile `now()` in partial-index predicates.

---

# 4. Root causes

## Root cause A — async presentation persistence was designed table-by-table

The exact source PlayerInteractionView and task need to be treated as one atomic workflow boundary.

## Root cause B — least privilege was deferred too far

Application-level immutability is useful but not sufficient when PostgreSQL can enforce a stronger capability boundary cheaply.

## Root cause C — hashes were named before their complete framing objects were specified

A hash field is only meaningful when the exact canonical input object is defined.

---

# 5. Verdict

**POSTGRESQL_SCHEMA_v0.1 is NOT ready for acceptance.**

Produce `POSTGRESQL_SCHEMA_v0.2.md` with:

- atomic PlayerInteractionView + pending PresentationRecord + BUILD outbox workflow;
- explicit player/presentation hash framing;
- relational FrozenInputSet -> ResolutionRecord integrity;
- command retry_after;
- logical role/privilege matrix;
- default-privilege policy;
- removal of terminal_at from sessions;
- explicit fenced CAS update shapes;
- explicit delivered-output hash framing.

No migration files yet.
