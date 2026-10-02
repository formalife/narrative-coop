# ADR-004 — Componentized Entity Model

**Status:** ACCEPTED  
**Date:** 2026-10-02

## Context

Scenarios need reusable structured entities without a heavyweight game-engine ECS or scenario-specific object/class hierarchy.

## Decision

Represent world entities as stable Entity identities with typed, versioned Components.

Use a **componentized domain entity model**, not a performance-oriented ECS runtime.

Behavior is defined by scenario rules/action definitions/resolver logic rather than hidden inside scenario-specific entity classes.

Canonical entities are normally deactivated/tombstoned instead of hard-deleted so historical references remain valid.

## Alternatives Considered

1. Heavyweight ECS framework.
2. Scenario-specific class inheritance.
3. Componentized domain model.

## Why Rejected

A full ECS adds scheduling/query/system machinery optimized for real-time simulation that is not currently required.

Scenario-specific classes make the engine less data-driven and increase code changes per episode.

## Consequences

Entity state remains structured and extensible.

Component schemas require explicit versioning/validation.

## Risks

Overusing generic components could recreate an untyped JSON soup.

## Revisit Conditions

Revisit only if measured runtime requirements demonstrate a need for ECS-style data-oriented processing or if the component model cannot express multiple real scenarios cleanly.
