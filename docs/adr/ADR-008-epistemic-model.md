# ADR-008 — Epistemic Model

**Status:** ACCEPTED  
**Date:** 2026-10-02

## Context

The product depends on asymmetric information. Fact, observation, knowledge, belief, suspicion, communication and secrecy must remain semantically distinct so that player views, lies, evidence and causal consequences can be modeled without conflation.

## Decision

Use explicit structures for:

- Proposition;
- Fact;
- ObservationRecord;
- KnowledgeRecord;
- BeliefRecord;
- SuspicionRecord;
- CommunicationClaim;
- Evidence relationship;
- Secret/visibility classification where needed.

A proposition carries explicit predicate/arguments/value/polarity.

Unknown is normally the absence of an applicable positive epistemic record, not a positive stance.

Believing `not P` is represented explicitly and is not equivalent to merely disbelieving `P`.

Facts use provenance and validity lifecycle; ordinary world evolution ends/supersedes validity rather than erasing historical truth.

Do not mirror every component field as a Fact. Model a Fact explicitly when it needs epistemic/narrative/evidence reference.

No automatic logical closure is performed unless a deterministic scenario rule explicitly grants the inference.

## Alternatives Considered

1. Single stance enum over propositions.
2. Treat player-visible information as simple event visibility flags.
3. Explicit epistemic records.

## Why Rejected

The first two collapse distinctions needed for lies, contradictory beliefs, evidence and asymmetric player views.

## Consequences

Visibility/player projection logic can be deterministic and auditable.

The data model is more verbose than a single knowledge set.

## Risks

Over-modeling epistemics could increase authoring burden.

## Revisit Conditions

Revisit if real scenarios show that some structures can safely be collapsed without losing required semantics.
