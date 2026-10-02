# ADR-011 — Versioning, Canonical Serialization, Hashing and Replay

**Status:** ACCEPTED  
**Date:** 2026-10-02

## Context

Deterministic replay is a core architectural requirement. Algorithmic determinism alone is insufficient if event identity, numeric representation, serialization or historical runtime versions drift.

## Decision

Each Session pins enough immutable/versioned material to reproduce or forensically inspect its mechanics, including:

- engine build/artifact id;
- source commit for traceability;
- scenario id/version/bundle hash;
- command/action/event/state/mutation contract versions;
- resolver/progression rule bundle version;
- canonicalization/hash version;
- RNG algorithm/version + Session seed;
- relevant Director/Realizer/template/prompt versions for interaction records.

Canonical mechanics avoid unconstrained floating-point; use deterministic numeric forms such as integers/fixed-point/declared units.

Hash only deterministic semantic structures after declared canonical serialization.

Operational metadata such as database surrogate IDs and wall-clock `recorded_at` is excluded from semantic event-batch equality/hash.

Maintain distinct replay modes:

1. State replay.
2. Resolver verification replay.
3. Forensic replay.
4. Narrative replay from stored PresentationRecords.

Historical stored event bytes are immutable. Upcasters/migrations change read interpretation, not the original record.

## Alternatives Considered

1. Best-effort replay using current code.
2. Store only final state snapshots.
3. Explicit version/hash/replay contract.

## Why Rejected

Current-code replay and snapshots cannot reliably prove historical causality/determinism.

## Consequences

Build/version retention and compatibility tooling become architectural responsibilities.

## Risks

Long-term historical compatibility can become expensive.

## Revisit Conditions

Revisit support windows/policies when real operational cost is known, while preserving forensic/state reconstruction guarantees.
