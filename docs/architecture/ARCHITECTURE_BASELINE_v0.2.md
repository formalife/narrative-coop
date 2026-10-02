# ARCHITECTURE BASELINE v0.2

**Status:** PROPOSED  
**Date:** 2026-10-02  
**Supersedes as working proposal:** v0.1  
**Implementation:** NOT STARTED

## 1. Architectural thesis

The product is a **deterministic two-player narrative simulation with asymmetric information, asymmetric agency, concurrent/temporally interacting semantic actions, and one canonical timeline per Session**.

It is not:

- a branching-story tree;
- a two-player chatbot;
- an LLM Game Master;
- a universal simulator;
- a distributed multi-agent infrastructure problem.

The distinguishing mechanic is:

> **ASYMMETRIC AGENCY + CAUSAL MERGE**

The engine must remain fully correct with runtime LLM generation disabled.

---

## 2. Core invariants

1. A Session has one canonical ordered event stream.
2. Canonical state changes only through committed canonical event batches.
3. A committed batch is resolved against one explicit `base_revision`.
4. Simultaneous actions use set semantics; network arrival order has no gameplay meaning.
5. Canonical randomness is explicit, named, seeded, versioned and logged.
6. Scenario content is immutable/version-pinned for an active Session.
7. LLM output never directly mutates canonical state.
8. Player views are derived from explicit epistemic state/permissions.
9. Narrative realization cannot alter simulation truth.
10. Historical replay must not depend on current wall-clock time or current LLM behavior.
11. Current state projections are rebuildable from the pinned scenario bundle + event stream.
12. Runtime behavior must be data-driven enough that new episodes normally require configuration/content rather than engine code.

---

## 3. Logical bounded contexts

These are **logical module boundaries**, not microservices and not separate databases by default.

### Scenario Authoring & Compilation
Owns source definitions and compilation/validation into immutable `CompiledScenarioBundle`.

### Session & Participation
Owns Session lifecycle, participants, invites, action slots, role/character assignment and synchronization state.

### Canonical Simulation
Owns resolution windows, authoritative action expansion, interaction detection, resolution plans, canonical events, state mutation application and session revision.

### Epistemics & Visibility
Owns facts/propositions, observations, knowledge/belief stance, communication claims, evidence relations, addressability and player-visible epistemic projections.

### Temporal & Scheduler
Owns logical time, action temporal metadata, scheduled logical consequences and deterministic due-effect generation.

### Narrative Direction
Selects deterministic presentation/scene/decision structures from canonical state. It cannot modify truth.

### Narrative Realization
Turns an already-authorized PresentationPlan into prose/dialogue/media references. LLM use is optional and noncanonical.

### Identity & Entitlements
Guest identity/account/ownership boundaries. Outside simulation core.

### Analytics & Operations
Product telemetry and operational observability. Outside canonical simulation and prohibited from receiving full private world state by default.

---

## 4. Sources of truth

### Canonical historical truth
The Session event stream.

### Current runtime state
A synchronous rebuildable `SessionState` projection at a known `stream_revision` and `state_hash`.

### Scenario truth
The immutable CompiledScenarioBundle pinned by the Session.

### Noncanonical outputs
Narrative prose, UI formatting, analytics projections and other derived artefacts.

---

## 5. Scenario bundle

A published `CompiledScenarioBundle` is immutable and content-addressed.

It may include:

- role definitions;
- entity/component schemas;
- initial state;
- proposition/fact schemas;
- ActionDefinitions;
- DecisionPointDefinitions;
- claim derivation rules;
- interaction/resolution policies;
- semantic event templates;
- allowed State Mutation IR templates;
- scene/presentation rules;
- narrative threads/goals/commitments;
- communication policy;
- ending rules;
- scheduler definitions;
- metrics;
- asset manifest;
- schema/rule version manifest.

Runtime never executes mutable authoring source directly.

The final authoring syntax remains unfrozen.

---

## 6. Componentized entity model

Use a componentized domain entity model, not a performance-oriented ECS runtime.

```
Entity
  id
  archetype/tags
  Components[]
```

Each Component has:

- `component_type`;
- schema version;
- validated data.

Behavior is not hidden in scenario-specific entity classes.

Canonical entities are normally deactivated/tombstoned rather than hard-deleted so historical references remain valid.

---

## 7. Semantic action model

### 7.1 ActionSubmission — client/input boundary

A participant submits only authoritative-safe input:

```
ActionSubmission
  submission_id
  session_id
  resolution_window_id
  participant_id
  action_type_id
  parameters
  selected_target_refs[]
  source
  submitted_at   // audit only
```

