# RED-TEAM — Domain Contracts v0.2

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/domain/DOMAIN_CONTRACTS_v0.2.md`  
**Governing architecture:** Architecture Baseline v0.3 — ACCEPTED  
**Verdict:** READY FOR ACCEPTANCE

## Scope

This review checks:

1. compatibility with ADR-001 through ADR-014;
2. the original 18 synthetic contract tests;
3. the additional acceptance-gate edge cases added after the v0.1 postmortem.

No ACCEPTED ADR needs to be superseded.

---

# 1. ADR compatibility

## ADR-001 — Selective Event Sourcing

PASS.

v0.2 keeps canonical Session transitions/events in the event stream while keeping pending inputs, command processing, resolver attempts and presentation artifacts outside canonical world history.

## ADR-002 — Session stream / Single Canonical Frontier

PASS.

v0.2 makes the frontier explicit through:

- fixed OPEN-window base revision;
- WindowInputGate outside canonical state;
- FrozenInputSet;
- one CanonicalTransitionRecord per committed event batch.

## ADR-003 — PostgreSQL baseline

PASS.

No new datastore/broker assumption is introduced.

The contracts remain compatible with one transactional PostgreSQL implementation.

## ADR-004 — Componentized Entity Model

PASS.

No class/ECS-specific behavior is introduced.

## ADR-005 — Command / Action authority

PASS.

v0.2 strengthens this ADR with semantic CommandKey idempotency and resumable CommandProcessingRecord.

## ADR-006 — Claims / Interaction / Resolver

PASS.

Eligibility versus ResolutionRequirement fixes same-window causal dependencies without weakening authoritative claims or bounded resolver strategies.

## ADR-007 — Semantic Event / Mutation IR

PASS.

Generic CanonicalTransitionRecord does not replace semantic events; it only owns the atomic batch.

## ADR-008 — Epistemics

PASS.

Dynamic PropositionValue and communication-reception separation improve the accepted epistemic semantics.

## ADR-009 — Temporal / Scheduler

PASS.

BoundaryOrderingPolicy resolves the previously missing due-effect/action boundary order while preserving three-clock separation.

## ADR-010 — Compiled bundle / pure rules

PASS.

Progression, eligibility, resolution requirements and ordering policy remain compiled/pure.

## ADR-011 — Versioning / hashing / replay

PASS.

v0.2 strengthens deterministic collections, state-hash input and mechanics/presentation version separation.

## ADR-012 — Progression / Narrative separation

PASS.

PlayerInteractionView removes mechanical affordances from Narrative Direction/PresentationPlan ownership.

## ADR-013 — CQRS / projection / outbox

PASS.

CommandProcessingRecord and DeliveryGuard refine application reliability without changing the accepted transactional outbox boundary.

## ADR-014 — LLM boundaries

PASS.

No canonical contract grants LLM authority.

---

# 2. Original 18 synthetic cases

All 18 are now contractually unambiguous at the architecture/contract level.

1. **Scarce battery** — PASS  
   Shared constrained claim domain groups all consumers; aggregate capacity is resolved jointly.

2. **Sabotage radio vs transmit** — PASS  
   INTERFERENCE/ORDER_SENSITIVE interaction can resolve both from the frozen base state.

3. **Independent action permutation** — PASS  
   Interaction groups are deterministic; independent groups must commute.

4. **Hidden EntityId** — PASS  
   Action target must be addressable/offered through authoritative interaction surface.

5. **Replacement immediately before deadline** — PASS  
   SlotInputRevision CAS + atomic WindowInputGate freeze linearizes replacement versus closure without wall-time sorting.

6. **Deadline delivered twice** — PASS  
   Stable semantic CommandKey deduplicates duplicate trigger delivery; payload reuse mismatch is rejected.

7. **Scheduled effect due while Window OPEN** — PASS  
   It cannot commit through the open window independently; BoundaryOrderingPolicy determines behavior at the frontier.

8. **Three-way aggregate capacity conflict** — PASS  
   Group construction is based on shared constrained domain and aggregate validation.

9. **Same-window dependency** — PASS  
   Submission eligibility does not reject a requirement that another admitted action may satisfy; ResolutionRequirement is evaluated jointly.

10. **Lie believed while canon false** — PASS  
    CommunicationClaim, BeliefRecord and FactRecord remain distinct.

11. **Evidence changes belief** — PASS  
    Belief records can supersede/end without changing the historical communication claim.

12. **Fact validity ends; observation remains** — PASS  
    Fact validity lifecycle does not erase ObservationRecord/Knowledge history.

13. **Realizer crash after canonical commit** — PASS  
    Required presentation work survives in transactional outbox.

14. **Outbox retry** — PASS  
    Deduplication key + immutable execution context prevent duplicate/version-drift behavior.

15. **Unrelated RNG draw added** — PASS  
    Named draw derivation excludes list position and unrelated draw context.

16. **Presentation wording changes; world hash unchanged** — PASS  
    Presentation output is outside canonical SessionState/event hash.

17. **Old Session after engine deployment change** — PASS AT CONTRACT LEVEL  
    MechanicsVersionManifest pins mechanical versions; ADR-011 forensic replay remains fallback if obsolete executable support is unavailable.

18. **Ending trigger + other progression trigger simultaneously true** — PASS  
    ProgressionEvaluation requires explicit composition/priority/seeded selection and bounded convergence.

---

# 3. Additional acceptance-gate edge cases

## A. Zero-event/no-op command

PASS.

NO_OP produces no CanonicalTransitionRecord, no event batch and no StreamRevision increment.

This preserves the rule that StreamRevision is canonical-event order, not command count.

## B. Crash after input freeze but before canonical commit

PASS.

Durable state remains:

- canonical Window OPEN at base revision;
- WindowInputGate FROZEN;
- FrozenInputSet immutable;
- CommandProcessingRecord resumable;
- ResolutionAttempt may be ABORTED/retried.

Retry does not re-open input or create a second logical command/transition.

## C. Repeated DecisionPoint activation

PASS.

DecisionPointDefinition and DecisionPointInstance are separated.

The same definition can produce multiple uniquely referenceable runtime instances.

## D. Communication interception / failed delivery

PASS.

CommunicationClaim records the speaker's assertion/channel/addressed audience.

Actual reception requires ObservationRecord.

Therefore an intercepted/failed communication can exist canonically without granting knowledge to the intended target.

## E. Stale presentation worker

PASS.

BUILD_PRESENTATION is pinned to:

- player_view_key;
- state revision;
- PresentationGenerationContext.

DeliveryGuard prevents an old PlayerInteractionView from being delivered as current after it is superseded.

## F. Canonical set/map hashing

PASS.

Replay/hash-relevant collections declare OrderedList, CanonicalSet or CanonicalMap semantics.

Canonical sorting prevents incidental iteration order from changing hashes.

## G. Progression non-convergence

PASS.

ProgressionEvaluation has a compiled step budget.

Exhaustion aborts the uncommitted transition; partial progression does not commit.

---

# 4. Remaining open details

The remaining unfrozen items do not require another contract version before moving to Domain Model design:

- opaque ID encoding;
- Claim.scope concrete representation;
- Mutation target concrete representation;
- event-family registry;
- metric-specific fixed-point scales/units;
- PRNG implementation;
- strategy parameter schemas;
- Relationship/Goal/Commitment concrete contracts;
- authoring DSL syntax;
- SQL tables/indexes;
- snapshot strategy;
- retention policy;
- PlayerInteractionView caching/materialization policy.

These are implementation/domain-model details **provided they preserve the accepted v0.2 contract semantics**.

If any later design conflicts with these semantics, it is not a local implementation choice and must trigger a contract revision or ADR review.

---

# 5. Residual risks

## Risk 1 — Contract surface area

v0.2 is intentionally explicit and introduces more artifacts:

- CommandProcessingRecord;
- WindowInputGate;
- FrozenInputSet;
- CanonicalTransitionRecord;
- PlayerInteractionView;
- PresentationGenerationContext.

This is justified by replay/reliability boundaries, but the persistence model should avoid turning every logical contract into a separate table/service.

## Risk 2 — Authoring complexity

Eligibility rules, ResolutionRequirements, Claims and Progression policies can become author-hostile if surfaced directly.

The future Scenario DSL/Studio should compile simpler authoring concepts into these runtime contracts.

## Risk 3 — Rule model becoming a hidden programming language

The accepted bounded/pure-rule constraint remains essential.

Contract richness must not become arbitrary expression-language complexity.

## Risk 4 — Debug storage volume

Resolution evidence, presentations and command audit data may become large.

Retention/snapshot/debug policies can be optimized later because canonical Session events remain authoritative.

---

# 6. Verdict

**DOMAIN_CONTRACTS_v0.2 is technically READY FOR ACCEPTANCE.**

I do not identify a remaining contract-level defect that justifies producing v0.3 before Domain Model design.

The correct next sequence is:

1. explicit acceptance of Domain Contracts v0.2;
2. mark v0.2 ACCEPTED in GitHub;
3. create `DOMAIN_MODEL_v0.1.md`;
4. red-team aggregate/entities/relationships between contracts;
5. only then propose PostgreSQL schema.

Until explicit acceptance, v0.2 remains PROPOSED.
