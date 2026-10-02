# ACCESS SCHEMA v0.1

**Status:** PROPOSED  
**Date:** 2026-10-02  
**Governing architecture:** Architecture Baseline v0.4 — ACCEPTED  
**Governing identity decision:** ADR-015 — ACCEPTED  
**Governing platform:** Implementation Platform v0.3 — ACCEPTED  
**Governing PostgreSQL schema:** PostgreSQL Schema v0.5 — ACCEPTED  
**Purpose:** concrete noncanonical authorization/invite persistence before executable migrations.

---

# 1. Ownership boundary

The `access` schema stores operational authorization state only.

It answers:

- which verified AuthSubject may currently act for a ParticipantRef in one Session;
- which one-time invitation may claim one participant slot.

It does NOT own:

- canonical Participant existence;
- Role/Character assignment;
- canonical ParticipationState;
- world state;
- Session progression.

Canonical gameplay truth remains in `engine`.

---

# 2. Schema boundary

```sql
CREATE SCHEMA access;
REVOKE ALL ON SCHEMA access FROM PUBLIC;
```

The schema is not exposed through the browser Data API.

Browser roles receive no direct table access.

The authoritative API resolves access after JWT verification.

---

# 3. Identity types

For the accepted Supabase baseline:

- `session_id uuid`;
- `participant_ref uuid`;
- `auth_subject uuid`;
- `invite_id uuid`;
- `binding_id uuid`.

`auth_subject` represents the Supabase Auth JWT `sub`.

It is operational identity and is excluded from canonical state hashes/events.

No FK is created to Supabase `auth.users`.

---

# 4. session_principal_bindings

```sql
CREATE TABLE access.session_principal_bindings (
  binding_id uuid PRIMARY KEY,

  session_id uuid NOT NULL,
  participant_ref uuid NOT NULL,
  auth_subject uuid NOT NULL,

  binding_status text NOT NULL
    CHECK (
      binding_status IN (
        'ACTIVE',
        'REVOKED'
      )
    ),

  participation_origin_transition_key text NOT NULL,
  participation_origin_revision bigint NOT NULL
    CHECK (participation_origin_revision > 0),

  created_at timestamptz NOT NULL,

  revoked_at timestamptz NULL,
  revocation_reason_code text NULL,

  FOREIGN KEY (session_id)
    REFERENCES engine.sessions(session_id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  FOREIGN KEY (
    session_id,
    participation_origin_transition_key
  )
    REFERENCES engine.canonical_transitions(
      session_id,
      transition_key
    )
    DEFERRABLE INITIALLY DEFERRED
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  CHECK (
    (binding_status = 'ACTIVE'
      AND revoked_at IS NULL
      AND revocation_reason_code IS NULL)
    OR
    (binding_status = 'REVOKED'
      AND revoked_at IS NOT NULL)
  )
);
```

The transition reference is evidence that this ParticipantRef is canonically established in the Session.

For an initial claim, the binding row and referenced ParticipantBound transition commit atomically.

Future access recovery/rebind may create a new binding row pointing to the same original participation origin; it does not rewrite canonical Character history.

---

# 5. Active binding uniqueness

```sql
CREATE UNIQUE INDEX ux_access_active_auth_subject_per_session
ON access.session_principal_bindings(
  session_id,
  auth_subject
)
WHERE binding_status = 'ACTIVE';

CREATE UNIQUE INDEX ux_access_active_participant_per_session
ON access.session_principal_bindings(
  session_id,
  participant_ref
)
WHERE binding_status = 'ACTIVE';
```

Therefore under normal guest play:

- one authenticated subject controls at most one ParticipantRef in one Session;
- one ParticipantRef is controlled by at most one authenticated subject.

One AuthSubject may participate in multiple different Sessions.

---

# 6. Binding authorization lookup

After JWT verification:

```sql
SELECT participant_ref
FROM access.session_principal_bindings
WHERE session_id = $1
  AND auth_subject = $2
  AND binding_status = 'ACTIVE';
```

