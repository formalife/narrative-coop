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


### PostgreSQL — Row-Level Locking
https://www.postgresql.org/docs/current/explicit-locking.html

FACT:
PostgreSQL row-level locks such as FOR UPDATE serialize conflicting lockers/writers and are released at transaction end.

PROJECT IMPLICATION:
Use short row-locking transactions for Session frontier, input freeze and work claiming. Never hold database row locks across LLM/network/media calls.

### PostgreSQL — SKIP LOCKED
https://www.postgresql.org/about/featurematrix/detail/skip-locked-clause/

FACT:
PostgreSQL supports SKIP LOCKED so a worker can skip rows already locked by another claimant.

PROJECT IMPLICATION:
Useful for short transactional Outbox claim operations; task processing still uses explicit leases/fencing after the claim transaction commits.

### PostgreSQL — CREATE INDEX / Partial Indexes
https://www.postgresql.org/docs/current/sql-createindex.html

FACT:
PostgreSQL supports partial indexes and partial UNIQUE indexes over rows matching a predicate.

PROJECT IMPLICATION:
A partial unique index is a candidate for enforcing at most one nonterminal pending-input gate per Session.

### PostgreSQL — GIN / JSONB Indexing
https://www.postgresql.org/docs/current/gin.html

FACT:
PostgreSQL includes GIN operator classes for JSONB containment/path query patterns.

PROJECT IMPLICATION:
JSONB is viable for scenario-defined/current projection structures, but indexes should be added only for demonstrated query patterns rather than blanket-indexing all payloads.


### PostgreSQL — Default Function Privileges
https://www.postgresql.org/docs/current/sql-createfunction.html
https://www.postgresql.org/docs/current/sql-alterdefaultprivileges.html

FACT:
PostgreSQL grants PUBLIC EXECUTE on newly created functions/procedures by default. The documentation recommends revoking that access when inappropriate, and notes that removing the global default PUBLIC EXECUTE requires changing the role's global default privileges rather than relying on a per-schema revoke.

PROJECT IMPLICATION:
Use a dedicated migration owner, revoke default PUBLIC EXECUTE for its future functions, and explicitly revoke sensitive function EXECUTE in the same migration transaction that creates the function.

### PostgreSQL — Row Level Security
https://www.postgresql.org/docs/current/ddl-rowsecurity.html

FACT:
When Row Level Security is enabled and no applicable policy exists, PostgreSQL uses default deny for normal row access. Table owners typically bypass RLS unless configured otherwise.

PROJECT IMPLICATION:
RLS can be defense-in-depth for future player-facing relations, but internal canonical-table safety must primarily come from schema exposure boundaries, SQL privileges and non-owner runtime roles. Participant-specific policies wait for the accepted identity model.


### Supabase Edge Functions — Default environment variables
https://supabase.com/docs/guides/functions/secrets

FACT:
Hosted Supabase Edge Functions receive project defaults including SUPABASE_DB_URL and SUPABASE_SECRET_KEYS; secret keys bypass RLS.

PROJECT IMPLICATION:
Do not use hosted Edge Functions as the authoritative engine runtime when service-specific least-privilege credentials are a required boundary.

### Supabase — Database connections and custom roles
https://supabase.com/docs/guides/database/connecting-to-postgres
https://supabase.com/docs/guides/troubleshooting/fatal-password-authentication-failed

FACT:
Persistent backends can use direct or session-mode connections. Custom PostgreSQL LOGIN roles work through direct and shared Supavisor pooler connections.

PROJECT IMPLICATION:
engine-api and engine-worker can use distinct custom LOGIN roles without sharing the default postgres credential.

### Supabase — PostgreSQL SSL enforcement
https://supabase.com/docs/guides/platform/ssl-enforcement

FACT:
Supabase can enforce SSL for PostgreSQL/pooler connections and supports verify-full certificate/hostname verification.

PROJECT IMPLICATION:
Cross-provider Railway -> Supabase database traffic must use enforced SSL plus server identity verification.

### Supabase Auth — JWT/JWKS verification
https://supabase.com/docs/guides/auth/jwts
https://supabase.com/docs/reference/javascript/auth-getclaims

FACT:
Supabase Auth exposes public JWKS for asymmetric access-token signature verification.

PROJECT IMPLICATION:
The engine API can validate normal user JWTs without holding a Supabase secret/service-role key.

### Supabase Realtime — Database Broadcast
https://supabase.com/docs/guides/realtime/broadcast

FACT:
PostgreSQL can emit private Supabase Realtime Broadcast messages using realtime.send.

PROJECT IMPLICATION:
A narrow database wrapper can let engine-worker send content-free Session invalidations without a broad Supabase API secret.

### Railway — Services and service-scoped variables
https://docs.railway.com/overview/the-basics
https://docs.railway.com/variables
https://docs.railway.com/guides/cron-workers-queues

FACT:
Railway supports separate long-running services, service-scoped variables and continuous worker services.

PROJECT IMPLICATION:
Run engine-api and engine-worker as separate persistent services so DB and presentation-provider credentials remain scoped by process.


### PostgreSQL — FOR SHARE access linearization
https://www.postgresql.org/docs/17/explicit-locking.html
https://www.postgresql.org/docs/current/applevel-consistency.html

FACT:
PostgreSQL FOR SHARE row locks block concurrent UPDATE/DELETE on the same row, while allowing compatible shared locks.

PROJECT IMPLICATION:
Short FOR SHARE locks on ACTIVE Session principal bindings can linearize authorized gameplay/private-read requests against binding revocation without serializing compatible reads.

### PostgreSQL — transaction time vs wall-clock time
https://www.postgresql.org/docs/current/functions-datetime.html

FACT:
CURRENT_TIMESTAMP/now()/transaction_timestamp represent transaction-start time, while clock_timestamp returns actual current wall-clock time.

PROJECT IMPLICATION:
Invite expiry after a lock wait must use an explicitly current wall-clock evaluation point rather than transaction-start time.

### Supabase Auth — JWT subject identity
https://supabase.com/docs/guides/auth/jwt-fields

FACT:
Supabase JWT `sub` is the user ID UUID.

PROJECT IMPLICATION:
The accepted Supabase access layer may store AuthSubject as PostgreSQL UUID without putting provider identity into canonical gameplay state.

### Supabase Realtime — authorization cache
https://supabase.com/docs/guides/realtime/authorization

FACT:
Private-channel authorization is evaluated on channel connection/subscription and refreshed when a new JWT is supplied; policy changes are not re-evaluated for every message.

PROJECT IMPLICATION:
Binding revocation cannot rely on Realtime for immediate access removal. Realtime must carry content-free invalidations only; authoritative PlayerView access is rechecked by the API.
