# ADR-007 — Semantic Domain Events and State Mutation IR

**Status:** ACCEPTED  
**Date:** 2026-10-02

## Context

Canonical history must preserve semantic meaning for narrative/replay/debugging, but reducers also need a bounded mechanical representation. A single generic StateChanged event loses meaning; one bespoke reducer for every narrative verb creates unbounded engine code.

## Decision

Separate:

1. **Semantic Domain Event** — records what canonically happened.
2. **Typed State Mutation IR** — records/applies validated canonical projection changes.

Each event uses:
- finite engine-level `event_family`;
- precise core/scenario `event_code`;
- typed versioned payload;
- optional validated mutations.

Mutation IR remains small, typed and protected against arbitrary paths into engine state.

Canonical history is not hard-deleted. Time-varying truth/state closes/supersedes validity where appropriate.

## Alternatives Considered

1. Generic state-diff events only.
2. One engine event/reducer type per scenario verb.
3. Semantic event + typed mutation layer.

## Why Rejected

Generic diffs lose narrative/domain intent.

Bespoke reducers make new scenarios require engine code and complicate historical compatibility.

## Consequences

Narrative/debug systems retain semantic causality while replay can use bounded typed transitions.

Mutation IR needs strict schema/version control.

## Risks

Duplicating semantic payload and mutations can drift if compiler/runtime validation is weak.

## Revisit Conditions

Revisit if mutation IR becomes too generic/unsafe or if semantic events repeatedly cannot map cleanly to bounded transitions.
