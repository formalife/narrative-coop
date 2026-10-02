# POSTMORTEM — PostgreSQL Schema v0.4

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/persistence/POSTGRESQL_SCHEMA_v0.4.md`  
**Verdict:** MODIFY

## Executive verdict

v0.4 fixes the PresentationRecord state-machine defect.

A final relational/security review found two material issues:

1. the default-privilege statement for functions is not correct for PostgreSQL's default PUBLIC EXECUTE behavior;
2. Presentation/Outbox/Plan references prove that referenced rows exist, but not yet that they belong to the same Session/PlayerView/generation context.

These are schema-level integrity issues and should be corrected before acceptance.

No accepted higher-level decision changes.

---

# 1. PostgreSQL function default privileges

PostgreSQL grants EXECUTE on newly created functions/procedures to PUBLIC by default.

The v0.4 conceptual baseline used:

```
ALTER DEFAULT PRIVILEGES
  FOR ROLE <engine_owner_role>
  IN SCHEMA engine
  REVOKE ALL ON FUNCTIONS FROM PUBLIC;
```

Current PostgreSQL documentation notes that per-schema default privileges cannot remove a privilege granted by the global default in this way.

## Correction

Use a dedicated migration owner and set:

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE <engine_owner_role>
REVOKE EXECUTE ON FUNCTIONS FROM PUBLIC;
```

without `IN SCHEMA`.

Additionally, when creating a sensitive function, explicitly REVOKE PUBLIC EXECUTE in the same transaction as function creation.

For the PresentationRecord trigger function:

```sql
REVOKE ALL
ON FUNCTION engine.guard_presentation_record_update()
FROM PUBLIC;
```

The trigger still works through PostgreSQL trigger mechanics; the function is not a public application API.

Reference:
https://www.postgresql.org/docs/current/sql-createfunction.html
https://www.postgresql.org/docs/current/sql-alterdefaultprivileges.html

---

# 2. PresentationPlan context can mismatch PresentationRecord

## Problem

v0.4 has:

```
presentation_records.presentation_plan_id
  -> presentation_plans.presentation_plan_id
```

and separately stores:
- session_id;
- participant_ref;
- state_revision;
- player_view_key;
- presentation_plan_hash.

A buggy worker could attach a valid plan row from another audience/view/session while supplying an unrelated plan hash.

## Correction

On `presentation_plans` add a composite UNIQUE key:

```
(
  presentation_plan_id,
  session_id,
  participant_ref,
  state_revision,
  player_view_key,
  plan_hash
)
```

Replace the simple FK from PresentationRecord with:

```
(
  presentation_plan_id,
  session_id,
  participant_ref,
  state_revision,
  player_view_key,
  presentation_plan_hash
)
REFERENCES presentation_plans(...)
```

Because `presentation_plan_id/hash` are null while PENDING, default MATCH SIMPLE allows the pending row; once GENERATED all fields are non-null and must match one exact plan context.

---

# 3. Outbox presentation context can mismatch PresentationRecord

## Problem

v0.4 Outbox has:
- session_id;
- player_view_key;
- presentation_id;

and a FK only on presentation_id.

A task can theoretically reference:
- PresentationRecord A;
- PlayerViewKey from PresentationRecord B.

BUILD execution context can also drift from the PresentationRecord generation context.

## Correction

Add explicit Outbox field:

```
presentation_generation_context_hash bytea NULL
```

For BUILD/DELIVER presentation tasks require:

- presentation_id;
- player_view_key;
- presentation_generation_context_hash.

On PresentationRecord add composite UNIQUE:

```
(
  presentation_id,
  session_id,
  player_view_key,
  generation_context_hash
)
```

Add deferred composite FK:

```
(
  presentation_id,
  session_id,
  player_view_key,
  presentation_generation_context_hash
)
REFERENCES presentation_records(...)
```

For BUILD_PRESENTATION also require:

```
execution_context_hash = presentation_generation_context_hash
```

so the queued build context is exactly the pinned PresentationGenerationContext.

DELIVER_PRESENTATION may use a different delivery execution context but still carries the pinned generation-context hash for artifact identity.

---

# 4. PresentationRecord trigger review

PASS after v0.4.

It now enforces:

- immutable source/context;
- no partial generated result while PENDING;
- atomic GENERATED state;
- terminal generation failure;
- immutable generated artifact;
- terminal delivery immutability;
- delivered_at only for DELIVERED.

No additional trigger complexity is justified.

---

# 5. Migration bootstrap ordering

The migration archive should start with a bootstrap migration that:

1. creates/revokes schema access;
2. configures global default function EXECUTE revocation for the dedicated owner;
3. only then creates trigger/functions in later migrations.

Provider-role GRANT mapping may be applied after table/function creation.

---

# 6. Verdict

**POSTGRESQL_SCHEMA_v0.4 is NOT ready for acceptance.**

Produce v0.5 with:

- corrected function default privileges;
- explicit function PUBLIC revoke;
- composite Plan -> Presentation integrity;
- composite Outbox -> Presentation/View/GenerationContext integrity;
- migration bootstrap ordering.

No migration files yet.
