# POSTMORTEM — PostgreSQL Schema v0.2

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/persistence/POSTGRESQL_SCHEMA_v0.2.md`  
**Governing architecture/contracts/domain/persistence:** ACCEPTED  
**Verdict:** MODIFY

## Executive verdict

v0.2 corrects the material v0.1 issues around atomic PlayerInteractionView persistence, hash framing, retry scheduling, evidence FKs, default privileges and least-privilege intent.

A second schema-level red-team found a smaller set of issues that should still be fixed before migrations:

1. the presentation worker privilege/state model does not yet enforce immutability of generated/delivered PresentationRecords at the database boundary;
2. presentation Outbox tasks carry a stable presentation_id but do not relationally prove that the PresentationRecord exists;
3. the migration path assumes Supabase before the runtime/provider stack has been accepted;
4. the exact PostgreSQL deployment major is not yet selected, so migration files should not be generated until the DDL is tested against the selected target;
5. delivery/generation status constraints can be tightened.

No accepted ADR, Domain Contract, Domain Model, or Persistence Model changes are required.

---

# 1. v0.2 improvements that pass

## Atomic source view + build task

PASS.

The canonical transaction now commits together:

- PlayerInteractionView;
- stable PENDING PresentationRecord;
- BUILD_PRESENTATION Outbox item.

This closes the missing-view crash window.

## PlayerView/Plan/Output hash framing

PASS.

The exact logical input objects are defined.

## FrozenInputSet -> ResolutionRecord FK

PASS.

Resolution evidence is bound to the exact frozen input hash.

## command retry_after

PASS.

Retry schedule no longer overloads lease expiry.

## removal of sessions.terminal_at

PASS.

No duplicate lifecycle truth.

## default privileges

PASS in principle.

Future objects are covered when created by the configured owner role.

---

# 2. Findings

## PG2-01 — PresentationRecord state immutability is not DB-enforced

### Problem

v0.2 says the worker has "state-machine UPDATE" privileges.

PostgreSQL GRANT cannot express:

> "you may update this row only while generation_status=PENDING, and after GENERATED you may only change delivery state."

A worker role with ordinary UPDATE could accidentally overwrite:

- generated payload;
- output hash;
- generation context;
- plan linkage;

after the artifact is already generated/delivered.

### Correction

Add one narrow BEFORE UPDATE trigger dedicated to PresentationRecord lifecycle.

The trigger enforces:

### Generation state

Allowed:
- PENDING -> GENERATED
- PENDING -> FAILED

Disallowed:
- GENERATED -> different generation state
- FAILED -> another generation state

Once OLD.generation_status = GENERATED, these are immutable:
- session/participant/view identity;
- plan id/hash;
- generation context/hash;
- source build id;
- accepted build generation;
- rendered payload;
- output hash;
- generated_at.

### Delivery state

Allowed:
- PENDING -> DELIVERED
- PENDING -> FAILED
- PENDING -> SUPERSEDED

Terminal:
- DELIVERED
- FAILED
- SUPERSEDED

Once delivery becomes terminal, delivery fields cannot be rewritten.

This trigger enforces artifact immutability; outbox lease-generation fencing still determines which worker is allowed to perform the PENDING -> GENERATED/DELIVERED transition.

A narrow lifecycle trigger is preferable to a generic "immutability framework."

---

## PG2-02 — Presentation outbox task should FK to stable PresentationRecord

### Problem

Outbox presentation tasks include `presentation_id`, but only PlayerViewKey has an FK.

A malformed task can point to a nonexistent presentation artifact.

### Correction

After both tables exist, add:

```sql
ALTER TABLE engine.outbox
ADD CONSTRAINT fk_outbox_presentation
FOREIGN KEY (presentation_id)
REFERENCES engine.presentation_records(presentation_id)
DEFERRABLE INITIALLY DEFERRED;
```

Nullable FK semantics allow non-presentation tasks to keep `presentation_id = NULL`.

The FK direction does not prevent deleting/compacting completed Outbox rows later.

The canonical transaction may insert Outbox and PresentationRecord in either order because the FK is deferred.

---

## PG2-03 — Delivery status constraints should be stricter

### Problem

Current CHECKs technically allow:

- generation_status=PENDING with delivery_status=FAILED.

A delivery attempt cannot fail before an output exists.

### Correction

Add:

```
delivery_status IN ('DELIVERED','FAILED')
=> generation_status = 'GENERATED'
```

SUPERSEDED may occur:
- while PENDING if DeliveryGuard/queue invalidates before generation;
- after GENERATED if stale before delivery.

---

## PG2-04 — Supabase migration path is premature

### Problem

v0.2 names:

`supabase/migrations/`

but the accepted architecture has PostgreSQL as the persistence baseline; Supabase remains a stack option rather than an accepted structural decision.

### Correction

Use provider-neutral canonical path:

```
db/migrations/
```

Provider-specific adapters/config can later point their migration runner to these files or mirror them if required.

Do not make the canonical schema archive provider-specific before backend/provider acceptance.

---

## PG2-05 — PostgreSQL target major must be selected before executable migrations

### Problem

The schema uses standard/widely supported PostgreSQL capabilities, but migration testing still needs one exact deployment major/provider environment.

Generated columns/version-specific improvements are not used, but DDL behavior, extension availability and provider role constraints must be tested on the actual target.

### Correction

Schema acceptance does NOT automatically authorize production-ready migration execution.

Before generating/finalizing executable migration files, record:

- target PostgreSQL major;
- provider/runtime if selected;
- role mapping;
- exposed-schema configuration;
- extension policy.

The schema itself remains provider-neutral.

If the provider decision remains open, migration drafting may begin in generic SQL but cannot be considered implementation-ready.

---

## PG2-06 — Runtime/publisher privilege matrix needs a migration-time verification test

### Problem

A documented privilege matrix is not enough if deployment roles/grants differ.

### Correction

Schema verification must include SQL assertions/tests that:

- runtime cannot UPDATE/DELETE immutable canonical tables;
- worker cannot mutate canonical tables;
- publisher cannot UPDATE bundle bytes/hash;
- browser roles have no schema/table access;
- owner/migrator is not the runtime credential.

---

# 3. PostgreSQL capability notes

The schema relies on ordinary PostgreSQL features.

PostgreSQL documents:

- row-level security default-deny when enabled with no applicable policies, while table owners typically bypass RLS;
- partial indexes and partial UNIQUE indexes;
- deferred foreign keys;
- row locks / SKIP LOCKED for queue-like work claiming.

These support the proposed design but do not replace application/domain validation.

---

# 4. Verdict

**POSTGRESQL_SCHEMA_v0.2 is NOT ready for acceptance.**

Produce v0.3 with:

- narrow PresentationRecord lifecycle trigger;
- deferred Outbox -> PresentationRecord FK;
- stricter generation/delivery checks;
- provider-neutral `db/migrations` path;
- explicit deployment-target prerequisite before executable migrations;
- privilege verification tests.

No actual migration files yet.