`submission_id` is an idempotency key.

The client does **not** author:

- claims;
- permissions;
- preconditions;
- canonical effects;
- resolver priority;
- resource costs.

### 7.2 ActionDefinition — scenario authority

Pinned inside the CompiledScenarioBundle:

```
ActionDefinition
  action_type_id
  parameter_schema
  actor_constraints
  target/addressability rules
  admission_predicates
  claim_derivation
  potential_read_write_footprint
  timing_model
  resolver_policy_refs
  semantic_event/effect_templates
  optional classification tags
```

Action families/tags are classification, not mechanic dispatch.

### 7.3 ExpandedAction — server-derived resolution input

Derived against one base revision:

```
ExpandedAction
  action_instance_id
  submission_id
  definition_ref
  actor_entity_id
  normalized_parameters
  resolved_targets
  derived_claims[]
  read_write_footprint
  logical_interval
  resolver_tags
  base_revision
```

ExpandedAction is generated deterministically from pinned definitions + submission + base state.

---

## 8. Admission versus in-world outcome

### AdmissionResult

Determines whether a submission can enter resolution:

- `ADMITTED`
- `REJECTED_INVALID`
- `REJECTED_UNAUTHORIZED`
- `REJECTED_STALE`
- `REJECTED_NOT_ADDRESSABLE`

Non-admitted submissions remain audit/debug data and normally do not become world events.

### ActionOutcome

For an admitted action:

- `SUCCEEDED`
- `PARTIAL`
- `FAILED`
- `INTERRUPTED`
- `TRANSFORMED`
- `NO_EFFECT`

A failed admitted action may still create canonical semantic events if the attempt itself matters.

---

## 9. Claim model

A Claim describes contention-relevant access, not general semantic intent.

Proposed structure:

```
Claim
  target_ref
  optional scope/component/path
  mode
  optional quantity
  optional unit
  optional logical_interval
```

Initial modes:

- `READ`
- `WRITE`
- `CONSUME`
- `RESERVE`
- `PRODUCE`
- `EXCLUSIVE`

Authority, capability, knowledge and addressability are validation constraints, not ordinary claims.

A versioned compatibility matrix determines generic interaction candidates.

Exact claim-selector syntax remains unfrozen.

---

## 10. Interaction Graph

Actions are vertices.

Typed edges represent meaningful interaction:

- `CONTENTION`
- `EXCLUSION`
- `INTERFERENCE`
- `DEPENDENCY`
- `COMPLEMENTARITY`
- `ORDER_SENSITIVE`

Independent actions have no edge.

Connected components are resolved independently, but the complete resulting plan is globally validated before commit.

A core test property is that permuting independent components does not change final canonical state.

---

## 11. Resolution strategies

Avoid a universal constraint solver or arbitrary runtime scripting.

Core versioned strategies should cover common classes:

- `COMPOSE` — compatible/commutative effects;
- `CAPACITY_ALLOCATE` — finite quantitative resource;
- `EXCLUSIVE_PRIORITY` — one action controls an exclusive target according to explicit priority;
- `ORDERED_APPLY` — declared logical order matters;
- `MUTUAL_FAILURE` — interaction invalidates all involved attempts;
- explicit compiled scenario interaction transformation for exceptional combinations.

Engine invariants cannot be bypassed by scenario interaction rules.

The exact strategy parameter schema remains subject to the next schema-design phase.

---

## 12. Deterministic resolver pipeline

1. Freeze/close the ResolutionWindow.
2. Choose the final submission for each action slot according to window policy.
3. Deduplicate submissions.
4. Freeze `base_revision` and current SessionState.
5. Validate schema, participant ownership and window membership.
6. Validate target addressability.
7. Load pinned ActionDefinitions.
8. Derive ExpandedActions against the same base revision.
9. Produce AdmissionResults.
10. Build Interaction Graph for ADMITTED actions.
11. Partition into connected components.
12. Resolve each component using:
    - engine invariants;
    - explicit scenario interaction rules;
    - declared generic strategy;
    - named deterministic random draws where required.
13. Produce a complete `ResolutionPlan` with one ActionOutcome per admitted action.
14. Validate the plan globally.
15. Materialize semantic canonical events + State Mutation IR.
16. Apply event batch to transient state.
17. Run post-state validators.
18. Canonicalize and hash before state, event batch and after state.
19. Atomically append the batch and update synchronous projection only if current Session revision equals `base_revision`.
20. Trigger narrative and secondary projections after commit.

If the expected revision check fails, the uncommitted plan is discarded. It may be recomputed only if the window remains valid in the new state.