Zero rows:
- caller has no participant authority.

Exactly one row:
- server derives the authoritative ParticipantRef.

More than one row:
- impossible if DB invariants hold; treat as security/integrity fault.

The API never trusts a client-supplied ParticipantRef as authority.

---

# 7. Binding revocation

Revocation is operational access state.

It does not automatically create a canonical world event.

Update:

```
ACTIVE -> REVOKED
```

sets:
- `revoked_at`;
- optional `revocation_reason_code`.

A revoked binding never becomes ACTIVE again.

Rebind/recovery creates a new row under an explicit access-recovery workflow.

---

# 8. Binding update guard

A narrow trigger protects historical identity fields.

Immutable after INSERT:

- binding_id;
- session_id;
- participant_ref;
- auth_subject;
- participation_origin_transition_key;
- participation_origin_revision;
- created_at.

Allowed status transition:

```
ACTIVE -> REVOKED
```

REVOKED is terminal.

The trigger contains no gameplay logic.

---

# 9. session_invites

```sql
CREATE TABLE access.session_invites (
  invite_id uuid PRIMARY KEY,

  session_id uuid NOT NULL,
  target_participant_slot text NOT NULL,

  token_hash bytea NOT NULL
    CHECK (octet_length(token_hash) = 32),

  invite_status text NOT NULL
    CHECK (
      invite_status IN (
        'ACTIVE',
        'CLAIMED',
        'REVOKED',
        'EXPIRED'
      )
    ),

  created_by_participant_ref uuid NOT NULL,

  created_at timestamptz NOT NULL,
  expires_at timestamptz NOT NULL,

  claimed_at timestamptz NULL,
  claimed_binding_id uuid NULL,
  claim_transition_key text NULL,

  revoked_at timestamptz NULL,
  revocation_reason_code text NULL,

  FOREIGN KEY (session_id)
    REFERENCES engine.sessions(session_id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  FOREIGN KEY (claimed_binding_id)
    REFERENCES access.session_principal_bindings(binding_id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  FOREIGN KEY (
    session_id,
    claim_transition_key
  )
    REFERENCES engine.canonical_transitions(
      session_id,
      transition_key
    )
    DEFERRABLE INITIALLY DEFERRED
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  UNIQUE (token_hash),

  CHECK (expires_at > created_at),

  CHECK (
    (invite_status = 'ACTIVE'
      AND claimed_at IS NULL
      AND claimed_binding_id IS NULL
      AND claim_transition_key IS NULL
      AND revoked_at IS NULL)
    OR
    (invite_status = 'CLAIMED'
      AND claimed_at IS NOT NULL
      AND claimed_binding_id IS NOT NULL
      AND claim_transition_key IS NOT NULL
      AND revoked_at IS NULL)
    OR
    (invite_status = 'REVOKED'
      AND claimed_at IS NULL
      AND claimed_binding_id IS NULL
      AND claim_transition_key IS NULL
      AND revoked_at IS NOT NULL)
    OR
    (invite_status = 'EXPIRED'
      AND claimed_at IS NULL
      AND claimed_binding_id IS NULL
      AND claim_transition_key IS NULL)
  )
);
```

---

# 10. Invite token

Raw invite token:

- 256 cryptographically random bits;
- URL-safe encoding;
- generated server-side.

Database stores:

```
SHA-256(raw_token)
```

only.

Why fast hashing is acceptable:
- the token is random/high entropy, not a human password.

Raw token:
- is returned only for URL construction;
- is never written to database;
- is excluded from logs, analytics and error telemetry.

---

# 11. One active invite per target slot

```sql
CREATE UNIQUE INDEX ux_access_active_invite_per_slot
ON access.session_invites(
  session_id,
  target_participant_slot
)
WHERE invite_status = 'ACTIVE';
```

Important:
wall-clock expiry is not expressible safely in the index predicate.

