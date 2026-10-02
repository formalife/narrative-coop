# Architecture Red-Team — Baseline v0.1

**Status:** PROPOSED review  
**Date:** 2026-10-02  
**Reviewed baseline:** `docs/architecture/ARCHITECTURE_BASELINE_v0.1.md`

## Purpose

This review stress-tests the v0.1 architecture before any concrete database schema or production code. It focuses on the four areas identified in `PROJECT_STATE.md`: effect algebra, semantic actions/claims, canonical event model, and deterministic resolution.

No item in this document is an ACCEPTED architectural decision by itself.

## External facts used

### Event Sourcing

Microsoft's current Event Sourcing guidance emphasizes that the pattern is useful where intent/history, auditability and replay matter, but also introduces significant costs around concurrency, schema evolution, projections and operations. It explicitly recommends selective rather than indiscriminate adoption.

Source: https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing

**Implication for this project:** keep Event Sourcing restricted to canonical Session history; do not extend it to accounts, drafts, analytics, assets or ordinary configuration.

### Optimistic concurrency

Kurrent/EventStore documentation uses expected stream revision/version as the basis for optimistic concurrency: appends succeed only when the stream is still at the version the writer observed.

Source: https://docs.kurrent.io/server/v25.1/http-api/introduction

**Implication:** the proposed Session revision check is a standard and appropriate concurrency pattern for a short two-player session stream.

### Event envelope design

CloudEvents defines a transport-neutral event context with stable identifiers, type, source, optional subject/time/schema metadata and separate event data.

Source: https://github.com/cloudevents/spec/blob/main/cloudevents/spec.md

**Implication:** borrow the separation of event identity/context from event payload, but do not adopt CloudEvents as the internal canonical storage contract unless interoperability later requires it.

### Generic JSON patching

RFC 6902 defines generic document mutation operations such as add, remove, replace, move, copy and test.

Source: https://www.rfc-editor.org/info/rfc6902/

**Implication:** generic patch operations prove that a small mutation algebra is possible, but JSON Patch alone is too semantically weak and too permissive for canonical game state. We need typed, schema-constrained mutation operations plus semantic domain events.

### Deterministic hashing

RFC 8785 defines a JSON Canonicalization Scheme specifically so logically identical JSON values can be serialized invariantly for repeatable hashes.

Source: https://www.rfc-editor.org/info/rfc8785/

**Implication:** state/event hashes must use a canonical serialization rather than ordinary `JSON.stringify` assumptions.

---

# Findings

## RT-01 — Effect algebra mixes abstraction levels

v0.1 places operations such as `SET`, `INCREMENT`, `MOVE`, `GRANT`, `OPEN` and `SCHEDULE` in one algebra.

Problem:

- `SET` is a low-level mutation.
- `MOVE` is a domain semantic action that may mean changing a Location component.
- `GRANT` could mean permission, knowledge, inventory or entitlement.
- `OPEN` could mean a door, narrative thread, access right or window.

This will produce ambiguous semantics and scenario-specific engine behavior.

### Proposed correction

Split two layers:

1. **Semantic Domain Event** — describes what canonically occurred.
2. **State Mutation IR** — small, typed internal representation of how canonical projections change.

A scenario author should normally work with semantic effects/macros. The compiler emits validated Mutation IR.

Proposed initial Mutation IR categories:

### World
- `entity.create`
- `entity.deactivate`
- `component.set`
- `number.adjust`
- `collection.add`
- `collection.remove`

### Epistemic
- `fact.assert`
- `fact.retract`
- `observation.record`
- `epistemic.set_stance`

### Scheduler
- `schedule.add`
- `schedule.cancel`

`move`, `transfer`, `consume`, `reveal`, `promise`, etc. remain semantic concepts/macros and compile into typed events + allowed state mutations.

Hard deletion of canonical entities should not be a normal operation; prefer deactivation/tombstoning.

---

## RT-02 — Client-owned claims are unsafe

v0.1 says an Action Proposal contains preconditions and claims.

That is too permissive.

A malicious or buggy client could omit a resource claim, change authority requirements or forge target semantics.

### Proposed correction

The client submits only a minimal **ActionSubmission**:

- submission id / idempotency key;
- session + resolution window;
- selected action type;
- allowed parameters/targets;
- optional source metadata.

The immutable Scenario Bundle owns the **ActionDefinition**:

- parameter schema;
- actor/role constraints;
- admission predicates;
- claim derivation;
- potential read/write footprint;
- duration model;
- resolver policy references;
- semantic event/effect templates.

The server derives an **ExpandedAction** against a specific base revision.

Claims, preconditions and potential effects are therefore authoritative and replayable.

---

## RT-03 — Action families must not dispatch mechanics

A fixed ontology such as COMMUNICATE / MOVE / SOCIAL / MANIPULATE risks becoming a universal verb taxonomy.

### Proposed correction

`action_type_id` is the mechanically authoritative identifier.

Action families/tags are optional classification metadata for:

- authoring;
- UI;
- analytics;
- narrative realization.

Resolver behavior comes from the ActionDefinition, claims and rules, not from a global verb hierarchy.

---

## RT-04 — Claim intersection is not enough

Two actions touching the same target are not automatically in conflict.

Examples:

- two READs are compatible;
- two resource CONSUMEs compete only if total demand exceeds capacity;
- a READ and WRITE may be causally relevant;
- a WRITE and WRITE may commute or conflict;
- two actions may be complementary rather than adversarial.

### Proposed correction

Claims need explicit access semantics.

Proposed claim modes:

- `READ`
- `WRITE`
- `CONSUME`
- `RESERVE`
- `PRODUCE`
- `EXCLUSIVE`

A Claim can include:

- target reference;
- optional component/field scope;
- mode;
- quantity + unit when relevant;
- logical time interval when relevant.

Authority, capability and knowledge requirements should be validators/preconditions, not contention claims.

The interaction builder uses a versioned compatibility matrix plus scenario interaction rules.

---

## RT-05 — Conflict graph should become an Interaction Graph

The v0.1 "conflict graph" framing misses positive and causal interactions.

### Proposed correction

Use **Interaction Graph** with typed edges:

- `CONTENTION` — limited capacity/resource;
- `EXCLUSION` — cannot both occur;
- `INTERFERENCE` — one may invalidate another's condition;
- `DEPENDENCY` — one action can enable/supply another;
- `COMPLEMENTARITY` — joint outcome differs from independent outcomes;
- `ORDER_SENSITIVE` — both may occur but order matters.

Independent actions have no edge.

Connected components can still be resolved separately.

---

## RT-06 — Admission and outcome are conflated

v0.1 outcomes such as ACCEPTED and REJECTED mix API/input validity with in-world consequences.

These are different concepts.

### Proposed correction

### AdmissionResult
Before the action enters canonical resolution:

- `ADMITTED`
- `REJECTED_INVALID`
- `REJECTED_UNAUTHORIZED`
- `REJECTED_STALE`
- `REJECTED_NOT_ADDRESSABLE`

Rejected submissions remain audit/debug records and normally do not become world events.

### ActionOutcome
For an admitted action resolved inside the world:

- `SUCCEEDED`
- `PARTIAL`
- `FAILED`
- `INTERRUPTED`
- `TRANSFORMED`
- `NO_EFFECT`

An admitted action that fails can still be canonically meaningful and narratable.

---

## RT-07 — Simultaneous semantics are underspecified

"Same resolution window" is insufficient if internal evaluation order can change outcomes.

### Proposed correction

A simultaneous ResolutionWindow uses **set semantics**:

- all submissions resolve against the same frozen base revision;
- HTTP arrival order has no semantic meaning;
- actions are expanded from the same base snapshot;
- interactions are resolved jointly;
- deterministic ordering is used only after semantics have been decided.

If one action is allowed to enable another in the same window, that must be an explicit interaction rule, not an accidental consequence of iteration order.

Sequential windows remain explicitly ordered.

---

## RT-08 — Internal iteration order must not become game priority

Sorting actions by UUID is acceptable for deterministic implementation, but not as a semantic conflict rule.

### Proposed correction

Semantic priority may come only from explicit data:

- logical time;
- authority/control rules;
- scenario priority;
- declared resolution strategy;
- explicitly seeded random selection.

