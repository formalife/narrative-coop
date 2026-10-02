# ADR-009 — Temporal and Scheduler Model

**Status:** ACCEPTED  
**Date:** 2026-10-02

## Context

The engine needs simultaneous/sequential actions, deadlines, logical delayed consequences and replay. Real time, in-world time and canonical ordering must not be conflated.

## Decision

Keep three distinct time concepts:

1. **Engine order** — Session stream revision + batch index.
2. **Logical time** — deterministic in-world integer/tick representation.
3. **Wall clock** — operational/UX/deadline time.

Wall-clock values never directly define historical state transitions. A deadline produces an idempotent EngineCommand processed through the Session frontier.

Logical delayed consequences are represented by canonical Scheduler records and evaluated at deterministic progression boundaries.

Interrupting scheduled effects require explicit scenario policy for how the current window is closed/interrupted.

Do not rely on platform-local timezone/DST behavior for canonical mechanics.

## Alternatives Considered

1. One shared timestamp model.
2. Real-time timers directly mutating state.
3. Explicit three-time model with canonical scheduler.

## Why Rejected

Shared/real-time models break deterministic replay and create ambiguous simultaneous ordering.

## Consequences

Replay is independent from current clock/timezone.

Scenario authoring must map human-facing time to canonical logical units.

## Risks

Complex scenarios may need richer calendars/durations.

## Revisit Conditions

Revisit if a real scenario requires calendar semantics that cannot be represented cleanly as deterministic logical time plus presentation formatting.
