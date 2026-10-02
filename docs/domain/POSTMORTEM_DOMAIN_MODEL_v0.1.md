# POSTMORTEM — Domain Model v0.1

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/domain/DOMAIN_MODEL_v0.1.md`  
**Governing architecture:** Architecture Baseline v0.3 — ACCEPTED  
**Governing contracts:** Domain Contracts v0.2 — ACCEPTED  
**Verdict:** MODIFY

## Executive verdict

Domain Model v0.1 correctly identifies the major ownership zones and avoids turning logical contracts directly into SQL tables/services.

However, the red-team found one material gap in the accepted Domain Contracts and several domain-model defects that should be corrected before the model is accepted.

### Contract-level gap

**DM-001 — Session seed has no explicit accepted owner/contract.**

ADR-011 requires a Session to pin RNG algorithm/version **and Session seed**. Domain Contracts v0.2 uses `session_seed` in NamedRandomDraw derivation but does not define where the seed is durably stored.

This is not safe to leave as an implementation detail.

**Required action:** propose Domain Contracts v0.3 with an explicit immutable SessionGenesis/Determinism record containing the seed and bootstrap hashes.

No accepted ADR needs to be superseded.

### Domain-model corrections

The model should be revised to:

- separate canonical participant binding from transient connection presence;
- introduce an immutable SessionGenesis bootstrap owner;
- remove derivable indexes from hash-authoritative state;
- add explicit Secret/Information Classification modeling;
- remove ambiguous `PROPOSED` state from canonical CommitmentInstance;
- tighten active epistemic-state uniqueness/conflict semantics;
- forbid duplicated derived Goal progress;
- require ending transitions to close/interrupt any active Window atomically.

---

# 1. Red-team results

## DM-001 — Session seed ownership is missing from accepted contracts

### Finding

DOMAIN_MODEL_v0.1 placed `session_seed` under SessionIdentity because deterministic RNG cannot be reconstructed without it.

Accepted ADR-011 explicitly requires the seed.

Domain Contracts v0.2's NamedRandomDraw formula also uses `session_seed`.

But no accepted contract says where the seed lives.

### Risk

An implementation could:

- store it only in environment/config;
- regenerate it;
- place it in a transient resolver object;
- omit it from forensic exports.

Any of these breaks historical RNG verification.

### Correction

Add immutable `SessionGenesis` to Domain Contracts v0.3:

```
SessionGenesis
  session_id
  mechanics_version_manifest_ref/hash
  scenario_bundle_hash
  session_seed
  initial_canonical_state_hash
  created_at // operational
```

The seed is immutable and durably retained for the Session.

Revision 0 state is defined from the pinned bundle + SessionGenesis bootstrap.

---

## DM-002 — Canonical ParticipationState contains transient connection status

### Finding

v0.1 proposed ParticipantSlotBinding states including:

- DISCONNECTED.

A socket/network disconnect is operational presence, not automatically canonical gameplay state.

### Risk

Temporary connection instability would pollute canonical replay or create non-deterministic Session state.

### Correction

Canonical ParticipantBinding status should describe gameplay assignment:

- UNBOUND;
- BOUND;
- LEFT/RELEASED if mechanically relevant.

Realtime presence/connectivity belongs to an operational PresenceState.

If a disconnect causes gameplay consequences, a timeout/system EngineCommand creates an explicit canonical transition such as PAUSE, DEFAULT_ACTION or ABANDON according to policy.

---

## DM-003 — Session bootstrap/reconstruction boundary is underspecified

### Finding

v0.1 says reconstruction requires:

`CompiledScenarioBundle initial state + Session identity/participation initialization + events`

but does not define which initialization data is immutable bootstrap truth versus event-sourced progression.

### Correction

Use:

1. immutable `SessionGenesis`;
2. bundle-defined initial canonical state at revision 0;
3. canonical Session events revision 1+.

Participant joins/role bindings after creation are canonical lifecycle events unless fully established in Genesis by Session creation policy.

This makes reconstruction inputs explicit.

---

## DM-004 — Derivable indexes should not be hash-authoritative canonical content

### Finding

v0.1 places structures such as:

- active_fact_index;
- active knowledge index;
- scheduler due_index;

near/in the canonical ledgers.

These duplicate facts already derivable from source records.

### Risk

A reducer bug can produce two different state hashes for semantically identical ledgers because a cached index drifted.

### Correction

Separate:

### CanonicalStateContent
Only semantic source state.

### SessionDerivedIndexes
Rebuildable acceleration structures excluded from state hash.

Examples:
- facts active-by-proposition;
- knowledge active-by-subject/proposition;
- scheduler due index;
- relationship lookup indexes.

Validators may compare derived index correctness in debug mode, but replay truth does not depend on them.

---

## DM-005 — Secret / Information Classification was omitted

### Finding

Accepted ADR-008 explicitly retains Secret as a visibility/access classification rather than a truth category.

Domain Model v0.1 models epistemic records but has no owner for active/static information classification.

### Correction

Introduce:

`InformationClassificationState`

with records such as:

```
InformationClassification
  classification_id
  subject_ref // proposition, evidence, entity data, thread
  policy_ref
  lifecycle/status
  optional reveal_state
