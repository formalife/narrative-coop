# RED-TEAM — Domain Contracts v0.3

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/domain/DOMAIN_CONTRACTS_v0.3.md`  
**Baseline:** Domain Contracts v0.2 — ACCEPTED  
**Verdict:** READY FOR ACCEPTANCE

## Scope

This review verifies that v0.3 is a narrow additive correction rather than an architectural redesign.

It focuses on four amendments:

1. SessionGenesis / session_seed ownership.
2. Revision-0 bootstrap determinism.
3. Predicate epistemic cardinality.
4. InformationClassification and terminal ending/window consistency.

## 1. Compatibility with accepted ADRs

### ADR-001 — Selective Event Sourcing

PASS.

SessionGenesis is immutable bootstrap metadata, not a substitute for event history. Canonical changes after revision 0 remain event-sourced.

### ADR-002 — Single Canonical Frontier

PASS.

No new writer/frontier is introduced.

Dynamic bindings/random initialization are explicitly moved to canonical revision-1+ transitions where needed.

### ADR-005 — Command/Action authority

PASS.

SessionGenesis does not allow clients to author gameplay effects.

### ADR-008 — Epistemic model

PASS and improved.

InformationClassification formalizes Secret/access semantics without collapsing it into Fact/Knowledge.

Predicate cardinality prevents ambiguous active belief states.

### ADR-009 — Temporal/Scheduler

PASS.

No time-boundary semantics are weakened.

### ADR-011 — Versioning/Replay

PASS and required.

Session seed now has an explicit durable owner.

Revision-0 reconstruction is defined.

### ADR-012 — Progression/Narrative separation

PASS.

Terminal ending/window consistency remains canonical progression responsibility.

## 2. Bootstrap replay tests

### Test A — reconstruct revision 0

Inputs:

- pinned CompiledScenarioBundle;
- SessionGenesis.

Expected:

- same CanonicalStateContent revision 0;
- same initial state hash.

PASS contractually.

### Test B — random scenario setup

v0.3 forbids hidden random bootstrap mutation.

Random setup must be first canonical transition with NamedRandomDraw evidence.

PASS.

### Test C — participant binding at startup

Any dynamic binding established after revision 0 is canonical/evented.

No mutable account lookup is required to reconstruct revision 0.

PASS.

### Test D — session_seed durability

NamedRandomDraw derivation explicitly resolves seed from SessionGenesis.

PASS.

## 3. Epistemic tests

### Single-value belief

A predicate configured SINGLE_VALUE cannot leave one subject with two simultaneously active incompatible values for the same argument/temporal scope.

PASS at contract/model boundary.

### Multi-hypothesis belief

A predicate configured MULTI_HYPOTHESIS may allow multiple active candidate PropositionValues.

PASS.

### Secret classification

InformationClassification can constrain visibility policy without creating/removing Fact or Knowledge.

PASS.

## 4. Terminal consistency test

If an ending is reached while a Window is active, the same frontier operation must resolve/cancel/interrupt the Window and prevent an ACCEPTING input gate from surviving terminal Session state.

PASS.

## 5. Residual risks

The amendments add no new subsystem, datastore or runtime authority.

Remaining risks are implementation/detail risks:

- concrete policy schemas;
- classification-policy authoring UX;
- predicate scope normalization;
- exact seed representation/entropy requirements.

These do not block contract acceptance.

## 6. Verdict

**Domain Contracts v0.3 is READY FOR ACCEPTANCE.**

It is a compatible additive supersession of v0.2.

No ADR needs to be superseded.

Until explicit acceptance, v0.2 remains the accepted contract baseline and v0.3 remains PROPOSED.
