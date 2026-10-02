# POSTMORTEM — PostgreSQL Schema v0.3

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/persistence/POSTGRESQL_SCHEMA_v0.3.md`  
**Verdict:** MODIFY

## Executive verdict

v0.3 resolves the provider-neutral migration issue and adds useful DB-level PresentationRecord lifecycle protection.

The red-team found one remaining material flaw in that lifecycle trigger:

> while a PresentationRecord is still PENDING, the worker can modify immutable identity/generation-context fields or pre-populate generated output fields without changing generation_status.

That weakens the exact source-view/context evidence we introduced specifically for replay and stale-worker safety.

A few delivery timestamp/state checks should also be tightened.

No architecture/domain/persistence decision changes.

Produce PostgreSQL Schema v0.4 with a complete but narrow PresentationRecord state-machine guard.

---

# 1. Finding — immutable fields must be immutable from row creation

## Problem

The v0.3 trigger freezes generation fields only after OLD.generation_status = GENERATED.

While OLD.generation_status = PENDING it can still change:

- session_id;
- participant_ref;
- state_revision;
- player_view_key;
- generation_context;
- generation_context_hash;
- source_build_outbox_id.

Those define what the async task is authorized to generate and must never change after the PENDING record is atomically enqueued.

## Correction

The trigger must always reject changes to immutable identity/context fields:

```
session_id
participant_ref
state_revision
player_view_key
generation_context
generation_context_hash
source_build_outbox_id
```

from the first UPDATE onward.

---

# 2. Finding — generated-result fields can be populated while status stays PENDING

## Problem

v0.3 allows:

- presentation_plan_id/hash;
- accepted_build_lease_generation;
- rendered_payload;
- output_hash;
- generated_at;

to change while generation_status remains PENDING.

That creates hidden partial state and weakens crash/retry reasoning.

## Correction

If:

```
OLD.generation_status = PENDING
AND NEW.generation_status = PENDING
```

then generated-result fields must remain unchanged/null.

The only allowed non-generation mutation on a still-PENDING artifact is a delivery status transition to SUPERSEDED when a task is invalidated before generation.

---

# 3. Finding — PENDING -> GENERATED must be atomic

When generation_status changes:

```
PENDING -> GENERATED
```

the same row update must set:

- presentation_plan_id;
- presentation_plan_hash;
- accepted_build_lease_generation;
- rendered_payload;
- output_hash;
- generated_at.

The row CHECKs already require most of this; the trigger should make transition intent explicit.

---

# 4. Finding — terminal generation failure semantics

For a terminal generation failure:

```
generation_status:
  PENDING -> FAILED
```

the same update must:

- keep plan/output/generated fields null;
- set delivery_status = FAILED.

Retryable provider errors should stay represented in the Outbox task and leave PresentationRecord PENDING rather than prematurely marking the artifact terminal.

---

# 5. Finding — delivery timestamp constraints

`delivered_at` should be non-null only for DELIVERED.

Add:

```
delivery_status = 'DELIVERED'
<=> delivered_at IS NOT NULL
```

For:
- PENDING;
- FAILED;
- SUPERSEDED;

`delivered_at` remains null.

This prevents misleading timestamps.

---

# 6. Finding — delivery transition after GENERATED

For an OLD GENERATED + delivery PENDING artifact, the trigger may change only:

- delivery_status;
- delivered_at.

It may not change any source/generation/output field.

Allowed:
- PENDING -> DELIVERED
- PENDING -> FAILED
- PENDING -> SUPERSEDED

After any terminal delivery state, the row is fully immutable.

---

# 7. Outbox -> Presentation FK

PASS.

The deferred FK is appropriate and does not block later Outbox compaction because the reference direction is Outbox -> PresentationRecord.

---

# 8. Provider-neutral migration strategy

PASS.

The canonical path is now `db/migrations/`.

Target PostgreSQL major/provider remains a required pre-executable-migration deployment decision, not a schema-model defect.

---

# 9. Verdict

**POSTGRESQL_SCHEMA_v0.3 is NOT ready for acceptance.**

Produce v0.4 with:

- always-immutable Presentation identity/context fields;
- no hidden generated data while PENDING;
- atomic PENDING -> GENERATED requirements;
- terminal generation failure rules;
- exact delivery timestamp/state equivalence;
- post-generation update restricted to delivery fields.

No migration files yet.
