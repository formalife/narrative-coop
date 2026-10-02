# ACCESS SCHEMA v0.2

**Status:** PROPOSED  
**Date:** 2026-10-02  
**Supersedes as working proposal:** ACCESS_SCHEMA_v0.1  
**Governing architecture:** Architecture Baseline v0.4 — ACCEPTED  
**Governing ADR:** ADR-015 — ACCEPTED  
**Governing platform:** Implementation Platform v0.3 — ACCEPTED  
**Governing PostgreSQL schema:** PostgreSQL Schema v0.5 — ACCEPTED

---

# 1. Boundary

`access` is operational authorization state.

It answers:
- may this verified AuthSubject currently act/read as this ParticipantRef?
- may this one-time invite claim a participant slot?

It never defines:
- Participant existence;
- Character/Role;
- gameplay capability;
- knowledge;
- Session lifecycle.

Those remain canonical engine state.

---

# 2. Schema

```sql
CREATE SCHEMA access;
REVOKE ALL ON SCHEMA access FROM PUBLIC;
```

Not exposed through browser Data API.

No browser role receives direct table privileges.

---

# 3. Identity representation

Under accepted Supabase Auth baseline:

```
AuthSubject -> uuid
ParticipantRef -> uuid
SessionId -> uuid
```

Current Supabase JWT contract defines `sub` as the user-id UUID.

No FK to `auth.users`.

Provider migration that cannot retain UUID-shaped subjects requires access-layer migration, not canonical gameplay migration.

---

# 4. session_principal_bindings

```sql
CREATE TABLE access.session_principal_bindings (
  binding_id uuid PRIMARY KEY,

  session_id uuid NOT NULL,
  participant_ref uuid NOT NULL,
  auth_subject uuid NOT NULL,

  binding_kind text NOT NULL
    CHECK (
      binding_kind IN (
        'INITIAL_CLAIM',
        'RECOVERY_REBIND'
      )
    ),

  binding_status text NOT NULL
    CHECK (
      binding_status IN (
        'ACTIVE',
        'REVOKED'
      )
    ),

  participation_origin_transition_key text NOT NULL,

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

  UNIQUE (
    binding_id,
    session_id,
    participation_origin_transition_key
  ),

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

`participation_origin_transition_key` is historical evidence pointer.

The FK proves the canonical transition exists in the Session.

It does **not** alone prove semantic ParticipantRef equality inside canonical event bytes.

The application/replay access validator verifies that relationship.

---

# 5. Active uniqueness

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

One AuthSubject may still participate in multiple different Sessions.

---

# 6. Binding insert/update lifecycle

Normal INSERT:
- status ACTIVE;
- terminal revocation fields NULL.

Public guest flows may insert only:

`binding_kind = INITIAL_CLAIM`.

`RECOVERY_REBIND` is reserved for future privileged access-recovery workflow.

Allowed update:

```
ACTIVE -> REVOKED
```

REVOKED is terminal.

Immutable after INSERT:

- binding_id;
- session_id;
- participant_ref;
- auth_subject;
- binding_kind;
- participation_origin_transition_key;
- created_at.

Use a narrow lifecycle trigger to enforce these rules.

---

# 7. Normal role-hopping protection

Normal public `ClaimInvite` rejects if **any binding history** already exists for:

```
(session_id, auth_subject)
```

not merely an ACTIVE binding.

Therefore a subject whose previous binding was revoked cannot claim another role through the ordinary guest invite endpoint.

Future recovery/rebind is a separate privileged command/workflow.

---

# 8. Binding authorization transaction rule

A pre-request JWT check is necessary but not sufficient for private/canonical access.

Any participant-authorized database transaction must re-read the ACTIVE binding and acquire a row lock that conflicts with revocation.

For mutating gameplay/pending-input requests:
- lock ACTIVE binding for the duration of the authoritative transaction.

For private PlayerInteractionView reads:
- hold a short shared row lock while the authorized view is selected/derived.

Revocation updates/locks the same binding row.

Linearization rule:

> a request that acquires the valid binding lock before revocation may finish; a request that acquires it after revocation must reject.

No command/view authorization depends only on middleware state captured before the transaction.

---

# 9. Extended lock hierarchy

Preserve all accepted relative engine lock ordering.

When relevant:

```
1. engine.session_runtime
2. access authorization row(s)
3. engine.command_processing
4. engine.window_input_gates
5. engine.window_slot_inputs ordered by action_slot_id
```

Transactions using only a subset preserve the same relative order.

Examples:

### Submit/Replace pending action

```
ACTIVE binding
-> command_processing
-> gate
-> slot
```

### Canonical participant command

```
session_runtime
-> ACTIVE binding
-> command_processing
-> relevant gate/slot
```

### Binding revocation

```
binding row only
```

If a future revocation operation also needs SessionRuntime, it must lock SessionRuntime first.

---

# 10. session_invites

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

  expired_at timestamptz NULL,

  FOREIGN KEY (session_id)
    REFERENCES engine.sessions(session_id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  FOREIGN KEY (
    claimed_binding_id,
    session_id,
    claim_transition_key
  )
    REFERENCES access.session_principal_bindings(
      binding_id,
      session_id,
      participation_origin_transition_key
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
      AND revoked_at IS NULL
      AND expired_at IS NULL)
    OR
    (invite_status = 'CLAIMED'
      AND claimed_at IS NOT NULL
      AND claimed_binding_id IS NOT NULL
      AND claim_transition_key IS NOT NULL
      AND revoked_at IS NULL
      AND expired_at IS NULL)
    OR
    (invite_status = 'REVOKED'
      AND claimed_at IS NULL
      AND claimed_binding_id IS NULL
      AND claim_transition_key IS NULL
      AND revoked_at IS NOT NULL
      AND expired_at IS NULL)
    OR
    (invite_status = 'EXPIRED'
      AND claimed_at IS NULL
      AND claimed_binding_id IS NULL
      AND claim_transition_key IS NULL
      AND revoked_at IS NULL
      AND expired_at IS NOT NULL)
  )
);
```

