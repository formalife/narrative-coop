# RED-TEAM — PostgreSQL Schema v0.5

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/persistence/POSTGRESQL_SCHEMA_v0.5.md`  
**Governing architecture/contracts/domain/persistence:** ACCEPTED  
**Verdict:** READY FOR ACCEPTANCE

## Important scope distinction

This verdict means:

> the logical/concrete PostgreSQL schema design is ready to become the accepted schema baseline.

It does **not** mean:

> production migrations have already been validated on the final hosted PostgreSQL target.

Executable migrations remain gated on:
- target PostgreSQL major;
- provider;
- deployment role mapping;
- exposed-schema configuration.

---

# 1. Table/ownership mapping

PASS.

The schema preserves Persistence v0.3 ownership:

- immutable Scenario version bytes;
- immutable SessionGenesis;
- one mutable SessionRuntime frontier row;
- append-only SessionEvents;
- immutable CanonicalTransitions;
- normalized command/pending-input concurrency state;
- FrozenInputSet evidence;
- ResolutionRecords;
- materialized PlayerInteractionViews only when needed;
- Presentation workflow;
- transactional Outbox.

No new source of canonical world truth was introduced.

---

# 2. Session canonical concurrency

PASS.

`session_runtime` remains the canonical transition lock/CAS anchor.

`canonical_transitions` enforces:

- one transition per Session/base revision;
- one transition per resulting revision;
- positive event count;
- resulting_revision = base_revision + event_count.

`session_events` enforces unique StreamRevision and unique batch index per transition.

Cross-row event-range/count assertions remain deliberately in commit verifier logic.

---

# 3. Freeze/input concurrency

PASS.

- one nonterminal gate per Session via partial UNIQUE index;
- slot current submission can reference only a submission from the same Session/Window/Slot;
- FrozenInputSet is one-per-Window;
- ResolutionRecord references the exact FrozenInputSet hash;
- no wall-clock ordering defines action winner.

The accepted global lock order remains implementable.

---

# 4. Command idempotency/recovery

PASS.

Command table supports:

- semantic key/hash;
- processing lease;
- retry_after separate from lease;
- processing generation fencing;
- crash/takeover recovery.

The schema does not attempt to encode semantic command logic in triggers.

---

# 5. Event history integrity

PASS.

Authoritative event content is exact canonical bytes + hash.

Query metadata remains explicitly non-authoritative.

Deferred event -> transition FK allows transaction insertion flexibility.

Batch hash framing is already frozen by Persistence v0.3.

No DB trigger tries to reinterpret event semantics.

---

# 6. PlayerInteractionView integrity

PASS.

PlayerViewKey hash framing includes:

- Session;
- participant;
- revision;
- Window/input context;
- schema version;
- view content.

The DB stores those fields without duplicating a misleading partial content hash.

PlayerView rows are immutable and can be safely referenced by async tasks.

---

# 7. PresentationPlan -> PresentationRecord integrity

PASS.

The composite FK ensures a generated PresentationRecord can only reference a plan with the same:

- plan id;
- Session;
- participant;
- state revision;
- PlayerViewKey;
- plan hash.

A valid plan from the wrong audience/view can no longer be accidentally attached.

---

# 8. Outbox -> Presentation integrity

PASS.

The composite deferred FK binds each presentation task to the same:

- PresentationId;
- Session;
- PlayerViewKey;
- PresentationGenerationContext hash.

For BUILD_PRESENTATION:

`execution_context_hash = presentation_generation_context_hash`.

Therefore the queued build cannot silently use a context different from the immutable PresentationRecord context.

The deferred constraint supports atomic creation of PENDING PresentationRecord + BUILD task.

---

# 9. PresentationRecord state machine

PASS.

Row CHECKs + narrow trigger together enforce:

## PENDING

- immutable identity/context;
- no partial generated result;
- can become GENERATED/FAILED;
- may become SUPERSEDED before generation.

## GENERATED

- plan/output/generation fields complete;
- artifact immutable;
- only delivery state may advance.

## Generation FAILED

- no partial generated artifact;
- delivery FAILED.

## Delivery terminal

- DELIVERED/FAILED/SUPERSEDED are immutable.

`delivered_at` exists exactly for DELIVERED.

This is a persistence state-machine guard, not narrative/game logic.

---

# 10. Worker fencing

PASS.

Schema supports compare-and-set completion using:

- processing_generation;
- outbox lease_generation.

Presentation generation uses both:

- outbox generation fencing;
- PresentationRecord PENDING-only transition.

A stale external worker result cannot overwrite a newer accepted artifact.

---

# 11. Immutable canonical history privileges

PASS at schema-design level.

Logical DB roles separate:

- owner/migrator;
- publisher;
- runtime;
- worker;
- readonly ops.

Runtime does not receive UPDATE/DELETE on:
- Genesis;
- canonical events;
- transitions;
- resolution evidence;
- frozen selected input;
- immutable PlayerViews.

Worker does not receive canonical Session mutation privileges.

Column-level publisher grants preserve Scenario bundle immutability.

Actual GRANT statements require deployment role mapping and are part of migration implementation.

---

# 12. PUBLIC function privilege handling

PASS after v0.5 correction.

Current PostgreSQL grants PUBLIC EXECUTE on newly created functions by default.

v0.5 correctly requires:

```
ALTER DEFAULT PRIVILEGES
FOR ROLE <dedicated_engine_owner>
REVOKE EXECUTE ON FUNCTIONS FROM PUBLIC;
```

before function creation, without incorrectly limiting the revoke to one schema.

The specific trigger function is also explicitly revoked from PUBLIC in its creation transaction.

References:
- https://www.postgresql.org/docs/current/sql-createfunction.html
- https://www.postgresql.org/docs/current/sql-alterdefaultprivileges.html

---

# 13. RLS/access boundary

PASS for current architecture stage.

Participant RLS is intentionally not invented before ADR-015.

Internal protection is:

- engine schema not browser-exposed;
- no browser USAGE/table grants;
- backend role separation.

If provider configuration exposes the schema, RLS/default-deny must be enabled before such exposure.

Current PostgreSQL documents default deny when RLS is enabled and no applicable policy exists, while table owners typically bypass RLS.

---

# 14. Index strategy

PASS.

Initial schema contains only indexes justified by:

- uniqueness/invariants;
- worker claim/recovery;
- common Window/Presentation access.

No blanket JSONB GIN indexes.

No partitioning.

Both are measurement-driven future decisions.

PostgreSQL documentation confirms partial UNIQUE indexes can enforce uniqueness on qualifying row subsets.

---

# 15. Migration strategy

PASS after v0.5 correction.

Canonical archive is provider-neutral:

`db/migrations/`

Ordering begins with bootstrap/default privileges before trigger/function creation.

Actual executable migrations are not yet authorized because target PostgreSQL/provider/role mapping remain open.

This is intentional, not a schema defect.

---

# 16. Residual open items

These do not require another schema version before acceptance:

- exact UUID generation algorithm;
- target PostgreSQL major;
- hosting/provider;
- concrete deployment role names;
- ADR-015 participant identity/RLS;
- retention durations/cleanup jobs;
- JSONB query indexes;
- partitioning/snapshots.

They do block claiming the future migration set is production-ready where applicable.

---

# 17. Syntax/runtime validation limitation

The schema has been reviewed against current PostgreSQL documentation and accepted project invariants.

It has **not yet been executed against the final target PostgreSQL major/provider**, because that deployment target has not been selected.

Therefore the migration implementation gate must include actual DDL execution/integration tests on the selected target.

---

# 18. Verdict

**POSTGRESQL_SCHEMA_v0.5 is technically READY FOR ACCEPTANCE as the concrete schema baseline.**

No schema v0.6 is justified before selecting deployment target/provider.

After explicit acceptance, the correct next work is **not immediately production migrations**.

First freeze the implementation platform decisions required to materialize the schema:

1. target PostgreSQL/provider;
2. runtime deployment/API baseline;
3. DB role mapping;
4. ADR-015 guest identity/participation baseline;
5. exposed-schema/RLS strategy.

Then:

6. generate provider-compatible `db/migrations/`;
7. execute schema/integration tests;
8. create implementation monorepo skeleton;
9. implement contracts/domain/resolver/persistence adapters;
10. build the deterministic technical micro-scenario.