---

## 13. Semantic event model

Do not use one event per possible verb and do not use an untyped `GenericEvent`.

Each canonical event has two semantic axes:

### event_family
Finite engine-level category used by tooling/validation.

Examples of families:

- session.lifecycle
- temporal
- world.entity
- world.condition
- world.resource
- world.location
- communication
- epistemic
- relationship
- commitment
- goal
- narrative_thread
- action_outcome

### event_code
Precise core or scenario-specific semantic occurrence.

Example:

```
event_family: world.condition
event_code: island.radio_sabotaged
```

The family set is deliberately small. Scenario-specific narrative meaning lives in event_code + payload.

---

## 14. Canonical event envelope

Proposed minimum:

```
DomainEvent
  event_id
  session_id
  stream_revision
  event_batch_id
  batch_index

  event_family
  event_code
  event_schema_version

  logical_time
  recorded_at

  correlation_id
  causation_refs[]
  actor_refs[]
  subject_refs[]

  payload
  mutations[]
```

Notes:

- `recorded_at` is operational metadata, not replay ordering.
- `stream_revision` is the authoritative total session order.
- `batch_index` gives deterministic ordering inside one committed resolution batch.
- `correlation_id` normally points to resolution/window context.
- `causation_refs` can reference Action instances and previous events.
- visibility is **not** a generic envelope field.
- session-level version pins remain on Session/ResolutionRecord rather than copied unnecessarily to every event.

CloudEvents may later be used as an external transport representation, but is not the required internal schema.

---

## 15. State Mutation IR

The Mutation IR is an internal compiled/replay representation, not a narrative ontology.

Initial proposed operations:

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

Rules:

- every target is schema-resolved at compile/runtime validation;
- arbitrary JSON paths into protected system state are forbidden;
- numeric constraints/invariants are checked after application;
- macros such as move/transfer/consume/reveal compile to these operations plus semantic event context;
- batches apply atomically.

The exact path/reference encoding remains unfrozen.

---

## 16. Epistemic model

Keep objective truth and subjective state separate.

### Proposition

Scenario-defined structured proposition, for example:

```
predicate: radio.sabotaged
arguments: [radio_01]
value: true
```

Avoid requiring a universal formal logic engine.

### Fact
Canonical truth assertion about a Proposition, optionally with validity interval and provenance.

### Observation
A character directly perceives some proposition/evidence/event.

### Epistemic stance
Character-specific state referencing a Proposition, such as:

- KNOWN
- BELIEVED
- SUSPECTED
- DISBELIEVED
- UNKNOWN

Confidence is optional/scenario-specific rather than globally mandatory.

### Communication claim
A speaker asserts a Proposition to a recipient/channel. This does not make the proposition true.

### Secret
A visibility/access classification or narrative concept over propositions/evidence; it is not a separate truth category.

### Evidence
Canonical world entity/relation that can support or contradict propositions.

Player views derive from these structures. Raw canonical events are never simply handed to an LLM with instructions to "hide secrets."

---

## 17. Addressability

An Action target is valid only if the actor is allowed to address/select/control that target under scenario rules.

Addressability is distinct from:

- existence;
- physical reachability;
- capability;
- ownership;
- knowledge.

This prevents hidden canonical IDs from becoming usable merely because a modified client submits them.

---

## 18. Temporal model

Keep three time concepts separate.

### Engine order
`stream_revision` / batch ordering. Total canonical technical order.

### Logical time
In-world time used by scenario rules and delayed effects.

### Wall clock
Real time used for UX, deadlines and infrastructure triggers.

Wall-clock deadlines cause commands/events to be recorded. Replay consumes the recorded events; it does not re-evaluate historical real time.

### ResolutionWindow

Contains at least:

- id;
- base/opening revision;
- mode;
- action slots/participants;
- logical timing;
- close policy;
- real deadline if any;
- timeout/disconnect policy;
- resolution policy.

Modes may include simultaneous, sequential, secret-simultaneous, conditional, single-actor and automatic.

A simultaneous window resolves the action set against one base snapshot.

---

## 19. Scheduler

Logical delayed consequences are canonical scheduled records.

A scheduled item includes:

- schedule id;
- source/causation;
- logical due condition/time;
- semantic effect/event template;
- cancellation conditions;
- version info.

Scheduler firing is deterministic from logical state.

Real-world timers are infrastructure triggers, not replayed clocks.

---

## 20. Randomness

No ambient `Math.random()`.

Every canonical draw uses:

