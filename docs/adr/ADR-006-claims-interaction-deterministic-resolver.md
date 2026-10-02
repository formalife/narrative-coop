# ADR-006 — Claims, Interaction Graph and Deterministic Resolver

**Status:** ACCEPTED  
**Date:** 2026-10-02

## Context

The engine must resolve simultaneous actions that may contend, exclude, interfere, enable, complement or depend on one another. A fixed global verb ontology or arbitrary rule scripting would either be too rigid or become a universal programming environment.

## Decision

ActionDefinitions derive typed Claims against a frozen base state.

Claims describe contention-relevant access/demand using modes such as READ, WRITE, CONSUME, RESERVE, PRODUCE and EXCLUSIVE.

Admitted actions form a typed **Interaction Graph** with edge semantics including:
- CONTENTION;
- EXCLUSION;
- INTERFERENCE;
- DEPENDENCY;
- COMPLEMENTARITY;
- ORDER_SENSITIVE.

Connected interaction groups are resolved jointly.

Use bounded, versioned deterministic resolver strategies plus compiled declarative scenario interaction transforms.

Scenario runtime cannot execute arbitrary scenario-provided JavaScript/code.

Network arrival order and incidental UUID ordering are never semantic priority rules.

## Alternatives Considered

1. Global action/verb ontology driving mechanics.
2. Arbitrary scenario scripts.
3. General-purpose constraint solver.
4. Typed claims + bounded strategies.

## Why Rejected

The first is too rigid, while scripts/constraint solvers make determinism, validation and authoring substantially harder.

## Consequences

Most scenarios should be expressible through data/configuration.

Resolver behavior is property-testable.

Some exceptional interactions require explicit compiled transforms.

## Risks

The claim/strategy model may prove insufficient across diverse real scenarios.

## Revisit Conditions

Revisit if multiple concrete scenarios repeatedly require engine-specific code or if bounded strategies become an opaque DSL/programming language.
