# RED-TEAM — Persistence Data Model v0.2

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/persistence/PERSISTENCE_MODEL_v0.2.md`  
**Governing architecture/contracts/model:** ACCEPTED  
**Verdict:** MODIFY

## Executive verdict

v0.2 fixes the major v0.1 transaction/deadlock/recovery gaps.

The remaining defects are narrower but still implementation-significant:

1. leases lack fencing generations, so a stale worker can wake after takeover and attempt to finalize state;
2. selected ActionSubmission content is immutable by policy but lacks its own durable content hash in the persistence model;
3. FrozenInputSet hash identity does not explicitly include selected submission content hashes;
4. async presentation generation/delivery needs stable artifact identity and stale-worker result fencing, not only an outbox lease.

No accepted ADR, Domain Contract or Domain Model change is required.

Produce Persistence Model v0.3 with these corrections.

---

# 1. Gate cases

## 1. Canonical workers race

PASS.

`session_runtime` lock + revision/hash checks ensure only one canonical transition can commit.

## 2. Freeze vs ReplaceAction

PASS.

Global lock order and gate/slot CAS define a valid linearization.

## 3. Freeze vs terminal/admin transition

PASS at normal-domain level.

Freeze locks session_runtime before gate, so it verifies the canonical Window/base before FROZEN state commits.

## 4. Deadline vs explicit close

PASS.

First successful gate freeze owns the FrozenInputSet.

Competing command cannot create a second ordinary resolution.

## 5. Worker dies after freeze

PASS.

Frozen set + command processing state are durable and resumable.

## 6. Lease expires; original command worker wakes late

PARTIAL FAIL.

Revision checks prevent duplicate canonical commit, but the stale worker has no fencing generation proving it still owns the command-processing lease.

It could attempt to overwrite command status/result after another worker took over.

### Required correction

Add monotonically increasing `processing_generation`.

Every lease claim/takeover increments generation.

A worker must present the expected generation when renewing/finalizing/attaching a transition result.

Stale generation cannot mutate command processing state.

---

# 2. Outbox cases

## 7. Worker dies before side effect

PASS/PARTIAL.

Lease expiry permits reclaim.

## 8. Worker dies after side effect, before COMPLETED

Expected at-least-once duplicate risk.

v0.2 correctly requires idempotent consumers, but needs a more explicit fencing/result rule for tasks such as LLM presentation generation.

## 9. Stale outbox worker wakes after lease takeover

FAIL/PARTIAL.

`lease_owner` alone is insufficient if a worker identity is reused or an old process completes after a new claim.

### Required correction

Add monotonically increasing `lease_generation`.

All completion/result writes compare:

- outbox id;
- expected lease_generation.

Stale generation cannot mark item completed or persist authoritative task result.

External operations still require stable semantic idempotency/dedup keys where duplicates can escape the database.

---

# 3. Frozen input integrity

## 10. Selected ActionSubmission content is altered/corrupted

FAIL evidence gap.

ActionSubmission is immutable by policy, but v0.2 stores no content hash.

FrozenInputSet currently proves selected IDs/revisions, not the exact structured action content that resolver verification consumed.

### Required correction

Each ActionSubmission stores:

`submission_content_hash`

computed from canonicalized mechanical submission content, excluding operational `received_at`.

FrozenInputSet hash includes for each selected slot:

- action_slot_id;
- SlotInputRevision;
- SubmissionId;
- submission_content_hash.

Resolver verification can therefore prove that the selected persisted input matches the frozen input set.

---

# 4. Presentation generation/delivery

## 11. Presentation generation worker loses lease after LLM call

PARTIAL FAIL.

A stale worker could return a different nondeterministic output after another worker has taken over.

### Required correction

Every async presentation task uses stable identities:

- `presentation_task_key`;
- stable `presentation_id` or generation artifact key;
- PlayerViewKey;
- PresentationGenerationContext hash.

Only the currently fenced outbox generation may persist/finalize the task result.

If a stale worker returns later, its output is discarded.

## 12. Crash after presentation delivery

At-least-once delivery can duplicate a notification.

### Required correction

Player-facing delivery carries stable `presentation_id`.

Client/API delivery path treats repeated delivery of the same presentation_id as idempotent.

PresentationRecord is one immutable delivered artifact, not one row per notification attempt.

---

# 5. Hash/evidence cases

## 13. Event index metadata corruption

PASS.

Canonical bytes remain authority and metadata is rebuildable.

## 14. Current projection corruption

PASS.

Genesis + events rebuild current state and verify state hash.

## 15. PlayerInteractionView corruption

PASS.

v0.2 defines content-hash verification.

## 16. Mutable media URL

PASS.

Immutable/versioned PresentationAssetRef fixes historical delivery identity.

---

# 6. Retention and terminal cases

## 17. Selected submission deletion

PASS by policy, but content-hash correction strengthens verification.

Selected rows remain required for compatibility window unless FrozenInputSet/ResolutionRecord later becomes self-contained.

## 18. Terminal transition with FROZEN gate

PASS.

Same DB transaction closes/cancels pending gate and prevents further input.

---

# 7. Scale/recovery cases

## 19. Large JSONB current state

ACCEPTED RISK.

No evidence yet justifies normalization of canonical component/epistemic state into many mutable tables.

## 20. Event growth without partitioning

ACCEPTED RISK.

Measure before partitioning.

## 21. Full restore

PASS.

Bundle + Genesis + events reconstruct current canonical state.

## 22. Admin projection patch

PASS BY BOUNDARY.

Normal domain API forbids it; break-glass remains external audited procedure.

---

# 8. Postmortem of v0.2

## Root cause A — lease recovery was modeled without fencing

A lease answers "who should work now" but does not prove that an old worker is still authorized when it later wakes.

A monotonically increasing generation token is needed for safe finalization.

## Root cause B — immutable input policy lacked cryptographic/content evidence

Resolver verification needs to prove not only which SubmissionId was selected but which structured action bytes/values were resolved.

## Root cause C — async generation was treated like an ordinary idempotent side effect

LLM/media generation can be nondeterministic and costly.

The database must choose one stable artifact identity/result even when external calls duplicate.

---

# 9. Verdict

**PERSISTENCE_MODEL_v0.2 is NOT ready for acceptance.**

Create v0.3 with:

- command processing fencing generation;
- outbox lease generation/fencing;
- ActionSubmission content hash;
- FrozenInputSet hash including selected submission hashes;
- stable presentation task/artifact identity;
- fenced presentation result acceptance;
- presentation-id idempotent delivery.

No SQL migrations yet.
