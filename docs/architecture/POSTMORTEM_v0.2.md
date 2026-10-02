# POSTMORTEM — Architecture Baseline v0.2

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/architecture/ARCHITECTURE_BASELINE_v0.2.md`  
**Verdict:** MODIFY — v0.2 should not be accepted as the final Phase-0 baseline.

## Executive verdict

v0.2 is directionally strong and fixes most of the architectural weaknesses found in v0.1. It establishes the correct high-level center of gravity:

- one canonical Session timeline;
- server-authoritative action expansion;
- deterministic causal merge;
- semantic events separated from low-level state mutation;
- explicit epistemics;
- immutable compiled scenario bundles;
- selective Event Sourcing;
- logical CQRS;
- versioning and replay;
- LLM outside canonical state transitions.

However, a second-pass review found several structural gaps that should be corrected before architecture acceptance and before concrete schema/code work.

The most important defects are not implementation details. They affect determinism, replay semantics, authority boundaries and session progression.

The correct action is therefore:

> Keep v0.2 as an important historical proposal, but replace it as the active working baseline with v0.3.

---

# What v0.2 got right

## 1. Server authority was materially improved

The split:

`ActionSubmission → ActionDefinition → ExpandedAction`

is correct.

It prevents the browser/client from authoring claims, permissions, preconditions, resource costs or canonical effects.

This should be preserved.

## 2. Interaction Graph is better than a simple Conflict Graph

The move from conflict-only thinking to typed interactions correctly supports:

- contention;
- exclusion;
- interference;
- dependency;
- complementarity;
- order sensitivity.

This is important because asymmetric cooperation is not primarily about "who wins" a conflict. Many interesting consequences come from actions that combine, enable or interfere indirectly.

## 3. Semantic Domain Event + Mutation IR is the right split

Microsoft's Event Sourcing guidance explicitly recommends events that capture business/domain intent rather than merely recording resulting state changes.

Source:
https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing

The v0.2 split avoids both extremes:

- one bespoke reducer/event implementation for every narrative verb;
- a meaningless generic state-change log.

This remains a core architecture decision.

## 4. Admission versus ActionOutcome is correct

A malformed/unauthorized/stale submission is not the same thing as a valid in-world attempt that fails.

The v0.2 separation improves:

- canonical history;
- security;
- analytics;
- debugging;
- narrative realization.

Preserve it.

## 5. Named deterministic randomness is correct

Stable draw keys are materially safer than consuming a sequential random stream where inserting a new draw changes all later results.

Preserve this design.

## 6. Selective Event Sourcing remains justified

The product explicitly needs:

- causal reconstruction;
- historical replay;
- debugging;
- recap;
- explanation of why the world reached a state.

Those benefits justify Event Sourcing for canonical Session history, but not for every subsystem.

Microsoft explicitly warns that Event Sourcing introduces substantial complexity in concurrency, schema evolution, querying and projections and recommends selective adoption.

Source:
https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing

Preserve selective scope.

---

# What v0.2 got wrong or left unresolved

## P2-01 — Missing authoritative Command layer

### Problem

v0.2 models ActionSubmission well, but the engine has more authoritative inputs than player actions:

- player submits/replaces action;
- wall-clock deadline fires;
- window closes;
- player disconnect policy triggers;
- session resumes;
- scheduled logical consequence becomes due;
- automatic scenario transition runs;
- administrative recovery command may be required.

Without a common command boundary, idempotency and causation semantics become inconsistent.

### Root cause

The v0.2 review focused on the resolver's inner loop rather than the complete lifecycle:

`external input → authoritative command → canonical transition → post-commit effects`

### Correction

Introduce an application-level `EngineCommand` envelope.

Commands are requests/inputs, not canonical facts.

Every authoritative command has:

- `command_id` for idempotency;
- Session;
- command type;
- authenticated/system principal set server-side;
- optional expected revision/window;
- causation/correlation metadata;
- payload;
- operational receipt time.

Player `ActionSubmission` is persisted input created by a Submit/Replace Action command.

System commands use the same idempotency discipline.

Microsoft CQRS guidance explicitly treats commands as task/business-intent requests rather than low-level data updates.

Source:
https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs

---

## P2-02 — ResolutionWindow snapshot semantics are incomplete

### Problem

v0.2 says simultaneous actions resolve against one base snapshot, but it does not define what happens if canonical state changes while the window is collecting submissions.

Examples:

- a logical scheduled event becomes due;
- another automatic transition runs;
- timeout fires;
- a participant-related event occurs;
- a different canonical writer advances the Session.

If the window's base state can drift, "same base revision" becomes ambiguous.

### Root cause

Optimistic concurrency was used as the main safeguard instead of first simplifying the Session's canonical progression model.

### Correction

Introduce the **Single Canonical Frontier** invariant:

> At most one canonical state-changing progression/resolution operation is active for a Session at a time.

An open simultaneous ResolutionWindow may collect noncanonical/pending ActionSubmissions while canonical gameplay state remains fixed at its opening revision.

World-changing scheduler/progression transitions do not independently commit "through" the open window.

A wall-clock deadline issues a command that closes/resolves/interrupts the current window according to policy.

Logical due effects are processed at deterministic progression boundaries or through an explicit interrupt policy.

Optimistic expected-revision concurrency remains as a safety check, not as normal gameplay arbitration.

---

## P2-03 — Event record mixes deterministic content with operational metadata

### Problem

v0.2 puts fields such as `recorded_at` in the DomainEvent envelope and then says the event batch is hashed/reproduced.

If replay generates:

- a different wall-clock timestamp;
- a new random UUID event id;
- a database-specific record identifier;

the semantic result can be identical but the batch hash differs.

### Root cause

The architecture correctly separated logical time from wall clock but did not carry that separation through to event identity/hashing.

### Correction

Split:

### CanonicalEventContent
Deterministic and hashable.

### EventRecordMetadata
Operational persistence metadata, explicitly excluded from semantic/replay hash.

Canonical event identity must either be:

- deterministically derivable from Session/batch/index; or
- treated as stored input and excluded from recomputed semantic equivalence.

Preferred baseline: deterministic `event_key = (batch_sequence, batch_index)` within a Session, with an optional database surrogate id outside canonical semantics.

`recorded_at` remains stored but outside the deterministic batch hash.

CloudEvents similarly distinguishes event context from payload, but its generic event identity/time rules are not sufficient to define our deterministic replay contract.

Reference:
https://github.com/cloudevents/spec/blob/main/cloudevents/spec.md

---

## P2-04 — Numeric determinism is not defined

### Problem

v0.2 uses concepts such as:

- resource quantity;
- relationship metrics;
- number.adjust;
- probabilities;
- state hashing;

without defining canonical numeric semantics.

Using ambient JavaScript/JSON floating-point arithmetic can create:

- rounding drift;
- unstable equality;
- unclear unit conversion;
- cross-version replay problems.

RFC 8785's JSON Canonicalization Scheme uses the JSON/I-JSON number domain and explicitly notes IEEE-754 constraints; applications needing higher precision/large integers should encode them differently.

Source:
https://www.rfc-editor.org/rfc/rfc8785.html

### Correction

Canonical mechanics must not depend on unconstrained floating point.

Default canonical numeric forms:

- integer;
- fixed-point/scaled integer;
- explicit unit;
- string-encoded integer where values exceed safe JSON integer range.

No NaN or Infinity.

Probabilities/metrics should use declared fixed scale where exact replay matters.

Logical time should likewise use deterministic integer/tick representation internally, with authored calendar display as a projection.

---

## P2-05 — Epistemic "stance enum" is semantically wrong

### Problem

v0.2 proposes one stance with values:

- KNOWN;
- BELIEVED;
- SUSPECTED;
- DISBELIEVED;
- UNKNOWN.

This conflates different concepts.

`DISBELIEVED(P)` is not the same as `BELIEVED(not P)`.

`UNKNOWN` is normally the absence of an epistemic record, not a positive mental state.

Knowledge, belief and suspicion also have different provenance/inference requirements.

### Root cause

The desire for a compact generic representation compressed distinctions that the founding requirements explicitly said must remain separate.

### Correction

Use structured epistemic records:

- `KnowledgeRecord`;
- `BeliefRecord`;
- `SuspicionRecord`;
- `ObservationRecord`;
- `CommunicationClaim`.

All reference a normalized Proposition with explicit value/polarity.

Unknown = no applicable positive record.

If a character believes a proposition is false, represent the proposition/value explicitly rather than a generic "disbelieved" stance.

Do not implement automatic logical closure. Knowledge is granted only by explicit deterministic rules/provenance.

---

## P2-06 — Fact lifecycle incorrectly allows "retract" semantics

### Problem

The Mutation IR includes `fact.retract`.

In an event-sourced canonical world, normal world change should not erase that a fact was once true.

Example:

"The door was open from t1 to t2" should not disappear because the door later closes.

### Root cause

Current-truth projection semantics and historical truth semantics were mixed.

### Correction

Facts relevant to epistemics use validity/provenance:

- assert/open validity;
- close/end validity;
- supersede where appropriate.

Corrections of erroneous historical data are compensating/correction events, never silent historical deletion.

Also: do **not** duplicate every world component as a Fact.

Use propositions/facts when information needs to be referenced by epistemics, narrative rules, evidence or communication. Ordinary current world values may remain Entity/Component state.

---

## P2-07 — Narrative Director currently owns canonical progression by accident

### Problem

v0.2 says the Narrative Director deterministically selects:

- scene activation;
- available DecisionPoints/action surfaces;
- endings.

It also says the Narrative Director does not modify canonical truth.

These statements conflict.

If the Director decides which actions the players are allowed to take next, or whether the Session ends, it is changing canonical Session progression.

### Root cause

"World truth" and "gameplay progression" were treated as if only the former were canonical.

But canonical Session state includes progression/availability, not only fictional physics.

### Correction

Introduce **Scenario Progression** as part of the canonical deterministic engine.

Scenario Progression:

- evaluates triggers;
- activates/deactivates beats/scenes as mechanical progression state;
- opens ResolutionWindows/DecisionPoints;
- applies pacing gates that affect availability;
- determines canonical ending activation.

Its outputs are canonical Domain Events.

Narrative Direction becomes presentation-only:

- what already-authorized information to foreground;
- presentation ordering;
- emphasis;
- recap/pacing presentation that does not alter mechanical availability.

Narrative Realization remains wording/media.

This is a major correction.

---

## P2-08 — Post-commit work has a reliability gap

### Problem

v0.2 says narrative and secondary projections trigger after commit.

Failure case:

1. canonical state commits;
2. process crashes;
3. realtime notification / presentation generation / secondary projection trigger is lost.

The Session is correct but appears stuck.

### Root cause

The architecture separated the canonical transaction from downstream work without specifying durable handoff.

### Correction

Use a **Transactional Outbox** in the same database transaction as canonical commit for work that must happen after commit.

Initially this can be a PostgreSQL table polled by the application/worker.

No broker is required.

Consumers must be idempotent because delivery can be at-least-once.

Microsoft describes Transactional Outbox specifically as writing the business change and outgoing work/event in one transaction so post-commit delivery is recoverable.

Sources:
https://learn.microsoft.com/en-us/azure/architecture/databases/guide/transactional-out-box-cosmos
https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs

---

## P2-09 — Narrative output is noncanonical but causally relevant

### Problem

LLM-generated or template-generated prose does not define world truth.

But a player reads that prose and makes the next decision because of it.

Therefore it is causally relevant evidence for playtest/debug/replay.

Treating it as merely disposable derived content would lose why a human made a later choice.

### Correction

Store an immutable `PresentationRecord`/interaction artifact containing:

- audience;
- PresentationPlan hash;
- realizer/template/model version;
- prompt version where relevant;
- exact output;
- output hash;
- related canonical event/progression range;
- generation status.

It is **not Canonical World State**, but it is part of the immutable Session interaction record.

Narrative replay displays the stored artifact.

---

## P2-10 — Rule execution purity is implicit, not explicit

### Problem

The Scenario Bundle can contain rules, conditions, claim derivation and interaction policies, but v0.2 does not explicitly prohibit:

- I/O;
- current wall clock reads;
- ambient randomness;
- network calls;
- arbitrary JavaScript;
- mutable global state.

Any of these would break replay.

### Correction

Compiled scenario rules must be pure/deterministic functions of declared inputs.

They may access:

- pinned bundle;
- canonical base state;
- command/submission;
- explicit logical time;
- explicit named RNG service only where declared.

They may not access ambient clock, network, filesystem, arbitrary environment or unseeded randomness.

No arbitrary scenario script execution in MVP runtime.

---

## P2-11 — Build reproducibility is underspecified

### Problem

"Source commit or build identifier" is too weak if:

- dependency versions differ;
- build config changes;
- compiler/runtime changes.

### Correction

Pin an immutable `engine_build_id` / artifact digest produced from the built resolver runtime.

Retain source commit and dependency lockfile in the repository for reconstruction.

Historical replay support can be:

- native current-version replay;
- compatibility runner for supported historical build;
- forensic replay using stored canonical mutations/results when executable historical code is unavailable.

---

## P2-12 — Resolver replay needs two modes

### Problem

v0.2 says replay uses named random inputs, but does not distinguish recomputation from forensic reconstruction.

### Correction

### Verification replay
Recompute named draws from pinned RNG/version/seed and compare with stored results.

### Forensic replay
Use stored ActionSubmissions, ResolutionRecord, random evidence and canonical events to inspect/reconstruct a historical session even if an obsolete runtime cannot execute.

State replay must always remain possible from canonical transition data within the supported schema compatibility policy.

---

# Secondary issues that do not require baseline redesign

These can remain for contract/data-model work:

- exact Claim selector syntax;
- exact Mutation IR path/reference encoding;
- exact PRNG choice;
- exact fixed-point scales;
- exact event-family list;
- exact resolver strategy parameter schemas;
- snapshot frequency;
- exact SQL table/index design;
- exact scenario authoring syntax;
- relationship storage representation;
- exact goal/commitment/thread schemas.

---

# Postmortem root causes

## Root cause A — resolver-centric review

v0.2 spent most effort on:

`Action → Interaction → Resolution → Event`

and less on:

`Command → Session frontier → Resolution → Atomic commit → Durable post-commit effects → Next progression`.

Lesson: deterministic gameplay is an end-to-end property, not only a resolver property.

## Root cause B — "noncanonical" was treated too broadly

World truth, gameplay progression and player-facing presentation are three different categories.

v0.2 correctly excluded prose from canonical world state but incorrectly let presentation-layer logic choose mechanics.

Lesson: anything that changes future legal actions or ending state belongs to canonical progression even if it does not change fictional physics.

## Root cause C — determinism was specified semantically but not representationally

v0.2 said replayable/hashed but did not fully define:

- event metadata hash scope;
- ID generation;
- floating-point constraints;
- logical-time representation.

Lesson: deterministic replay requires deterministic data representation as well as deterministic algorithms.

## Root cause D — epistemics was compressed too early

The founding specification correctly insists on Fact ≠ Knowledge ≠ Belief.

The single stance enum reintroduced conflation in a different form.

Lesson: simplify data structures only after semantic distinctions are safe.

---

# Failure impact if v0.2 were implemented unchanged

Likely failures:

1. duplicate timeout/system commands creating repeated effects;
2. ambiguous scheduler behavior while a decision window is open;
3. false replay mismatches caused by timestamps/UUIDs;
4. floating-point drift in resource/relationship outcomes;
5. incorrect "disbelief" semantics and knowledge leakage;
6. facts disappearing from historical truth when current state changes;
7. Narrative Director silently becoming a second canonical authority;
8. committed sessions occasionally appearing stuck after process failure;
9. inability to explain player decisions because exact presented prose was not retained;
10. rules becoming accidentally non-deterministic through ambient APIs.

These are expensive to migrate after scenario content exists.

---

# Decision

**v0.2 is NOT validated as the final Phase-0 architecture baseline.**

It is strong enough to preserve most of its structure, but the issues above require a new working version.

**Decision:** produce Architecture Baseline v0.3 with targeted corrections rather than redesign from zero.

The following v0.2 pillars remain valid:

- Session-level canonical stream;
- selective Event Sourcing;
- PostgreSQL baseline;
- logical CQRS;
- componentized entities;
- immutable compiled Scenario Bundle;
- server-derived actions;
- typed claims/Interaction Graph;
- bounded deterministic resolver;
- semantic events + mutation representation;
- explicit epistemics;
- three time concepts;
- named seeded randomness;
- replay/version pinning;
- LLM outside canonical simulation.

v0.3 modifies the boundaries around commands, progression, event determinism, numeric determinism, epistemics and durable post-commit execution.
