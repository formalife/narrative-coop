# POSTMORTEM — Domain Contracts v0.1

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/domain/DOMAIN_CONTRACTS_v0.1.md`  
**Governing architecture:** Architecture Baseline v0.3 — ACCEPTED  
**Verdict:** MODIFY — do not accept Domain Contracts v0.1.

## Executive verdict

Domain Contracts v0.1 successfully translated most accepted architecture into concrete runtime vocabulary, but the red-team found several contract-level ambiguities that would create bugs or force migrations if implemented now.

No finding requires superseding ADR-001 through ADR-014.

The problems are refinements inside the accepted architecture, especially around:

- ResolutionWindow freeze semantics;
- idempotency and deadline/replacement races;
- simultaneous dependency semantics;
- canonical transition records not caused by player resolutions;
- canonical collection ordering/hashing;
- state-hash self-reference;
- runtime DecisionPoint instances;
- dynamic propositions;
- scheduler/progression ordering;
- presentation/mechanical affordance separation;
- progression termination;
- presentation-worker version pinning.

**Decision:** preserve v0.1 as historical proposal and replace it with `DOMAIN_CONTRACTS_v0.2.md`.

---

# 1. Synthetic red-team results

| # | Case | v0.1 result | Finding |
|---|---|---|---|
| 1 | Both players consume same scarce battery | PARTIAL PASS | Aggregate grouping exists, but capacity allocation outcome contract is still underspecified. |
| 2 | Sabotage radio while other transmits | PASS | Interaction Graph can express interference/order-sensitive resolution. |
| 3 | Independent actions in any arrival order | PASS | Frozen-set semantics + commutativity invariant are adequate. |
| 4 | Hidden EntityId manually submitted | PASS | Addressability provides the required domain/security gate. |
| 5 | Replace action immediately before deadline | FAIL | `LATEST_VALID_SUBMISSION` + wall `received_at` does not define a safe race/linearization contract. |
| 6 | Deadline command delivered twice | PARTIAL FAIL | Same CommandId deduplicates retries, but two equivalent deadline deliveries with different IDs can still duplicate intent. |
| 7 | Logical scheduled effect becomes due while window is open | FAIL/AMBIGUOUS | The accepted frontier principle exists, but pre/post-action boundary ordering is not contracted. |
| 8 | Three actions collectively exceed capacity though each pair fits | PASS | Aggregate interaction-group invariant is present. |
| 9 | One action depends on another in same simultaneous window | FAIL | State-dependent preconditions are evaluated before interaction, so an action enabled by another action may be rejected too early. |
| 10 | Player lies; recipient believes; canon remains false | PASS | CommunicationClaim / Belief / Fact separation works. |
| 11 | Evidence changes belief; historical lie remains | PASS | Append/end epistemic records support this, though supersession semantics need tightening. |
| 12 | Fact validity ends; historical observation remains | PASS | Fact validity and ObservationRecord are independent. |
| 13 | Realizer crashes after canonical commit | PASS | Transactional Outbox protects required post-commit work. |
| 14 | Outbox worker retries twice | PASS WITH CONDITION | Requires strict deduplication and immutable generation context. |
| 15 | Unrelated RNG draw is added | PARTIAL PASS | Named draw exists, but derivation context must explicitly exclude unrelated draw/order state. |
| 16 | Presentation wording changes; world replay hash unchanged | PASS | Presentation is outside canonical state/hash. |
| 17 | Engine build upgrades; old Session replayed | PARTIAL PASS | Version manifest and forensic replay exist, but mechanical/presentation versions are currently mixed. |
| 18 | Ending trigger + another progression trigger become true together | FAIL/AMBIGUOUS | Deterministic priority is mentioned, but progression convergence/termination/composition contract is missing. |

---

# 2. Findings

## DC-01 — FROZEN/RESOLVING window state conflicts with the Single Canonical Frontier

### Problem

v0.1 models:

```
OPEN -> FROZEN -> RESOLVING -> RESOLVED
```

as a ResolutionWindow lifecycle while also stating:

> canonical gameplay state does not advance while an OPEN simultaneous window collects submissions.

If FROZEN or RESOLVING are themselves canonical SessionState transitions committed before resolution, the Session revision advances away from `base_revision` before the resolver runs.

If they are not canonical, they should not be modeled as canonical Window status.

### Correction

Separate:

### Canonical ResolutionWindow state
- OPEN
- RESOLVED
- CANCELLED
- INTERRUPTED

### ResolutionAttempt operational/audit state
- PREPARING
- FROZEN
- RESOLVING
- COMMITTED
- ABORTED

Freezing the selected input set occurs inside one frontier operation without first committing a world-state revision.

The canonical closing/resolution event is part of the final atomic transition batch.

---

## DC-02 — Replacement versus deadline race is not linearized safely

### Problem

`LATEST_VALID_SUBMISSION` appears to imply ordering by `received_at`, but wall-clock receipt time is operational metadata and can be affected by retries/network ordering.

A ReplaceAction and DeadlineElapsed can race.

### Correction

Give each ActionSlot an **InputRevision** / compare-and-set input gate.

Replacement command contains:

- action_slot_id;
- expected_slot_input_revision;
- supersedes_submission_id where applicable.

Window close/freeze atomically closes the input gate and captures a **FrozenInputSet**.

Whichever operation successfully linearizes against the current input-gate revision wins.

Do not choose the final submission by sorting wall-clock timestamps.

---

## DC-03 — CommandId alone is insufficient semantic idempotency

### Problem

If a deadline worker accidentally emits two commands with different random CommandIds, both are unique and may both be processed even though they represent the same deadline occurrence.

Also, reuse of the same CommandId with a different payload is not defined.

### Correction

Distinguish:

- `command_key` — semantic idempotency key stable across retries/duplicate delivery;
- optional `attempt_id` — operational delivery attempt.

For system triggers, the command_key must be deterministically derived from the logical trigger, e.g.:

`deadline:<session>:<window>:<deadline-policy-version>`

If the same command_key is received with different semantic payload hash, reject as `IDEMPOTENCY_KEY_REUSE`.

Exact duplicate returns the originally stored CommandResult; it does not create a special gameplay result called DUPLICATE.

---

## DC-04 — Admission preconditions are too broad for same-window dependencies

### Problem

v0.1 validates `admission_predicates` before building the Interaction Graph.

This makes some valid causal merge impossible.

Example:

- Player A hands/allocates a battery.
- Player B uses the battery in the same simultaneous window.

If B's "has battery" check is an admission predicate against base state, B is rejected before the dependency can be resolved.

### Correction

Split conditions into two classes.

### SubmissionEligibility / Admission constraints
Must be true in the frozen base state and cannot be enabled by another action in the same window.

Examples:
- correct participant/role;
- action is offered in the ActionSlot;
- parameter schema;
- target addressability;
- knowledge required to choose/target something.

### ResolutionRequirement
State/resource/capability requirements that may be satisfied, invalidated or transformed by other admitted actions in the same InteractionGroup.

Examples:
- resource available;
- device remains operational;
- target remains at location;
- another action supplies/opens/enables something.

Failure of a ResolutionRequirement normally produces an ActionOutcome, not a rejected submission.

---

## DC-05 — No generic canonical transition record

### Problem

v0.1 makes `ResolutionRecord` central to atomic canonical commits.

But not every canonical batch is a player-action resolution.

Examples:
- initial Scenario Progression;
- resume transition;
- deadline-driven interruption with no admitted action;
- pure scheduled effect between windows;
- explicit administrative compensating transition.

### Correction

Introduce generic:

`CanonicalTransitionRecord`

with TransitionKind such as:

- WINDOW_RESOLUTION;
- PROGRESSION;
- SCHEDULED_EFFECT;
- SESSION_LIFECYCLE;
- DEADLINE;
- ADMIN_COMPENSATION.

A `ResolutionRecord` becomes resolution-specific evidence referenced by a CanonicalTransitionRecord when applicable.

Every canonical event batch is owned by exactly one CanonicalTransitionRecord.

---

## DC-06 — Stream revision convention is too open for contract-level replay

### Problem

The document leaves the exact revision convention open while event identity and before/after state hashes already depend on it.

### Correction

Freeze the contract convention:

- initial Session state is revision 0;
- each canonical Domain Event consumes exactly one revision;
- for a batch against base revision N:
  - event at batch_index 0 has revision N+1;
  - event at batch_index i has revision N+1+i;
  - after-state revision is N + event_count;
- a transition that makes no canonical change emits no batch and does not advance revision.

A completed ResolutionWindow normally emits at least its canonical closure/outcome/progression event(s).

---

## DC-07 — Canonical sets versus ordered lists are unspecified

### Problem

Canonical JSON serialization preserves array order; it does not decide whether an array is semantically a set.

Fields such as:
- actor_refs;
- subject_refs;
- causation_refs;
- recipient refs;
- selected submission ids;

can hash differently if constructed in different iteration orders.

### Correction

Every collection field must declare one of:

- `OrderedList<T>` — order is semantic;
- `CanonicalSet<T>` — duplicates forbidden and values canonical-sorted before hashing;
- `CanonicalMap<K,V>` — deterministic key order in serialization.

Do not rely on incidental database/object iteration order.

---

## DC-08 — SessionState contains its own hash

### Problem

v0.1's logical SessionState contains:

`state_hash`

while also saying state_hash is computed from SessionState.

This is recursively defined unless the field is silently excluded.

### Correction

Split:

### CanonicalStateContent
Hashable gameplay/progression state.

### SessionStateProjection
- session_id;
- stream_revision;
- state_hash;
- version_manifest_ref;
- canonical_state_content.

Define:

`state_hash = H(version_manifest_mechanics_hash || stream_revision || canonical(CanonicalStateContent))`

No self-reference.

---

## DC-09 — VersionManifest over-pins noncanonical presentation runtime

### Problem

v0.1 pins `director_version` and `realizer_version` at Session level.

The accepted architecture requires mechanical replay/version pinning, while presentation output is stored as immutable interaction evidence and may evolve independently.

### Correction

Split:

### MechanicsVersionManifest — Session pinned
Contains engine/scenario/contracts/resolver/progression/hash/RNG versions.

### PresentationGenerationContext — per presentation task/record
Contains:
- director/presentation-policy version;
- realizer version;
- template version;
- provider/model/prompt versions where relevant.

The outbox item that requests presentation generation captures this context at enqueue time so a retry after deployment cannot silently change generation version.

---

## DC-10 — PresentationPlan can accidentally own legal actions

### Problem

v0.1 gives PresentationPlan:

`available_ui_actions[]`

But ADR-012 explicitly prohibits Narrative Direction from adding/removing legal actions.

### Correction

Introduce a deterministic authoritative-safe **PlayerInteractionView** derived before Narrative Direction.

It contains:
- visible world/epistemic projection;
- current DecisionPoint/Window reference;
- mechanical ActionSlots/allowed ActionDefinitions/target affordances.

Narrative Direction receives PlayerInteractionView and produces PresentationPlan.

PresentationPlan may reference the immutable `interaction_surface_hash`, but cannot author the mechanical action set.

The UI renders mechanical affordances from PlayerInteractionView and narrative content from PresentationPlan.

---

## DC-11 — Dynamic propositions are not supported cleanly

### Problem

v0.1 treats `PropositionId` as stable within the Scenario Bundle.

That implies propositions must be predeclared, which fails for runtime-created entities or parameterized statements.

### Correction

Split:

### PredicateDefinition
Compiled in Scenario Bundle.

### PropositionValue
Immutable runtime value object:

```
predicate_ref
arguments[]
value
optional temporal_qualifier
```

Its deterministic `proposition_key` is content-derived/canonical.

Fact/Knowledge/Belief/Suspicion/CommunicationClaim reference PropositionValue/proposition_key.

---

## DC-12 — Epistemic temporal meaning remains ambiguous

### Problem

A character may learn "radio is sabotaged" at t10; the radio is repaired at t20.

A KnowledgeRecord that simply stays valid against a timeless proposition can be misread as knowledge of the current state.

### Correction

PropositionValue supports an explicit optional temporal qualifier, e.g.:

- AT_TICK;
- DURING_RANGE;
- AS_OF_TICK;
- timeless semantic proposition.

Knowledge/belief records retain acquisition/provenance separately from the proposition's temporal meaning.

Ending a FactRecord does not automatically erase historical knowledge.

---

## DC-13 — Communication recipient semantics are conflated with reception

### Problem

`CommunicationClaim.recipient_refs[]` can be interpreted as "these characters heard it."

But addressed audience and actual observation are different.

### Correction

CommunicationClaim records:
- speaker;
- channel;
- addressed/audience target if any;
- asserted PropositionValue.

Actual reception is represented by ObservationRecord and subsequent epistemic changes.

A failed/intercepted communication may therefore exist as an attempted canonical claim without granting observation/knowledge.

---

## DC-14 — Scheduled-effect ordering at a resolution boundary is undefined

### Problem

The architecture says logical scheduled effects cannot independently mutate through an open simultaneous window, but v0.1 does not specify whether a due effect is applied before or after the window's actions when the frontier advances.

### Correction

ResolutionWindow/Progression policy must specify a **BoundaryOrderingPolicy**.

Initial semantic choices:

- DUE_BEFORE_ACTIONS;
- ACTIONS_BEFORE_DUE;
- INTERRUPT_WINDOW.

Exact names may evolve, but ordering must be explicit and compiled, never implicit.

Logical time advancement and due-effect evaluation occur according to that policy in one frontier operation.

---

## DC-15 — Progression convergence/termination is undefined

### Problem

ProgressionTransition can activate another condition, which activates another transition, etc.

The contract says multiple candidates compose/select deterministically but does not define when evaluation stops.

### Correction

Introduce `ProgressionEvaluation`:

- deterministic rounds;
- candidates evaluated from current transient state;
- selected transitions applied;
- repeat until stable, ending/completion, or declared step budget;
- compiler rejects statically detectable illegal cycles where possible;
- runtime non-convergence aborts the uncommitted transition rather than partially committing.

Each bundle declares/compiles a bounded maximum progression-step budget.

---

## DC-16 — DecisionPointDefinition lacks a runtime instance

### Problem

The same DecisionPointDefinition may be activated more than once (loop, retry, recurring crisis), but v0.1 has no runtime `DecisionPointInstance`.

References become ambiguous for replay/presentation/causation.

### Correction

Add:

`DecisionPointInstance`

with deterministic instance key, definition ref, activation revision/tick, status and associated ResolutionWindow instance.

Definitions remain immutable bundle content; instances are canonical progression state.

---

## DC-17 — RNG derivation context is too vague

### Problem

`derivation_context_hash` could accidentally include the entire plan or action ordering.

Then adding an unrelated random decision could still perturb existing draws.

### Correction

Define draw derivation input narrowly:

```
session_seed
rng_algorithm/version
transition_key
rule_ref
draw_key
declared draw parameters/context
```

Do not include unrelated random calls, list position or complete mutable ResolutionPlan hash.

---

## DC-18 — Presentation retry can silently change versions

### Problem

Outbox `BUILD_PRESENTATION` contains generic payload, while realizer/director versions are not frozen in the task.

A commit under deployment A can be retried under deployment B and produce different output before any PresentationRecord exists.

### Correction

The outbox presentation task captures immutable `PresentationGenerationContext` at commit/enqueue time.

Retry uses the same context/version or fails explicitly if unavailable.

Once delivered, exact output is immutable evidence.

---

## DC-19 — Player action provenance should reference what the human actually saw

### Problem

ActionSubmission records source type but not the concrete PresentationRecord/interaction view that led to the choice.

That weakens playtest/debug causality.

### Correction

Human ActionSubmission optionally/typically records:

- source_interaction_view_key;
- source_presentation_id.

These are audit/provenance references, not authority by themselves.

Mechanical authorization still comes from current canonical Window/ActionSlot.

---

## DC-20 — Secret simultaneous pending inputs need an explicit confidentiality invariant

### Problem

Pending submissions are durable but outside canonical state. The contract does not state who may read them before resolution.

A data/API mistake could reveal Player A's secret pending action to Player B without violating canonical world-state visibility rules.

### Correction

Pending input has an explicit access policy.

For SECRET_SIMULTANEOUS:
- submission content is readable only by authoritative server/debug roles until resolution/policy release;
- the other player may receive only specifically allowed readiness/status signals;
- pending input is never included in the other PlayerInteractionView.

---

## DC-21 — AdminRecovery must not become a projection-edit escape hatch

### Problem

AdminRecoveryPrincipal is defined but its canonical constraints are not strong enough.

### Correction

Admin recovery may invoke only explicitly registered recovery/compensation commands that generate canonical events.

No normal recovery path directly patches historical events or the current projection.

Break-glass database repair, if ever needed operationally, is outside game-domain semantics and requires separate runbook/audit policy.

---

# 3. Root causes

## Root cause A — v0.1 modeled records before transition ownership

The contracts defined ResolutionRecord/EventRecord but did not first define the generic canonical transition that owns every batch.

Lesson:
start from the atomic Session transition boundary, then specialize resolution/progression evidence.

## Root cause B — "same base snapshot" was not carried through pending-input concurrency

The resolver itself was deterministic, but input replacement and deadline closure still lacked a deterministic linearization contract.

Lesson:
determinism starts before the resolver.

## Root cause C — eligibility and resolvability were conflated

Preconditions were treated as one category.

Lesson:
a simultaneous causal-merge engine must distinguish "the player was allowed to attempt this" from "the world conditions for success were satisfied."

## Root cause D — canonical serialization was specified without canonical collection semantics

Canonical JSON does not decide set/list semantics.

Lesson:
every hashable collection needs explicit ordering semantics.

## Root cause E — narrative separation stopped one layer too late

Narrative Direction was forbidden from changing mechanics, but PresentationPlan still contained mechanics-shaped action availability.

Lesson:
derive an authoritative PlayerInteractionView before presentation planning.

## Root cause F — static identifiers leaked into runtime semantics

PropositionId and DecisionPointDefinition were treated as if definitions and runtime instances were the same thing.

Lesson:
separate bundle definitions from runtime value/instance identity.

---

# 4. Impact if v0.1 were implemented unchanged

Likely migration/debug problems:

1. window freeze would advance or ambiguously not advance the canonical revision;
2. replacement/deadline races would produce hard-to-replay behavior;
3. duplicated deadline triggers with different IDs could run twice;
4. complementary same-window actions could be rejected before interaction resolution;
5. scheduled effects would have undefined ordering at window boundaries;
6. automatic progression batches would be awkwardly forced into ResolutionRecord;
7. semantically identical event/state structures could hash differently because set ordering differed;
8. SessionState hash computation would require an undocumented self-exclusion;
9. repeated DecisionPoint definitions could not be uniquely referenced;
10. runtime-created propositions would require ad-hoc IDs or predeclaration;
11. stale/noncanonical presentation logic could accidentally hide legal actions;
12. progression rules could loop indefinitely;
13. presentation retries across deployment versions could silently change what players see;
14. secret pending submissions could leak through read APIs;
15. AdminRecovery could become a dangerous bypass if not constrained.

These are cheaper to correct now than after SQL schemas and scenario content exist.

---

# 5. Decision

**DOMAIN_CONTRACTS_v0.1 is not accepted.**

The accepted Architecture Baseline v0.3 and ADR-001 through ADR-014 remain valid.

Proceed with a targeted **DOMAIN_CONTRACTS_v0.2 — PROPOSED** that:

- introduces CanonicalTransitionRecord;
- separates canonical Window state from ResolutionAttempt state;
- linearizes pending input with slot revisions;
- strengthens semantic idempotency;
- splits admission eligibility from resolution requirements;
- freezes stream-revision semantics;
- defines canonical set/list behavior;
- fixes state-hash self-reference;
- separates mechanics/presentation version manifests;
- introduces PlayerInteractionView;
- supports runtime PropositionValue and DecisionPointInstance;
- defines scheduler boundary ordering;
- defines progression convergence;
- narrows RNG derivation;
- pins presentation-generation context;
- strengthens secret-input/recovery invariants.