- session seed;
- resolution id;
- rule id;
- stable `draw_key`;
- pinned RNG algorithm/version;
- distribution/parameters;
- recorded result.

Named draw scopes prevent insertion of an unrelated random decision from changing other outcomes.

Exact PRNG selection remains unfrozen until implementation contract design.

---

## 21. CQRS and projections

CQRS is logical, not distributed infrastructure.

### Synchronous critical projection

Updated in the same database transaction as canonical append:

- current SessionState;
- stream revision;
- state hash;
- scheduler state required for following commands.

### Secondary projections

Rebuildable and allowed to lag:

- recap;
- product analytics;
- aggregate admin/reporting;
- optional historical views.

Player-specific views should be deterministically derived from current state + epistemics/permissions, not maintained as a competing source of truth.

No broker or second database is required for v0.2.

---

## 22. Concurrency and commit

Session remains the runtime consistency boundary.

Commit protocol:

```
load state @ revision N
resolve against N
build event batch
build after-state
validate
BEGIN
  verify current revision == N
  append ordered event batch
  update critical projection + hash
  set revision = N + event_count (or defined revision convention)
COMMIT
```

The concrete revision convention will be specified in the data model.

Expected-revision optimistic concurrency is sufficient for the current two-player short-session product.

Revisit Session partitioning only if measured requirements introduce long persistent worlds, many players, high-frequency autonomous agents or unacceptable write contention.

---

## 23. Canonical serialization and hashes

Raw object serialization must not define hashes accidentally.

The implementation must pin a canonical representation for hashable JSON structures.

**Preferred reference candidate:** RFC 8785 JSON Canonicalization Scheme.

Record/derive at least:

- scenario bundle hash;
- before-state hash;
- event-batch hash;
- after-state hash.

Hash algorithm and canonicalization version are version-pinned.

---

## 24. Versioning

Each Session pins enough information to reproduce its mechanics:

- engine version / source commit or immutable build identifier;
- scenario id/version/bundle hash;
- state schema version;
- event schema version(s);
- action contract version;
- mutation IR version;
- resolver/rule bundle version;
- director version;
- RNG algorithm/version + session seed;
- narrative template/prompt versions for stored realization audit.

Published event records are immutable.

Schema evolution uses explicit versions/upcasters/migrations where needed; never silently reinterpret old events with new semantics.

---

## 25. Replay

### State replay

```
initial pinned bundle state
+ canonical event mutations in stream order
→ reconstructed SessionState
→ compare state hash
```

### Resolver replay

```
base state
+ final ActionSubmissions
+ pinned bundle/rules
+ named random inputs
→ AdmissionResults
→ Interaction Graph
→ ResolutionPlan
→ canonical event batch
```

Must reproduce historical canonical outputs/hash for supported historical versions.

### Narrative replay

Historical generated prose is displayed from stored output. Do not call the current LLM and call that "replay."

---

## 26. Narrative architecture

### Simulation
Determines what canonically happens.

### Narrative Director
Deterministically decides:

- scene/presentation activation;
- which known events/information are surfaced;
- available DecisionPoints/action surfaces;
- pacing gates;
- endings.

Director behavior is versioned and obeys deterministic/seeded rules.

### Narrative Realization
Produces wording/media presentation from a player-safe PresentationPlan.

LLM use is allowed here but optional.

Runtime must have a deterministic/template fallback so an unavailable model cannot invalidate a playable Session.

---

## 27. Decision points and free text

A `DecisionPointDefinition` is an authoring/presentation structure, not the world action itself.

A displayed Option can map to one or more ActionSubmission templates.

Potential future free-text path:

```
Player utterance
→ optional LLM/parser
→ candidate structured action
→ validation/optional confirmation
→ ActionSubmission
```

The parser output is logged; historical replay uses the stored structured action rather than rerunning the model.

Whether confirmation is mandatory remains a product decision.

---

## 28. Event Sourcing scope

Event Source the canonical Session stream because historical causality/replay is product-critical.

Do not Event Source by default:

- accounts/profile data;
- entitlements;
- analytics;
- page views;
- source scenario drafts;
- media generation;
- temporary UI state;
- technical logs.

Snapshots, if introduced, are optimization only and never replace the event stream as historical source.

---

## 29. Validation architecture

### Scenario compile-time

Validate:

- schema correctness;
- reference integrity;
- ActionDefinitions;
- claim/mutation targets;
- protected state access;
- interaction policies;
- scene/ending rule ambiguity;
- deterministic priority rules;
- scheduler definitions;
- version manifest;
- asset references;
- epistemic/presentation constraints where statically decidable.

