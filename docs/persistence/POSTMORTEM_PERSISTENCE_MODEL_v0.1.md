# POSTMORTEM — Persistence Data Model v0.1

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/persistence/PERSISTENCE_MODEL_v0.1.md`  
**Governing architecture:** Architecture Baseline v0.3 — ACCEPTED  
**Governing contracts:** Domain Contracts v0.3 — ACCEPTED  
**Governing domain model:** Domain Model v0.2 — ACCEPTED  
**Verdict:** MODIFY

## Executive verdict

Persistence v0.1 has the correct overall shape:

- one PostgreSQL authority;
- immutable scenario/genesis/event history;
- one synchronous current Session projection;
- normalized command/pending-input concurrency state;
- immutable transition/resolution evidence;
- transactional outbox;
- optional materialized PlayerInteractionView for async presentation;
- no premature partitioning/broker/event-store product.

However, a transaction/concurrency red-team found several gaps that should be corrected before concrete SQL.

No accepted ADR, Domain Contract or Domain Model decision needs to change.

Required result:

> Produce `PERSISTENCE_MODEL_v0.2.md` with explicit lock hierarchy, resumable leases, stronger idempotency evidence, safer current-state representation and immutable presentation-asset semantics.

---

# 1. Red-team case results

| # | Case | v0.1 result | Finding |
|---|---|---|---|
| 1 | Two canonical transition workers race | PASS/PARTIAL | session_runtime lock/revision protects commit, but global lock order is not specified. |
| 2 | Freeze races ReplaceAction | PARTIAL FAIL | Gate CAS helps, but freeze must also validate/serialize against canonical runtime using consistent lock order. |
| 3 | Deadline races explicit finalize | PARTIAL | First gate freeze can win, but frozen owner is not stored, making resume/result ownership ambiguous. |
| 4 | Crash during Session creation | PASS | One creation transaction rolls back all bootstrap rows. |
| 5 | Crash after event append before projection/outbox | PASS | Single DB transaction rolls back partial canonical commit. |
| 6 | Duplicate event-batch retry | PASS | revision/transition uniqueness plus command idempotency protect canonical append. |
| 7 | Large session_runtime JSONB | ACCEPTED RISK | Whole-state projection is acceptable for MVP; measure before normalization. |
| 8 | Convenience columns drift from JSONB | FAIL RISK | lifecycle/logical_tick duplicate canonical state semantics without a strict derivation mechanism. |
| 9 | Event bytes disagree with extracted metadata columns | SAFE FOR REPLAY, BAD FOR QUERIES | Need explicit non-authoritative/index-column repair/validation policy. |
| 10 | Async PlayerView task after input changes | PASS | Materialized PlayerInteractionView + DeliveryGuard solves exact source-view recovery. |
| 11 | Outbox crash after external side effect | PARTIAL FAIL | at-least-once is acknowledged but processing lease/recovery semantics are incomplete. |
| 12 | Same outbox dedup key, changed payload | FAIL | uniqueness alone cannot distinguish exact retry from semantic-key misuse. |
| 13 | Scenario retired while old Session replays | PASS | immutable scenario bytes remain retained. |
| 14 | Nonselected submissions deleted | PASS WITH POLICY | state replay unaffected; forensic guarantees must state retention class. |
| 15 | Event table grows without partitions | PASS FOR MVP | monitor; no premature partitioning. |
| 16 | Admin patches current state | PASS BY POLICY | normal path forbidden; break-glass remains separate audited operation. |
| 17 | Restore loses derived indexes | PASS | rebuildable. |
| 18 | command_processing stuck PROCESSING | FAIL | no processing lease/claim recovery fields/protocol. |
| 19 | terminal Session while gate FROZEN | PASS/PARTIAL | commit closes gate, but persistence invariant should be explicit. |
| 20 | PresentationRecord points to mutable asset | FAIL | forensic narrative replay can drift if external asset URL changes. |

---

# 2. Findings

## PM-01 — Global database lock hierarchy is missing

### Problem

v0.1 defines:

- canonical commit: lock `session_runtime`, then later gate;
- freeze: lock gate, then slot rows;
- command processing: potentially lock command row at a different point.

A close/freeze/retry implementation could acquire these in inconsistent order and deadlock.

More importantly, freeze does not lock/validate the canonical Session row before committing a FROZEN gate.

### Correction

Define one lock hierarchy for transactions that touch multiple concurrency rows:

```
session_runtime
  -> command_processing
  -> window_input_gates
  -> window_slot_inputs (canonical slot-id order)
