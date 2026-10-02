# RED-TEAM — Persistence Data Model v0.3

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/persistence/PERSISTENCE_MODEL_v0.3.md`  
**Governing architecture/contracts/domain model:** ACCEPTED  
**Verdict:** READY FOR ACCEPTANCE

## 1. Concurrency and lock-order review

### Two canonical workers race

PASS.

Both must serialize on `session_runtime` and verify base revision/state hash.

Only one transition can advance the canonical frontier.

### Freeze races ReplaceAction

PASS.

The shared lock order is consistent:

```
session_runtime
-> command_processing
-> gate
-> slot(s)
```

Submit/Replace uses the compatible subset:

```
command_processing
-> gate
-> slot
```

No reverse-order cycle is introduced.

### Deadline races explicit close

PASS.

The first successful freeze owns:

- FrozenInputSet;
- freeze_owner_command_key.

The competing command cannot create a second ordinary resolution.

### Freeze races terminal/admin transition

PASS for normal domain paths.

Freeze locks/validates `session_runtime` before freezing the gate.

A later canonical commit must still verify base revision and frozen ownership.

---

# 2. Command worker recovery

### Crash after freeze

PASS.

Durable state is sufficient:

- Window canonically OPEN at base revision;
- gate FROZEN;
- FrozenInputSet immutable;
- owner CommandKey;
- CommandProcessingRecord PROCESSING.

### Lease expiry and takeover

PASS.

`processing_generation` is a fencing token.

Takeover increments generation.

A stale worker cannot:

- finalize command state;
- attach a transition;
- complete canonical commit as command owner.

Canonical revision CAS provides an additional independent safeguard.

---

# 3. Selected input integrity

### ActionSubmission persistence corruption

PASS at evidence level.

Each selected submission has:

- immutable structured content;
- `submission_content_hash`.

FrozenInputSet commits to:

- slot id;
- slot input revision;
- SubmissionId;
- submission content hash.

Resolver verification rehashes selected input and detects mismatch.

### Selected submission deletion

PASS under retention policy.

Selected structured submissions remain mechanical-verification evidence for the supported compatibility window.

Replaced/nonselected submissions may have shorter retention.

---

# 4. Canonical event/history integrity

### Duplicate transition retry

PASS.

Protected by:

- Session revision;
- transition key uniqueness;
- event revision uniqueness;
- CommandKey idempotency.

### Event query metadata drift

PASS.

Index metadata is explicitly non-authoritative and repairable from canonical bytes.

### Batch hash ambiguity

PASS.

v0.3 freezes:

```
H(canonical(OrderedList<CanonicalEventContent>))
```

rather than ambiguous byte/hash concatenation.

### Projection corruption

PASS.

`session_runtime` is rebuildable from:

- immutable Scenario Bundle;
- SessionGenesis;
- Session events.

State hash detects mismatch.

---

# 5. Outbox worker recovery

### Crash before external work

PASS.

Lease expires; another worker reclaims with a higher generation.

### Crash after external generation before result persistence

PASS with expected duplicate-cost risk.

A replacement worker may repeat the external LLM/media call, but only the worker holding the current `lease_generation` may persist the authoritative generated artifact.

The stale worker's result is discarded.

This preserves persisted/player-visible correctness even if an external provider call is duplicated.

### Stale outbox worker completes after takeover

PASS.

Lease generation fences result acceptance/completion.

### Same dedup key with different payload/context

PASS.

Payload/context hashes distinguish exact retry from semantic-key misuse.

---

# 6. Presentation pipeline

### BUILD worker returns nondeterministic output after losing lease

PASS.

Stable `presentation_id` + fenced BUILD outbox generation selects one persisted result.

### Crash after generated output, before delivery

PASS.

BUILD completion and creation of `DELIVER_PRESENTATION` task occur transactionally together.

### Crash after player delivery, before DB completion

PASS at protocol level.

Retry carries the same `presentation_id`.

Player-facing delivery treats the presentation identity idempotently.

### View becomes stale before delivery

PASS.

DeliveryGuard checks current applicability.

Stale record becomes SUPERSEDED instead of being delivered as current gameplay.

### External media changes later

PASS.

Delivered media references immutable object/version identity + content hash rather than mutable URL alone.

---

# 7. Current-state representation

### JSONB state becomes large

ACCEPTED MVP RISK.

The current projection is one rebuildable row and the workload is a short two-player Session.

No evidence yet justifies normalizing every entity/component/epistemic record into independently mutable canonical tables.

The architecture includes a clear measurement/revisit path.

### Duplicate convenience fields

PASS.

v0.3 avoids lifecycle/logical-time relational duplicates by default.

Future derived/generated fields must be explicitly non-authoritative.

---

# 8. Terminal/pending-input consistency

### Terminal ending while gate is FROZEN

PASS.

The terminal canonical transaction must:

- close/interrupt/cancel canonical Window;
- close/cancel pending gate;
- make slot input non-writable;
- commit all alongside final events/current state.

No terminal Session can leave an ACCEPTING/FROZEN normal gameplay gate.

---

# 9. Restore/recovery

### Lose session_runtime and derived indexes

PASS.

Reconstruct from:

```
Scenario Bundle
+ SessionGenesis
+ SessionEventStream
```

Then rebuild secondary indexes/projections.

### Old ScenarioVersion is RETIRED

PASS.

Retirement does not delete or mutate immutable bundle bytes.

### Lose completed outbox history

PASS for canonical reconstruction.

Outbox history is not canonical truth after reliability guarantees are satisfied.

---

# 10. PostgreSQL capability check

The model relies only on normal PostgreSQL capabilities:

- transactions;
- unique constraints/indexes;
- row-level locking;
- partial unique indexes as an optional enforcement technique;
- JSONB;
- short work-claim locking for outbox workers.

Current PostgreSQL documentation confirms:

- `FOR UPDATE` row locking serializes conflicting lockers/writers and releases locks at transaction end;
- `SKIP LOCKED` is available for skipping already-claimed rows;
- partial indexes can enforce uniqueness over a qualifying subset;
- JSONB supports targeted indexing through GIN operator classes.

References:
- https://www.postgresql.org/docs/current/explicit-locking.html
- https://www.postgresql.org/about/featurematrix/detail/skip-locked-clause/
- https://www.postgresql.org/docs/current/sql-createindex.html
- https://www.postgresql.org/docs/current/gin.html

None of these capabilities requires a broker or specialized event-store product.

---

# 11. Residual risks

## Whole-state write amplification

Measure before changing representation.

## Event/audit growth

Measure before partitioning.

## External duplicate provider calls

Fencing prevents duplicate persisted result, but cannot always prevent duplicate provider cost after ambiguous failure.

Use provider idempotency where available, but do not make canonical correctness depend on it.

## Audit/privacy retention

Exact retention periods remain a product/privacy decision.

## Direct DB break-glass operations

Need an operational runbook later; they are outside normal domain APIs.

---

# 12. Verdict

**PERSISTENCE_MODEL_v0.3 is technically READY FOR ACCEPTANCE.**

No remaining persistence-model defect justifies a v0.4 before concrete PostgreSQL schema design.

If explicitly accepted, next steps are:

1. mark Persistence Model v0.3 ACCEPTED;
2. create `POSTGRES_SCHEMA_v0.1.md` as DDL/schema proposal;
3. red-team constraints, transactions, indexes, RLS/access and migration strategy;
4. only after schema acceptance create actual migration files and implementation repository packages.

Until explicit acceptance, no migration SQL should be committed.
