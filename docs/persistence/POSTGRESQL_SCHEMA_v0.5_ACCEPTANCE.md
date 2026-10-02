# PostgreSQL Schema v0.5 Acceptance

**Date:** 2026-10-02  
**Decision:** ACCEPTED  
**Accepted schema:** POSTGRESQL_SCHEMA_v0.5  
**Supersedes:** POSTGRESQL_SCHEMA_v0.4

## Accepted baseline

- internal `engine` schema;
- concrete table/key/constraint layout;
- canonical bytes/hash representation;
- command/input/outbox fencing persistence;
- composite presentation context integrity;
- PresentationRecord lifecycle guard;
- least-privilege logical DB roles;
- provider-neutral migration archive;
- migration bootstrap/default-privilege ordering.

## Evidence

- `RED_TEAM_POSTGRESQL_SCHEMA_v0.5.md`
- `POSTMORTEM_POSTGRESQL_SCHEMA_v0.1.md`
- `POSTMORTEM_POSTGRESQL_SCHEMA_v0.2.md`
- `POSTMORTEM_POSTGRESQL_SCHEMA_v0.3.md`
- `POSTMORTEM_POSTGRESQL_SCHEMA_v0.4.md`

## Implementation gate

Acceptance does not yet make production migrations implementation-ready.

Before executable migrations:
1. choose/test target PostgreSQL major/provider;
2. map migration/runtime/worker roles;
3. accept guest identity/participation;
4. freeze exposed-schema/RLS strategy.