A stable ID sort is allowed only as a final serialization/tie ordering when outcomes are otherwise semantically equivalent.

---

## RT-09 — Generic resolver needs bounded strategies

A fully generic rule engine would drift toward a programming language.

### Proposed correction

The core resolver should offer a small number of versioned strategies, parameterized by scenario data:

- `COMPOSE`
- `CAPACITY_ALLOCATE`
- `EXCLUSIVE_PRIORITY`
- `ORDERED_APPLY`
- `MUTUAL_FAILURE`
- explicit scenario interaction table/transform for exceptional paired or grouped interactions.

No arbitrary runtime script execution in published scenario bundles for MVP.

---

## RT-10 — Event taxonomy needs two axes

A choice between "one event per verb" and "GenericEvent" is false.

### Proposed correction

Every canonical event gets:

1. **event_family** — finite core category for tooling/reducers.
2. **event_code** — precise semantic code, core or scenario-defined.

Example:

```
event_family: world.condition
event_code: island.radio_sabotaged
```

This preserves semantic narrative value while keeping the engine taxonomy bounded.

---

## RT-11 — Event envelope should not own visibility

v0.1 includes `visibility_policy` in the canonical event concept.

That risks treating secrecy as a filter over truth instead of explicit epistemic state.

### Proposed correction

Canonical occurrence events contain full truth.

Who observed/learned/believes what changes only through explicit epistemic events/mutations such as:

- ObservationRecorded
- Knowledge/StanceChanged
- CommunicationClaimMade

Player views derive from epistemic state.

There is no generic "hide this event from Player B" switch as the epistemic model.

---

## RT-12 — Proposed event envelope

Minimum context:

- `event_id`
- `session_id`
- `stream_revision`
- `event_batch_id`
- `batch_index`
- `event_family`
- `event_code`
- `event_schema_version`
- `logical_time`
- `recorded_at` — wall-clock audit only
- `correlation_id` — normally resolution/window
- `causation_refs[]` — actions/events
- `actor_refs[]`
- `subject_refs[]`
- `payload`
- `mutations[]` — validated State Mutation IR where state changes are required

Session/version manifest information should generally be pinned once on Session/ResolutionRecord rather than redundantly copied into every event.

---

## RT-13 — Random draw indexing is fragile

Deriving all randomness from "operation index 1, 2, 3..." means adding an unrelated random draw can change every subsequent result.

### Proposed correction

Every canonical random decision has a stable **draw_key**.

The RNG service derives an independent deterministic draw scope from:

- session seed;
- resolution id;
- rule id;
- draw key;
- RNG algorithm/version.

Adding an unrelated random decision should not perturb existing named draws.

The exact PRNG remains unfrozen, but its algorithm/version must be pinned.

---

## RT-14 — State hashing requires canonical serialization

Hashing ordinary JSON serialization is unsafe because representation details can change.

### Proposed correction

Define one canonical state serialization before hashing. RFC 8785/JCS is the preferred reference candidate for JSON-compatible state.

Hash:

- state projection;
- event batch;
- compiled scenario bundle;

only after canonicalization.

---

## RT-15 — CQRS consistency needs two classes of projection

Gameplay cannot safely wait for eventually consistent projections.

### Proposed correction

### Synchronous critical projection
Updated atomically with the canonical event append:

- Session revision/state;
- state hash;
- scheduler state required for subsequent commands.

### Rebuildable/secondary projections
May update asynchronously or on demand:

- recap;
- analytics;
- admin aggregations;
- historical reports.

Player view can be computed deterministically from the current Session/Epistemic projection rather than maintained as an independent source of truth.

---

## RT-16 — Narrative Director must be deterministic

The v0.1 separation is correct, but it did not explicitly require deterministic scene/presentation selection.

### Proposed correction

Narrative Director decisions that affect:

- which scene becomes active;
- which decision options become available;
- pacing gates;
- ending selection;

must be deterministic/versioned/seeded just like simulation rules.

Only realization wording may be non-deterministic.

When multiple scene candidates match, scenario data must provide deterministic priority or an explicitly declared seeded selection policy.

---