The composite FK binds CLAIMED invite evidence to one exact binding/participation-origin transition context.

---

# 11. Invite active uniqueness

```sql
CREATE UNIQUE INDEX ux_access_active_invite_per_slot
ON access.session_invites(
  session_id,
  target_participant_slot
)
WHERE invite_status = 'ACTIVE';
```

No wall-clock function appears in the partial index.

Expired-but-not-yet-marked rows remain ACTIVE until an authoritative access transaction terminalizes them.

---

# 12. Invite insert/update lifecycle

Normal INSERT:
- ACTIVE only;
- no claim/revoke/expiry evidence fields set.

Allowed:

```
ACTIVE -> CLAIMED
ACTIVE -> REVOKED
ACTIVE -> EXPIRED
```

All terminal states remain terminal.

Immutable after INSERT:

- invite_id;
- session_id;
- target_participant_slot;
- token_hash;
- created_by_participant_ref;
- created_at;
- expires_at.

A narrow lifecycle trigger enforces INSERT/update rules.

---

# 13. Token contract

Application generates:
- 32 cryptographically random bytes;
- URL-safe encoding for transport.

Database stores only:

`SHA-256(raw_token)`

with exactly 32 bytes.

No plaintext token:
- database;
- logs;
- analytics;
- exception telemetry.

The token is an unguessable capability for selecting an invite, but claim still requires a valid authenticated AuthSubject.

---

# 14. Invite lookup and claim lock order

Before authoritative transaction:
- hash raw token;
- perform a non-locking lookup only to discover SessionId.

That lookup grants no authority.

Authoritative claim transaction then:

```
1. lock engine.session_runtime for discovered Session
2. lock invite row by exact token_hash
3. revalidate SessionId/status/expiry
4. lock/create command_processing if ClaimInvite is journaled
5. insert binding / canonical transition
```

Never lock invite first and SessionRuntime second in a transaction that needs both.

---

# 15. Expiry linearization

PostgreSQL transaction-start `current_timestamp` is not the intended authority after lock waits.

After SessionRuntime + invite row are locked:

1. capture one database wall-clock evaluation value, e.g. `clock_timestamp()`;
2. require:
   `claim_evaluated_at < expires_at`.

Policy:

> expiry is evaluated once at the locked authoritative claim point.

If valid at that point, crossing expires_at later during the same short transaction does not cancel the already-linearized claim.

`claimed_at` records that evaluation time or a later commit-side operational timestamp.

Wall-clock expiry remains noncanonical.

---

# 16. Invite issuance serialization

Invite issuance always locks SessionRuntime first.

Then:
1. validate creator ACTIVE binding within transaction;
2. validate target slot from canonical state;
3. lock existing ACTIVE invite for target slot if any;
4. if expired at evaluation time, mark EXPIRED;
5. otherwise apply explicit regenerate/revoke policy;
6. insert new ACTIVE invite;
7. commit;
8. return plaintext token exactly once.

This serializes concurrent issue/regenerate operations within the Session.

---

# 17. Lost create-invite response

Plaintext invite token is intentionally unrecoverable from DB.

Therefore:
- client does not blindly network-retry create-invite;
- uncertain/lost response requires explicit **Regenerate Invite**;
- regeneration terminalizes the existing ACTIVE invite then creates a new token.

This security/usability trade-off is accepted for MVP.

No reversible token storage is introduced.

---

# 18. Invite claim transaction

After JWT verification and token hash/session discovery:

1. lock SessionRuntime;
2. lock invite row;
3. require ACTIVE;
4. evaluate expiry under lock;
5. verify target slot still canonically claimable;
6. reject if any prior binding history exists for AuthSubject in Session;
7. allocate ParticipantRef;
8. prepare ParticipantBound transition;
9. insert INITIAL_CLAIM binding referencing transition key;
10. set invite CLAIMED with binding id + same transition key;
11. append canonical transition/events + update SessionRuntime;
12. materialize required PlayerView/Presentation/Outbox;
13. commit.

