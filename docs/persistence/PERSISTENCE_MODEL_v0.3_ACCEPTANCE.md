# Persistence Data Model v0.3 Acceptance

**Date:** 2026-10-02  
**Decision:** ACCEPTED  
**Accepted model:** PERSISTENCE_MODEL_v0.3  
**Supersedes:** PERSISTENCE_MODEL_v0.2

## Accepted persistence boundaries

- PostgreSQL as one initial persistence system.
- Global DB lock ordering.
- SessionRuntime as current-state/frontier anchor.
- Immutable canonical scenario/genesis/event history.
- Command-processing fencing generation.
- Outbox lease fencing generation.
- ActionSubmission content hashes.
- FrozenInputSet commits to selected input hashes.
- Stable PresentationId and async build/delivery split.
- Immutable presentation asset identity.
- Event-index metadata remains non-authoritative.
- No initial partitioning or historical snapshots.
- Provider-neutral persistence semantics.

## Evidence

- `RED_TEAM_PERSISTENCE_MODEL_v0.3.md`
- `POSTMORTEM_PERSISTENCE_MODEL_v0.1.md`
- `RED_TEAM_PERSISTENCE_MODEL_v0.2.md`

This acceptance authorizes concrete PostgreSQL schema design, not production migrations by itself.
