# RED-TEAM — Domain Model v0.2

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/domain/DOMAIN_MODEL_v0.2.md`  
**Governing architecture:** Architecture Baseline v0.3 — ACCEPTED  
**Accepted contracts:** Domain Contracts v0.2 — ACCEPTED  
**Required amendment:** Domain Contracts v0.3 — PROPOSED  
**Verdict:** READY FOR ACCEPTANCE AFTER Domain Contracts v0.3

## 1. Ownership review

PASS.

Every major state family has one clear owner:

- immutable definitions -> CompiledScenarioBundle;
- revision-0 bootstrap -> SessionGenesis;
- canonical gameplay/progression -> SessionEventStream / SessionStateProjection;
- pending input -> PendingInputAggregate;
- transition evidence -> TransitionEvidenceStore;
- presentation evidence -> Presentation domain;
- connectivity/outbox/logging -> Operations.

No new competing world-state authority was found.

## 2. Reconstruction review

Required reconstruction inputs are:

```
CompiledScenarioBundle
+ SessionGenesis
+ SessionEventStream
```

Operational/presentation stores are not required.

PASS, conditional on Domain Contracts v0.3 acceptance.

## 3. Participation / presence

PASS.

Role/character assignment remains canonical.

Socket presence/disconnection is operational.

Gameplay consequences of disconnection must enter through EngineCommand and canonical transition.

## 4. Entity / relationship model

PASS.

Componentized Entities preserve ADR-004.

Relationship is modeled separately because endpoint/directionality invariants are distinct from ordinary Entity identity.

No graph database implication is introduced.

## 5. Epistemic model

PASS after v0.2 corrections.

- PropositionKey is content-stable.
- Fact history is not erased.
- Observation != Knowledge.
- Communication assertion != reception.
- Knowledge current-state uniqueness is explicit.
- Belief/Suspicion multiplicity is scenario-defined.
- InformationClassification models Secret/access independently from truth.

## 6. Goal / commitment / thread model

PASS with intentional narrowness.

- Goal progress is stored only when irreducible canonical state.
- Commitment begins at mechanically established obligation; proposed offers are not overloaded into CommitmentInstance.
- NarrativeThread state is canonical only when rules consume it.

This avoids turning the engine into a universal social simulator.

## 7. Scheduler / progression

PASS.

- scheduled effects have one lifecycle;
- derived due index is noncanonical;
- ending requires active Window/input-gate closure in same transition;
- Progression convergence remains governed by accepted contracts.

## 8. Derived-index review

PASS.

Derived indexes are explicitly outside CanonicalStateContent hash.

This eliminates duplicate semantic truth while still allowing optimized persistence later.

## 9. Retention/replay review

PASS.

The model distinguishes:

- minimum mechanical reconstruction data;
- verification/forensic evidence;
- player-experience evidence;
- optional privacy/audit artifacts.

This allows privacy-driven deletion of nonselected/raw input without breaking state replay.

## 10. Synthetic edge cases

### Account profile changes mid-session

PASS.

Opaque ParticipantBinding remains stable.

### Entity deactivated but referenced historically

PASS.

Historical references remain valid.

### Duplicate active knowledge

PASS.

Rejected by semantic uniqueness.

### Multi-hypothesis beliefs

PASS when PredicateDefinition permits.

### Commitment offer never accepted

PASS.

No CommitmentInstance is created merely because an offer exists.

### Goal progress drift

PASS.

Derived progress is not duplicated.

### Repeated DecisionPoint definition

PASS.

Instances are separate.

### Ending with open Window

PASS.

Terminal invariant closes/intercepts the active interaction state atomically.

### Secret classification

PASS.

Classification affects view policy; it does not fabricate Knowledge.

### Rebuild state without transition/presentation/outbox stores

PASS.

SessionGenesis + bundle + event stream are sufficient.

## 11. Residual open design details

Not blockers for Domain Model acceptance:

- exact relationship component catalog;
- exact commitment terms schema;
- exact goal progress schemas when irreducible;
- ID encoding;
- persistence normalization/JSONB choices;
- cache/materialization choices;
- retention windows.

These belong to persistence/schema design.

## 12. Verdict

**DOMAIN_MODEL_v0.2 is technically READY FOR ACCEPTANCE once Domain Contracts v0.3 is accepted.**

No further Domain Model v0.3 is justified at this stage.

The correct next sequence is:

1. explicitly accept Domain Contracts v0.3;
2. mark Domain Model v0.2 ACCEPTED;
3. move to persistence/data model proposal;
4. red-team persistence design;
5. only then create SQL migrations/implementation skeleton.