Deferred FKs validate at commit.

Any failure rolls back access + canon together.

---

# 19. Creator Session transaction

One transaction:

1. verify creator AuthSubject;
2. create Session/Genesis/Runtime revision 0;
3. allocate ParticipantRef;
4. prepare ParticipantBound transition;
5. insert INITIAL_CLAIM binding;
6. append transition/events/update Runtime;
7. optionally issue invite;
8. create PlayerView/Presentation/Outbox;
9. commit.

No visible orphan Session with no creator access.

---

# 20. Participation semantic verifier

Because canonical event content is immutable serialized bytes, SQL FK cannot prove:

`binding.participant_ref == ParticipantRef established by participation_origin_transition`.

Required validator:

```
verifyAccessBindingOrigin(binding, canonical transition/events)
```

It checks:
- transition belongs to Session;
- transition contains/causes the expected ParticipantBound semantic occurrence;
- resulting canonical ParticipationState contains the same ParticipantRef.

Run:
- in initial transaction plan validation;
- in DB integration tests;
- in forensic/integrity scans.

Binding evidence can never override canonical ParticipationState.

---

# 21. Revocation semantics

Revocation:
- locks binding row for update;
- ACTIVE -> REVOKED;
- records revoked_at/reason;
- never rewrites canonical participation.

Gameplay/private access transaction holding the binding authorization lock linearizes before revocation.

Requests arriving after revocation reject.

A terminal Session may retain ACTIVE binding for recap/resume/audit policy; canonical lifecycle still controls command legality.

---

# 22. Privileges

## engine_runtime

- USAGE access schema;
- SELECT bindings/invites;
- INSERT bindings/invites;
- column-limited UPDATE for lifecycle fields only;
- no DELETE.

## engine_worker

No access table DML.

## engine_ops_readonly

Optional explicit SELECT.

## browser/authenticated

No schema/table privilege.

Provider-specific Realtime authorization uses a narrow boolean SECURITY DEFINER helper.

---

# 23. Lifecycle triggers

Use two narrow triggers:

`access.guard_session_principal_binding()`

Enforces:
- INSERT ACTIVE only;
- immutable identity/origin fields;
- ACTIVE -> REVOKED only;
- REVOKED terminal.

`access.guard_session_invite()`

Enforces:
- INSERT ACTIVE only;
- immutable token/slot/time/creator fields;
- ACTIVE -> CLAIMED/REVOKED/EXPIRED only;
- terminal state immutable.

Functions:
- empty search_path;
- PUBLIC EXECUTE explicitly revoked.

They protect audit/access integrity only and contain no gameplay rules.

---

# 24. Privacy / retention

Persist only:
- opaque AuthSubject UUID;
- Participant/Session refs;
- invite hashes;
- operational timestamps/reason codes.

Do not persist:
- JWTs;
- invite plaintext;
- Auth email/phone/profile;
- raw IP for domain purposes.

Retention duration remains a product/privacy decision.

Cleanup cannot remove access evidence required by a nonterminal/resumable Session.

---

# 25. Invariants

1. AuthSubject is operational, not canonical.
2. One ACTIVE binding per Session/AuthSubject.
3. One ACTIVE binding per Session/ParticipantRef.
4. Public claim rejects any previous binding history for same AuthSubject/Session.
5. Initial binding and ParticipantBound canonical transition are atomic.
6. Binding authorization is revalidated/locked inside private/write DB transactions.
7. Revoked binding never reactivates.
8. Invite plaintext never persists.
9. Token hash globally unique.
10. One ACTIVE invite per Session/target slot.
11. Expiry evaluated at a defined locked wall-clock point.
12. Claim atomically consumes invite + creates binding + commits ParticipantBound.
13. Claimed invite points to same binding/participation-origin transition.
14. Terminal invite never reactivates.
15. Browser cannot read access tables directly.
16. Valid binding without valid JWT grants nothing.
17. SQL FK proves transition identity, semantic validator proves ParticipantRef meaning.
18. Access state cannot override canonical ParticipationState.

---

# 26. Red-team gate

Test:

1. revoke vs pending-action submission;
2. revoke vs canonical command;
3. revoke vs PlayerView read;
4. two users claim one invite;
5. same subject claims two invites concurrently;
6. same subject previously revoked then uses normal invite;
7. two invite issuers race;
8. expired ACTIVE invite blocks replacement;
9. claim waits across expires_at;
10. claim reads SessionId then invite is revoked before lock;
11. target slot changes before claim lock;
12. binding/transition mismatch corruption;
13. claimed invite references wrong binding/transition;
14. trigger mutation attempts;
15. role grants allow immutable-column update;
16. raw token logging;
17. lost invite response/regenerate;
18. Auth user deletion with active binding;
19. terminal Session + ACTIVE binding;
20. future RECOVERY_REBIND path accidentally exposed publicly;
21. provider-specific Realtime helper enumerates access data;
22. retention cleanup removes needed binding evidence.

No executable migrations before this gate passes.
