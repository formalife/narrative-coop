# ADR-005 — EngineCommand and ActionSubmission Authority Boundary

**Status:** ACCEPTED  
**Date:** 2026-10-02

## Context

Player actions are not the only authoritative inputs. Deadlines, scheduler/progression triggers, resume/recovery operations and retries also require consistent idempotency and authority semantics. Clients must never author canonical claims/effects/permissions.

## Decision

All authoritative runtime inputs enter through idempotent **EngineCommands** with server-bound Principal authority.

Player action input is persisted as **ActionSubmission**, which is pending input rather than canonical world truth.

The client may submit only allowed action type/parameters/targets. It may not author:
- claims;
- permissions;
- preconditions;
- canonical effects;
- resource costs;
- resolver priority.

The immutable ActionDefinition and frozen canonical state deterministically produce the server-authoritative ExpandedAction.

Command and submission identities are immutable/idempotent. Reprocessing cannot duplicate canonical effects.

## Alternatives Considered

1. Client submits fully expanded semantic actions.
2. Separate ad-hoc endpoints for each system/player transition.
3. Unified command boundary with server action expansion.

## Why Rejected

Client-expanded actions create security/replay risks.

Ad-hoc transition paths make idempotency, causation and authorization inconsistent.

## Consequences

Authority is explicit and testable.

Pending submissions can be replaced without pretending each draft is a world event.

## Risks

Command/application-layer design can become bureaucratic if every internal function is unnecessarily modeled as a command.

## Revisit Conditions

Revisit if the command boundary creates measurable complexity without reliability/security benefit, while preserving the server-authority invariant.