```

Rules:

- acquire only locks needed, but never reverse this relative order;
- Submit/Replace may use subset:
  `command_processing -> gate -> slot`;
- Freeze/close:
  `session_runtime -> command_processing -> gate -> all slots`;
- canonical resolution commit:
  `session_runtime -> command_processing -> gate`.

Freeze verifies under the runtime lock:

- Session current revision == Window base revision;
- canonical active Window id matches gate;
- Session is not terminal.

This closes stale-freeze ambiguity.

---

## PM-02 — Frozen input has no explicit command owner

### Problem

If DeadlineElapsed and explicit close/finalize race, the gate can become FROZEN but v0.1 stores only the FrozenInputSet hash.

A later retry cannot easily tell which logical command owns the in-flight resolution.

### Correction

Persist on WindowInputGate/FrozenInputSet:

- `freeze_owner_command_key`;
- optional `planned_transition_key` once known.

A competing command that finds the gate already FROZEN does not create a second resolution.

It returns/references the existing owner according to command semantics, or receives a controlled `WINDOW_ALREADY_FROZEN` result.

---

## PM-03 — command_processing requires a resumable lease

### Problem

PROCESSING can survive a worker crash indefinitely.

A new worker needs an explicit safe takeover rule.

### Correction

Add operational fields:

```
processing_owner
processing_lease_until
processing_attempt_count
```

Rules:

- short processing lease;
- takeover only after lease expiry;
- takeover keeps same CommandKey/FrozenInputSet logical operation;
- lease timestamps do not affect canonical mechanics;
- long resolver compute may renew lease.

The exact duration is operational configuration, not a domain rule.

---

## PM-04 — Outbox needs claim leases, not only PROCESSING status

### Problem

A worker can claim an item, crash and leave it PROCESSING forever.

### Correction

Add:

```
lease_owner
lease_until
last_error_code
```

Worker protocol:

1. short transaction selects ready items with row locks, skipping rows already locked;
2. marks lease/status and commits;
3. performs external work outside DB transaction;
4. completes item with matching lease owner;
5. expired leases are reclaimable.

PostgreSQL row locks are suitable for short claim transactions; do not hold a DB row lock while calling external providers.

---

## PM-05 — Outbox semantic idempotency is underspecified

### Problem

A UNIQUE deduplication key prevents duplicate inserts, but if the same key is accidentally reused with a different payload/context the conflict alone does not explain whether the existing task is an exact retry or a bug.

### Correction

Persist:

- `payload_hash`;
- `execution_context_hash`.

For a repeated dedup key:

- same hashes -> same logical task/idempotent reuse;
- different hashes -> `OUTBOX_IDEMPOTENCY_KEY_REUSE` and fail loudly.

This mirrors EngineCommand semantic idempotency.

---

## PM-06 — session_runtime duplicates canonical semantics in convenience columns

### Problem

v0.1 stores:

- lifecycle_state;
- logical_tick;

both in relational columns and CanonicalStateContent JSONB.

These are semantically canonical, so drift creates two current truths even though JSONB was declared authoritative.

### Correction

v0.2 should keep session_runtime minimal:

```
session_id
current_revision
state_hash
mechanics_manifest_hash
canonical_state_content_jsonb
updated_at
```

Do not persist duplicate lifecycle/logical-time columns initially.

If later query performance requires them, use a clearly derived projection/generated/index strategy whose values are rebuildable and explicitly non-authoritative.

---

## PM-07 — Extracted event query columns need a repair contract

### Problem

session_events stores canonical bytes plus family/code/schema/tick query columns.

Replay remains safe if query metadata drifts, but admin/research queries become wrong.

### Correction

Classify these fields explicitly as **event index metadata**.

At insert:

- derive them from the in-memory CanonicalEventContent used to create canonical bytes;
- store event content hash.

Provide validator/rebuild tooling:

`canonical_event_bytes -> parse -> recompute query metadata/hash -> compare/repair non-authoritative columns`.

Never rebuild canonical bytes from extracted columns.

A future schema may use generated/expression mechanisms where practical, but canonical bytes remain authority.

---

## PM-08 — Presentation media needs immutable/versioned asset identity

### Problem

Exact delivered text can be preserved while an external image/audio/video URL later changes.

That breaks exact player-experience forensic replay.

### Correction

PresentationRecord/plan asset references used for delivered gameplay must include immutable identity:

- content hash;
- immutable object/version key;
- media type;
- optional original generation metadata.

A mutable bare URL is insufficient as historical delivered evidence.

The media bytes can remain in R2/object storage; PostgreSQL stores immutable reference/hash.

---

## PM-09 — Freeze and canonical commit are intentionally two transactions; ownership must be explicit

### Problem

The model correctly avoids holding a DB transaction while the resolver computes, but that creates an intermediate durable state:

- canonical Window OPEN;
- input gate FROZEN;
- command PROCESSING.

This is valid only if no other normal canonical operation can steal the frontier.

### Correction

Persist explicit frozen ownership and ensure command routing treats FROZEN gate as an in-progress frontier operation.

Normal progression/system commands that would advance the Session while the active gate is FROZEN must:

- wait/return busy;
- or invoke an explicit recovery/interrupt policy.

Do not allow a second ordinary transition to advance the Session behind the frozen resolver.

---

## PM-10 — Selected ActionSubmission retention must be stronger than optional audit

### Problem

FrozenInputSet contains Submission IDs, not the full structured input.

If selected ActionSubmission rows are later deleted, resolver verification loses required input.

### Correction

Classify selected submissions as **mechanical verification evidence** for the supported replay window.

Only replaced/nonselected submissions are optional privacy/audit retention.

If future policy needs to delete selected structured submissions, FrozenInputSet/ResolutionRecord must contain a self-sufficient canonical copy before deletion.

---

## PM-11 — Interaction-view persistence needs content-hash verification

### Problem

`player_view_key` is intended as content identity, but v0.1 does not explicitly require persisted view content to rehash to the key.

### Correction

On insert/read validation:

```
PlayerViewKey == H(canonical(PlayerInteractionView content))
```

Same principle for PresentationPlan hash and delivered output hash.

This lets stale/corrupt interaction evidence be detected.

---

## PM-12 — Terminal transition must finalize pending-input storage atomically

### Problem

The Domain Model requires no accepting/open interaction state after terminal Session, but persistence v0.1 says only "finalize/cancel relevant gate if transition closes the Window."

### Correction

Make it a persistence invariant:

If committed after-state is terminal:

- gate cannot remain ACCEPTING/FROZEN;
- relevant pending slot rows become closed/archive state;
- no pending command can later create a gameplay submission for that Window.

This pending-store finalization occurs in the same DB transaction as the terminal canonical transition.

---

# 3. What v0.1 got right

Preserve:

1. one PostgreSQL system initially;
2. exact canonical bytes for scenario bundle/event content;
3. current SessionState as one JSONB projection;
4. normalized command/gate/slot concurrency records;
5. FrozenInputSet separate from canon;
6. generic CanonicalTransitionRecord;
7. immutable ResolutionRecord evidence;
8. selective PlayerInteractionView materialization;
9. transactional Outbox;
10. no initial table partitioning;
11. no historical snapshot requirement;
12. internal tables unavailable for direct browser mutation.

---

# 4. PostgreSQL-specific validation

Current PostgreSQL documentation supports row-level locking for writer/locker serialization; row locks are released at transaction end. This fits short claim/freeze/commit transactions, but external work should never occur while holding those locks.

PostgreSQL also supports partial indexes/unique indexes, making "one active gate per Session" a plausible DDL enforcement technique where the final status representation permits it.

JSONB and GIN indexes are available for targeted containment/path query patterns, but blanket indexing is unnecessary and would add write/storage cost.

These are implementation capabilities, not changes to the accepted domain architecture.

---

# 5. Verdict

**PERSISTENCE_MODEL_v0.1 is NOT ready for acceptance.**

No architecture/domain contract/model revision is required.

Produce `PERSISTENCE_MODEL_v0.2.md` with:

- global lock order;
- frozen resolution ownership;
- command leases;
- outbox leases and payload/context hashes;
- minimal nonduplicated session_runtime;
- event index-metadata repair semantics;
- immutable presentation asset refs;
- selected-submission retention guarantee;
- content-hash verification for interaction/presentation artifacts;
- terminal pending-input finalization invariant.

Do not create SQL migrations yet.