Therefore an expired row remains technically ACTIVE until an authoritative access operation marks it EXPIRED.

Before issuing a replacement invite for a slot, the transaction:

1. locks any ACTIVE invite for that Session/slot;
2. if `expires_at <= current_timestamp`, changes it to EXPIRED;
3. otherwise explicitly revokes/reuses policy or rejects duplicate issuance;
4. inserts the new invite only after no ACTIVE row remains.

---

# 12. Invite lifecycle

Allowed:

```
ACTIVE -> CLAIMED
ACTIVE -> REVOKED
ACTIVE -> EXPIRED
```

Terminal:
- CLAIMED;
- REVOKED;
- EXPIRED.

No terminal invite reactivates.

Expiration is operational wall-clock state and does not itself mutate canonical Session state.

---

# 13. Invite update guard

Immutable after INSERT:

- invite_id;
- session_id;
- target_participant_slot;
- token_hash;
- created_by_participant_ref;
- created_at;
- expires_at.

Allowed updates are only valid lifecycle transitions and their matching terminal evidence fields.

The guard rejects:
- ACTIVE -> ACTIVE mutation of identity/token/expiry;
- terminal -> anything;
- CLAIMED without binding/transition evidence;
- REVOKED without revoked_at;
- EXPIRED with claim/revoke evidence.

---

# 14. Invite issue transaction

Preconditions:
- verified JWT;
- ACTIVE creator binding;
- canonical Session permits inviting target slot.

Transaction:

1. lock current SessionRuntime if target-slot eligibility depends on current canon;
2. validate creator ParticipantRef from binding;
3. lock existing ACTIVE invite for target slot if present;
4. mark it EXPIRED if already past wall-clock expiry;
5. apply replacement/revoke policy if still ACTIVE;
6. generate raw 256-bit token in application;
7. hash token;
8. INSERT ACTIVE invite;
9. commit;
10. return raw token once to caller.

Invite creation is noncanonical unless a future scenario explicitly makes invitation mechanics part of game canon.

---

# 15. Invite claim transaction

Input:
- verified AuthSubject;
- raw invite token.

Before transaction:
- compute SHA-256 token hash.

Transaction lock order for access claim:

```
1. matching invite row
2. engine.session_runtime
3. relevant command_processing row if EngineCommand-owned
4. access binding uniqueness rows/index effects
```

Steps:

1. lock invite by token hash;
2. require invite_status ACTIVE;
3. require `expires_at > current_timestamp`;
4. lock SessionRuntime;
5. verify target participant slot is still canonically claimable;
6. verify AuthSubject has no ACTIVE binding in Session;
7. allocate ParticipantRef;
8. prepare canonical ParticipantBound transition;
9. INSERT ACTIVE binding referencing that transition;
10. update invite to CLAIMED with binding id + transition key;
11. append canonical transition/events/update SessionRuntime;
12. create required PlayerView/Presentation/Outbox;
13. commit.

The deferred FKs are checked at commit.

Any failure rolls back both operational access and canonical participation.

---

# 16. Creator Session transaction

Preferred Session creation is one transaction:

1. verify creator JWT;
2. create Session + Genesis + Runtime revision 0;
3. allocate creator ParticipantRef;
4. prepare creator ParticipantBound transition;
5. insert ACTIVE binding referencing transition;
6. append transition/events and update Runtime;
7. optionally insert partner invite;
8. create required PlayerView/Presentation/Outbox;
9. commit.

This prevents a visible Session that exists without creator access.

---

# 17. Canonical versus operational participant rules

The access schema never decides:

- which Character the Participant controls;
- Role;
- capabilities;
- initial knowledge;
- canonical status.

Those are validated/created by the ParticipantBound canonical transition under scenario/domain rules.

The access schema only gates which AuthSubject may submit commands as the resulting ParticipantRef.

---

# 18. Revocation / Session termination

Session termination does not automatically delete bindings/invites.

Operational policy may:
- revoke ACTIVE invites;
- retain bindings for resume/history until retention cutoff.