### Runtime pre-admission

Validate:

- session/window;
- participant;
- action schema;
- actor/role;
- addressability;
- capability;
- permissions;
- base revision/staleness;
- preconditions.

### Resolution-plan

Validate:

- one outcome per admitted action;
- resource allocation;
- allowed mutations;
- deterministic random evidence;
- no unresolved interaction component.

### Post-state

Validate:

- state schemas;
- resources/invariants;
- entity references;
- temporal consistency;
- epistemic provenance;
- relationship constraints;
- scheduler consistency;
- ending/session constraints.

### Replay

Validate canonical hashes and historical resolver output.

---

## 30. Testing architecture

Priority order:

1. deterministic resolver unit tests;
2. state transition tests;
3. property-based tests;
4. interaction/conflict fixtures;
5. scenario compiler validation;
6. replay regression;
7. synthetic playthroughs;
8. later load/performance tests.

Critical properties include:

- same inputs + versions + seed → same result;
- simultaneous submission permutation → same semantic result;
- duplicate submission id → no duplicate action;
- independent interaction component permutation → same after-state;
- resources do not violate declared invariants;
- forbidden knowledge/addressability never appears;
- event replay → stored state hash;
- unrelated named random draw addition does not perturb existing draw results;
- narrative realization failure does not modify canon.

A common PlayerController boundary should support Human, Scripted, Random and future Agent controllers.

---

## 31. Persistence baseline

Still PROPOSED:

- PostgreSQL as primary persistence;
- canonical `session_events` append-only table/stream representation;
- relational metadata for Session/participants/scenario versions;
- schema-validated JSON/JSONB where component flexibility materially helps;
- synchronous current Session projection;
- no dedicated event-store product required for MVP unless Postgres implementation proves inadequate.

No concrete DB schema is frozen in v0.2.

---

## 32. Technical infrastructure baseline

Still under evaluation, not ACCEPTED:

- React + TypeScript browser/PWA client;
- Supabase/PostgreSQL baseline for MVP database/auth/API/realtime;
- realtime as notification, never source of truth;
- Cloudflare hosting/storage where useful;
- R2 for media;
- PostHog for privacy-minimized product telemetry;
- monorepo with pure domain/resolver/compiler packages separated from providers.

Do not introduce Redis, brokers, graph DB, vector DB, Durable Objects or microservices without measured need.

---

## 33. Security/privacy invariants

- Browser cannot directly append canonical events or mutate canonical state.
- Participant identity never authorizes another participant's private projection.
- Secret/unknown entity IDs are not automatically addressable.
- Full world state, private player views and free-text secrets are not sent to analytics by default.
- Technical logs redact private state by design.
- Guest play should remain possible with minimal PII.
- Published scenario/runtime bundles contain no service secrets.

---

## 34. Decisions deliberately still unfrozen

- exact action-family/tag vocabulary;
- exact claim-selector syntax;
- exact Mutation IR target/path representation;
- exact PRNG;
- final Scenario DSL syntax;
- natural-language action confirmation policy;
- snapshot strategy/frequency;
- exact DB tables/indexes;
- precise component schema representation;
- Scenario Studio UI;
- NPC agent architecture;
- vector/graph databases;
- Redis/Durable Objects;
- microservice split;
- payment provider;
- retention periods;
- recap/video generation.

---

## 35. Revisit triggers

### Single Session stream
Revisit if measured contention, long-lived worlds, many players or high-frequency autonomous agents make serialization unacceptable.

### PostgreSQL event storage
Revisit if operational/replay/query requirements demonstrate that a specialized event store materially simplifies the system.

### Synchronous critical projection
Revisit only if transaction cost becomes a measured bottleneck.

### Declarative resolver strategies
Revisit if multiple real scenarios cannot be expressed without repeated engine changes.

### State Mutation IR
Revisit if authoring repeatedly requires unsafe generic path mutation or scenario-specific engine code.

---

## 36. What v0.2 establishes if accepted

The following should become ADR candidates for explicit acceptance:

- selective Session Event Sourcing;
- one Session canonical stream / consistency boundary;
- server-derived Action expansion;
- typed claims + Interaction Graph;
- deterministic bounded resolver strategies;
- semantic event + State Mutation IR split;
- explicit epistemic model;
- temporal three-clock separation;
- deterministic Narrative Director / noncanonical Realizer;
- immutable compiled scenario bundle;
- version pinning, canonical hashes and replay;
- logical CQRS with synchronous critical projection;
- PostgreSQL baseline.

Until explicitly accepted, this entire file remains a **PROPOSAL**.
