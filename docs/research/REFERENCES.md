# Research References

This file is the technical index for external research relevant to architecture decisions.

## Storage rule

- Large source material, papers, screenshots, recordings and reference assets live in Google Drive under `Narrative Co-op/01_Research`.
- GitHub contains concise findings, primary links and architecture implications.
- A research reference does not become an architectural decision until explicitly accepted and captured in an ACCEPTED ADR.

Google Drive research folder:
https://drive.google.com/drive/folders/1GA18zY0rfb-FrFWqYWM9WI74wZJlWhpP

## Verified architecture references

### Microsoft Azure Architecture Center — Event Sourcing
https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing

FACT:
Microsoft documents Event Sourcing as valuable for intent/history, auditability and replay, while warning that it adds substantial complexity around concurrency, schema evolution, querying and projections. It recommends selective use rather than applying the pattern universally.

PROJECT IMPLICATION:
Supports event-sourcing canonical Session history while leaving accounts, analytics, drafts, assets and UI state as ordinary data.

### Microsoft Azure Architecture Center — CQRS
https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs

FACT:
CQRS separates write and read models but becomes more complex when combined with Event Sourcing; materialized projections introduce additional design/consistency costs.

PROJECT IMPLICATION:
Use CQRS as logical separation only. Maintain gameplay-critical current state synchronously; allow recap/analytics/admin projections to lag or rebuild.

### Kurrent/EventStore documentation — expected stream revision
https://docs.kurrent.io/server/v25.1/http-api/introduction

FACT:
Appending events with an expected stream version/revision implements optimistic concurrency: the append is rejected if another writer has changed the stream.

PROJECT IMPLICATION:
Supports a Session-level expected-revision commit protocol without requiring a specialized EventStore product.

### CloudEvents specification
https://github.com/cloudevents/spec/blob/main/cloudevents/spec.md

FACT:
CloudEvents separates standard event context metadata (identity, source, type, time/schema/subject where applicable) from event data.

PROJECT IMPLICATION:
Use the same conceptual separation for the internal event envelope, but do not adopt CloudEvents as the canonical storage schema unless future interoperability requires it.

### RFC 6902 — JSON Patch
https://www.rfc-editor.org/info/rfc6902/

FACT:
JSON Patch demonstrates a small generic mutation vocabulary (add/remove/replace/move/copy/test) over JSON documents.

PROJECT IMPLICATION:
A compact mutation IR is feasible, but raw JSON Patch is too semantically weak/permissive for canonical game state. The project should use typed, schema-constrained operations and semantic Domain Events.

### RFC 8785 — JSON Canonicalization Scheme
https://www.rfc-editor.org/info/rfc8785/

FACT:
RFC 8785 defines invariant JSON serialization for repeatable cryptographic hashing by constraining representation and deterministic property ordering.

PROJECT IMPLICATION:
State, event-batch and scenario-bundle hashes must be computed from a declared canonical serialization, not incidental object serialization.

## Agent / simulation references to retain

- Google DeepMind Concordia
- Generative Agents
- AI Town

These remain architecture references rather than runtime dependencies.

## Narrative authoring references to retain

- Ink / Inky
- Data-driven scenario authoring/compiler/validation patterns

## Comparable games

Research scope includes:
- It Takes Two
- Split Fiction
- We Were Here
- The Past Within
- BOKURA
- Tick Tock: A Tale for Two
- Operation: Tango
- Keep Talking and Nobody Explodes
- CouchEscape
- The Click

Purpose: distinguish established asymmetric-information patterns from this project's focus on **asymmetric agency + causal merge + emergent consequences**.

## Research discipline

For material architecture conclusions distinguish:

- **FACT** — directly supported by authoritative material.
- **INFERENCE** — conclusion drawn from facts.
- **PROPOSAL** — suggested project design.
- **DECISION** — only after explicit user acceptance and ADR recording.


### Microsoft — Transactional Outbox pattern
https://learn.microsoft.com/en-us/azure/architecture/databases/guide/transactional-out-box-cosmos

FACT:
The Transactional Outbox pattern persists business state changes and outgoing work/events atomically, then lets a separate worker publish/process the outbox. This avoids the failure window where the business transaction commits but the downstream notification/message is lost.

PROJECT IMPLICATION:
Canonical Session commit and required post-commit work should insert outbox records in the same PostgreSQL transaction. Consumers must be idempotent. A broker is not required for the MVP baseline.