## RT-17 — Wall-clock deadlines cannot be replayed from time

Real-time timers are external triggers.

### Proposed correction

When a deadline occurs, runtime infrastructure issues a command and the engine records the resulting canonical window/session event.

Historical replay replays the recorded event; it does not wait for or re-evaluate the real clock.

Logical delayed consequences remain inside the canonical Scheduler.

---

## RT-18 — Target addressability is a security/domain invariant

A participant should not be able to submit a hidden entity ID discovered by client tampering.

### Proposed correction

Action validation includes **addressability**:

a target can be referenced only when scenario rules permit the acting character to address/control/select it, independently of whether the raw ID exists in canonical state.

This is distinct from capability and physical reachability.

---

## RT-19 — Session aggregate remains acceptable but needs revisit conditions

A single Session stream is the simplest consistency model for the current two-player, short-session product.

It could become a bottleneck if future scenarios contain:

- many autonomous high-frequency NPCs;
- substantially longer persistent worlds;
- more players;
- extremely high automatic event rates.

Do not partition now. Record these as revisit conditions for ADR-002.

---

# Resolver v0.2 — proposed algorithm

1. Close/freeze the ResolutionWindow.
2. Select the final submission for every required action slot.
3. Deduplicate by submission id/idempotency key.
4. Freeze `base_revision` and current canonical projection.
5. Validate submission schema and participant/window ownership.
6. Resolve actor and target addressability.
7. Load immutable ActionDefinition from the pinned Scenario Bundle.
8. Derive ExpandedAction:
   - normalized parameters;
   - admission predicates;
   - claims;
   - read/write footprint;
   - logical timing;
   - applicable resolver tags/policies.
9. Produce AdmissionResult for each submission.
10. Build Interaction Graph over ADMITTED actions.
11. Partition into connected components.
12. For each component:
    - enforce engine invariants;
    - match explicit scenario interaction rules;
    - choose declared generic resolution strategy;
    - request any named deterministic random draws;
    - produce ActionOutcomes and a ResolutionPlan.
13. Validate the complete ResolutionPlan globally.
14. Materialize semantic Domain Events + typed Mutation IR.
15. Apply the event batch to an in-memory projection.
16. Run post-state, resource, temporal and epistemic invariants.
17. Canonicalize and hash before state, event batch and after state.
18. Atomically append the batch and update the synchronous current projection only if the Session is still at `base_revision`.
19. On expected-revision conflict, discard the uncommitted plan. Re-evaluate only if the ResolutionWindow is still valid under the new state; otherwise surface a controlled stale-resolution condition.
20. Trigger narrative/secondary projections only after commit.

---

# New invariants proposed for v0.2

1. **No client-authored canonical claims or effects.**
2. **Simultaneous outcome is invariant to submission arrival order.**
3. **Every state mutation is schema-valid and targets an allowed state domain.**
4. **Every admitted action has exactly one ActionOutcome.**
5. **Every canonical state change is attributable to one or more canonical events.**
6. **Independent interaction components commute at the final-state level.**
7. **No wall-clock value directly participates in replay semantics.**
8. **Random decisions are named, seeded, versioned and recorded.**
9. **Player views cannot expose a proposition without epistemic provenance/permission.**
10. **A current projection can always be rebuilt from the pinned bundle + event stream.**
11. **State/event hashing uses a declared canonical serialization.**
12. **Narrative realization can fail without corrupting canonical simulation.**

---

# Decisions still premature after this review

- exact action-family vocabulary;
- exact syntax of claim selectors;
- exact State Mutation IR field/path encoding;
- exact PRNG;
- exact scenario DSL syntax;
- whether natural-language actions require player confirmation;
- snapshot frequency;
- exact scene-priority syntax;
- whether some read projections are persisted or computed on demand;
- eventual future custom scripting/plugin escape hatch.

These should not block Architecture Baseline v0.2.

---

# Recommended transition

The above corrections are substantial enough that v0.1 should remain as historical proposal and a new `ARCHITECTURE_BASELINE_v0.2.md` should become the active PROPOSED baseline.

No ADR should be marked ACCEPTED until the user explicitly accepts the v0.2 decisions.
