# ADR-002 — Session Stream and Single Canonical Frontier

**Status:** ACCEPTED  
**Date:** 2026-10-02

## Context

The defining mechanic requires simultaneous/asymmetric actions to merge into one canonical timeline without network arrival order or competing writers becoming gameplay rules.

## Decision

A Session is the primary runtime consistency boundary and owns one ordered canonical event stream.

A Session advances through a **Single Canonical Frontier**: at most one canonical state-changing resolution/progression operation advances the Session at a time.

While a simultaneous ResolutionWindow is open:
- pending ActionSubmissions may be created/replaced;
- canonical gameplay state remains fixed at the window base revision;
- unrelated canonical gameplay transitions do not independently commit through the open window.

Wall-clock deadlines and system triggers enter as commands and are serialized through the same frontier.

Expected-revision optimistic concurrency remains a safety mechanism rather than normal gameplay arbitration.

## Alternatives Considered

1. Independent entity/character event streams.
2. Multiple concurrent world writers with conflict resolution at persistence time.
3. Single Session stream/frontier.

## Why Rejected

Multiple streams/writers move the core causal-merge problem into distributed consistency and make replay/order semantics harder without current scale justification.

## Consequences

Canonical ordering is simple and explicit.

Simultaneous actions can use frozen-set semantics.

Some high-frequency future workloads may eventually require partitioning.

## Risks

A single Session stream could become a bottleneck for very long-lived worlds, many players or high-frequency autonomous simulation.

## Revisit Conditions

Revisit after measured contention or if product scope expands to many players, long persistent worlds or high-frequency autonomous agents.