```

Important:

- classification does NOT itself grant/revoke Knowledge;
- actual player view remains derived from epistemics + permissions/policy;
- a "secret" is not a hidden Fact type.

Static classifications may originate in the Scenario Bundle; runtime reveal/reclassification state is canonical only if future rules depend on it.

---

## DM-006 — Canonical CommitmentInstance should not start at PROPOSED by default

### Finding

v0.1 gives CommitmentInstance status `PROPOSED`.

A proposed deal/promise may simply be:

- CommunicationClaim;
- ActionSubmission;
- social offer entity/state;
- DecisionPoint/progression state.

It is not necessarily an active canonical obligation.

### Risk

Commitment semantics become a catch-all for negotiation.

### Correction

Canonical CommitmentInstance begins when the scenario considers an obligation mechanically established.

Suggested status:

- ACTIVE;
- FULFILLED;
- BREACHED;
- CANCELLED;
- EXPIRED.

If a scenario requires explicit offers/negotiation as mechanics, model `CommitmentOffer` separately via scenario-defined Entity/Component or a future generic contract rather than overloading CommitmentInstance.

---

## DM-007 — Epistemic current-state uniqueness needs clearer source semantics

### Finding

v0.1 proposes active indexes and discusses replacing provenance.

For Knowledge especially, repeatedly creating active KnowledgeRecords for the same subject/proposition can produce ambiguity.

### Correction

Canonical ledgers keep record history, but current semantic state follows explicit rules.

### Knowledge

At most one active semantic knowledge state for:

`(knower, proposition_key)`.

Additional provenance does not imply "more known."

If provenance itself is mechanically important, emit a new provenance/observation relation or superseding KnowledgeRecord under a deterministic policy.

### Belief / Suspicion

Conflict cardinality is controlled by PredicateDefinition:

- SINGLE_VALUE belief scope: at most one active value per subject/predicate+arguments+temporal scope;
- MULTI_HYPOTHESIS: multiple active PropositionValues allowed.

This policy belongs in compiled epistemic/predicate metadata.

---

## DM-008 — Goal progress risks duplicated derived truth

### Finding

GoalInstance contains optional `progress_state` while GoalDefinition can derive success/failure from world state.

### Risk

Stored progress and actual canonical world state can diverge.

### Correction

A GoalInstance stores progress only when progress is itself an explicit canonical state variable required by mechanics.

If progress is purely derivable, it belongs in a GoalView/projection, not CanonicalStateContent.

Same rule applies to NarrativeThread stage/priority if derivable.

---

## DM-009 — Ending transition must resolve active Window consistency

### Finding

v0.1 allows EndingState REACHED but does not state what happens if a ResolutionWindow is active.

### Correction

Post-state invariant:

> A Session cannot commit `COMPLETED` / terminal EndingState while retaining an OPEN canonical ResolutionWindow.

The same canonical TransitionPlan must:

- resolve;
- cancel;
- or interrupt

the Window as dictated by progression policy.

No orphan active input gate may remain accepting after terminal Session state.

---

## DM-010 — Pending-input archive semantics need an explicit replay distinction

### Finding

v0.1 says nonselected submissions may be deleted after selected evidence is retained.

This is correct for **state replay**, but full forensic/player-input audit may require more.

### Correction

Define two retention guarantees:

### Required forever/compatibility-window for mechanical verification
- FrozenInputSet selected submissions;
- ResolutionRecord;
- canonical transitions/events;
- RNG evidence required by policy.

### Optional/retention-policy audit
- replaced/nonselected submissions;
- raw natural-language input;
- intermediate parser candidates.

Privacy policy may delete optional audit artifacts without breaking state replay.

---

# 2. Synthetic model tests

## 1. State reconstruction without operations stores

v0.1: PARTIAL.

Needs SessionGenesis and removal of derived-index dependency.

## 2. External account changes during Session

PASS with correction.

Opaque ParticipantBinding prevents role authority from following mutable account profile.

## 3. Deactivated Entity referenced by historical knowledge

PASS.

Historical EntityId remains valid/referenceable.

## 4. Directed vs undirected Relationship identity

PASS.

Definition directionality is explicit.

## 5. Duplicate active KnowledgeRecord

PARTIAL FAIL.

Needs the uniqueness/current-state rule in DM-007.

## 6. Contradictory beliefs

PARTIAL.

Needs PredicateDefinition belief cardinality policy.

## 7. Proposed commitment never accepted

FAIL in v0.1 semantics.

Correct by separating proposal/offer from established CommitmentInstance.

## 8. Goal progress drift

PARTIAL FAIL.

Correct by storing only irreducible canonical progress.

## 9. Repeated DecisionPointDefinition

PASS.

Definition/Instance split handles this.

## 10. Schedule fires at same frontier as ending

PASS if TransitionPlan applies BoundaryOrderingPolicy + progression and post-state invariant.

## 11. Ending reached with active Window

FAIL/AMBIGUOUS.

Requires DM-009.

## 12. Delete nonselected submissions

PASS for state replay; retention distinction required for forensic audit.

## 13. Runtime proposition references deactivated Entity

PASS.

Historical references remain semantically resolvable.

## 14. PlayerInteractionView changes with same StreamRevision

PASS.

Pending-input hash/revision can change PlayerViewKey independently of canonical revision.

## 15. Rebuild state hash without Transition/Presentation stores

PARTIAL PASS.

Requires explicit SessionGenesis + canonical events; derived indexes must not be hash inputs.

## 16. RNG verification

FAIL under accepted contract as written.

Session seed owner must be added.

---

# 3. Root causes

## Root cause A — Bootstrap state was treated as obvious

Architecture focused on event history after Session creation.

Replay requires explicitly versioned/bootstrap data before event 1.

## Root cause B — Projection accelerators leaked into domain truth

Useful indexes were placed too close to canonical ledgers.

The model needs a hard line between semantic source state and derived indexes.

## Root cause C — Operational presence looked like gameplay participation

A participant being assigned to a role is canonical; a websocket being disconnected is not.

## Root cause D — Generic concepts were widened too early

Commitment `PROPOSED` and optional goal progress make the model more universal but less precise.

The engine should model only mechanically meaningful canonical concepts.

---

# 4. Verdict

**DOMAIN_MODEL_v0.1 is NOT ready for acceptance.**

No accepted architecture ADR must change.

However, accepted Domain Contracts v0.2 need one additive clarification for deterministic bootstrap/RNG ownership.

Required next artifacts:

1. `DOMAIN_CONTRACTS_v0.3.md` — PROPOSED, minimal contract correction:
   - SessionGenesis;
   - durable session_seed ownership;
   - revision-0 bootstrap definition.

2. `DOMAIN_MODEL_v0.2.md` — PROPOSED:
   - corrected participation/presence split;
   - SessionGenesis;
   - SessionDerivedIndexes outside hashable state;
   - InformationClassificationState;
   - tightened epistemic cardinality;
   - corrected Commitment/Goal semantics;
   - ending/window consistency invariant.

Do not proceed to SQL schema until:
- Domain Contracts v0.3 is explicitly accepted;
- Domain Model v0.2 is red-teamed and accepted.