Exact retention durations remain unfrozen.

A terminal Session should reject gameplay commands through canonical application validation even if an old binding remains ACTIVE during retention.

---

# 19. Privilege matrix

## engine_runtime

Required:
- SELECT/INSERT on access bindings/invites;
- UPDATE only lifecycle/evidence columns needed for revoke/claim/expire;
- no DELETE.

## engine_worker

No direct access-schema DML required.

## engine_ops_readonly

May receive explicit SELECT for support/audit.

## browser/authenticated roles

No direct USAGE/table privileges.

Provider-specific Realtime authorization uses a narrow SECURITY DEFINER boolean helper rather than granting browser SELECT on bindings.

---

# 20. No hard FK to auth.users

Reason:

- AuthSubject authentication is already verified by JWT before access lookup;
- avoids coupling provider auth-table lifecycle to canonical/access retention;
- out-of-band Auth deletion may leave stale evidence but cannot authenticate by itself;
- supports future Auth provider migration.

A reconciliation job may later flag ACTIVE bindings whose AuthSubject no longer exists, but it is not part of authorization correctness.

---

# 21. Audit/privacy

Persist:
- opaque AuthSubject UUID;
- Session/Participant refs;
- invite hashes;
- operational timestamps/reason codes.

Do not persist:
- invite plaintext;
- access JWTs;
- email/phone/profile data in access tables;
- raw IP by default.

Analytics should not receive token hashes or AuthSubject unless explicitly required and privacy-reviewed.

---

# 22. Access-schema invariants

1. AuthSubject is not canonical identity.
2. Exactly zero/one ACTIVE binding exists per Session/AuthSubject.
3. Exactly zero/one ACTIVE binding exists per Session/ParticipantRef.
4. A binding references a canonical participation-origin transition.
5. Initial binding and ParticipantBound commit are atomic.
6. Revoked binding never reactivates.
7. Invite plaintext is never persisted.
8. Token hash is globally unique.
9. At most one ACTIVE invite exists per Session/target slot.
10. Expired invite cannot be claimed.
11. Claiming invite atomically produces access binding + canonical ParticipantBound.
12. Claimed invite references the binding and transition that consumed it.
13. Terminal invite never reactivates.
14. Browser has no direct access-schema table privileges.
15. Stale binding without valid JWT grants no authority.
16. Access state never overrides canonical ParticipationState.

---

# 23. Red-team cases

Before acceptance test:

1. two AuthSubjects claim one invite concurrently;
2. same AuthSubject claims two participant slots concurrently;
3. two invites are issued concurrently for same slot;
4. invite expires while claim transaction begins;
5. invite expires while claim transaction is already locked/running;
6. canonical target slot becomes unavailable before claim commit;
7. binding insert succeeds but canonical transition fails;
8. canonical transition prepared but invite update fails;
9. retry after claim response is lost;
10. creator Session transaction crashes midway;
11. revoked binding attempts gameplay;
12. Auth user deleted but binding remains ACTIVE;
13. same AuthSubject participates in two different Sessions;
14. support/admin rebind creates two ACTIVE bindings;
15. expired ACTIVE invite blocks new issuance;
16. token hash collision/duplicate insert;
17. raw token appears in URL/logging telemetry;
18. malicious client supplies another ParticipantRef;
19. terminal Session still has ACTIVE binding;
20. Realtime authorization helper is abused to enumerate bindings;
21. participant origin transition does not correspond to ParticipantRef;
22. access table row is manually mutated after terminal state;
23. retention cleanup removes evidence still needed for active/recoverable Session;
24. future Auth provider subject is not UUID-shaped.

---

# 24. Acceptance gate

Run access-schema red-team/postmortem.

If material flaws are found:
- produce ACCESS_SCHEMA_v0.2 before migrations.

If accepted:
- add provider-specific Realtime access helper design;
- perform Supabase/Railway integration spikes;
- then generate executable migrations.
