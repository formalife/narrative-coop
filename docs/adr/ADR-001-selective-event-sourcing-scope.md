# ADR-001 — Selective Event Sourcing Scope

**Status:** ACCEPTED  
**Date:** 2026-10-02

## Context

The product requires causal reconstruction, deterministic replay, debugging, recap generation and explanation of how a shared canonical timeline emerged. Applying Event Sourcing to every subsystem would add unnecessary complexity.

## Decision

Use Event Sourcing for the canonical history of each gameplay Session.

Event Source:
- canonical world/session progression;
- canonical Domain Events;
- logical scheduler/progression consequences;
- epistemic and relationship changes when they are part of canonical gameplay.

Do not Event Source by default:
- accounts/profiles;
- entitlements;
- product analytics;
- mutable scenario drafts;
- media-generation jobs;
- temporary UI state;
- ordinary technical logs.

Pending ActionSubmissions and command journal entries are durable input/audit records, not canonical world history until selected/resolved.

Snapshots are optimization only and never replace the canonical stream as historical source.

## Alternatives Considered

1. Full Event Sourcing for the entire product.
2. CRUD-only persistence for gameplay state.
3. Selective Event Sourcing for Session canon.

## Why Rejected

Full Event Sourcing expands operational/schema complexity without product value in non-gameplay subsystems.

CRUD-only gameplay state loses the historical causal record needed for replay/debug/recap.

## Consequences

The Session event stream becomes the canonical historical source for gameplay.

Current state and secondary read models must be rebuildable.

Schema/version discipline is required for historical events.

## Risks

Event evolution and replay compatibility can become costly if event semantics are poorly versioned.

## Revisit Conditions

Revisit if the canonical stream becomes operationally unmanageable or if some currently non-event-sourced subsystem develops a demonstrated requirement for historical causality/replay.
