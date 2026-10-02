# ACCESS SCHEMA v0.3

**Status:** ACCEPTED  
**Date:** 2026-10-02  
**Supersedes:** ACCESS_SCHEMA_v0.2  
**Governing architecture:** Architecture Baseline v0.4 — ACCEPTED  
**Governing ADR:** ADR-015 — ACCEPTED  
**Governing platform:** Implementation Platform v0.3 — ACCEPTED  
**Governing PostgreSQL schema:** PostgreSQL Schema v0.5 — ACCEPTED

---

# 1. MVP scope

This schema supports only initial guest/account binding for a Session.

It intentionally does **not** support access rebind/recovery after revocation.

That feature requires a later reviewed migration/ADR because it relaxes lifetime uniqueness.

---

# 2. Boundary

`access` stores noncanonical authorization/invite state.

It never owns canonical Participant/Role/Character/Session lifecycle.

---

# 3. session_principal_bindings

```sql
CREATE TABLE access.session_principal_bindings (
  binding_id uuid PRIMARY KEY,

  session_id uuid NOT NULL,
  participant_ref uuid NOT NULL,
  auth_subject uuid NOT NULL,

  binding_status text NOT NULL
    CHECK (
      binding_status IN ('ACTIVE', 'REVOKED')
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

  UNIQUE (binding_id, session_id),

  UNIQUE (session_id, auth_subject),
  UNIQUE (session_id, participant_ref),

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

The lifetime UNIQUE constraints mean:
- one AuthSubject can never switch ParticipantRef within the same Session under MVP schema;
- one ParticipantRef can never be rebound to a new AuthSubject under MVP schema.

Future recovery explicitly changes this design.

---

# 4. Binding lifecycle

INSERT:
- ACTIVE only;
- revocation fields NULL.

UPDATE:
- ACTIVE -> REVOKED only.

Immutable after INSERT:
- binding_id;
- session_id;
- participant_ref;
- auth_subject;
- participation_origin_transition_key;
- created_at.

REVOKED is terminal.

A narrow trigger enforces this.

---

# 5. Canonical-origin semantics

The FK proves the referenced canonical transition exists in the Session.

It does not prove the serialized event semantics establish the same ParticipantRef.

Required validator:

`verifyAccessBindingOrigin(binding, canonical transition/events/resulting state)`

Checks:
- Session identity;
- ParticipantBound semantic occurrence;
- resulting ParticipationState includes same ParticipantRef.

The binding never overrides canonical participation.

---

# 6. Authorization lock semantics

JWT verification occurs first.

For every participant-private/write transaction, authorization is revalidated in PostgreSQL.

Authorized request acquires:

```sql
SELECT ...
FROM access.session_principal_bindings
WHERE session_id = $1
  AND auth_subject = $2
  AND binding_status = 'ACTIVE'
FOR SHARE;
```

Revocation updates/locks the same row and therefore conflicts with FOR SHARE.

Do **not** use FOR KEY SHARE for this purpose.

Linearization:

- request obtains valid FOR SHARE first -> that request may finish;
- revocation obtains conflicting lock first -> later request sees REVOKED and rejects.

Private PlayerView reads use a short transaction with the same authorization lock.

---

# 7. Extended lock ordering

When locks are combined:

```
1. engine.session_runtime, if needed
2. access.session_principal_bindings
3. access.session_invites
4. engine.command_processing
5. engine.window_input_gates
6. engine.window_slot_inputs ordered by action_slot_id
```

A transaction may use a subset but never reverses relative order.

Examples:

### Pending Submit/Replace

```
binding FOR SHARE
-> command_processing
-> gate
-> slot
```

### Canonical participant command

```
session_runtime
-> binding FOR SHARE
-> command_processing
-> gate/slot if needed
```

### Invite issuance

```
session_runtime
-> creator binding FOR SHARE
-> active invite row
```

### Invite claim

No existing claimant binding row exists.

```
session_runtime
-> invite row
-> command_processing if used
-> INSERT new binding
```

### Revocation

Binding row only.

If a future revocation transaction also needs SessionRuntime, it locks SessionRuntime first.

---

# 8. session_invites

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

  created_by_binding_id uuid NOT NULL,

  created_at timestamptz NOT NULL,
  expires_at timestamptz NOT NULL,

  claimed_at timestamptz NULL,
  claimed_binding_id uuid NULL,

  revoked_at timestamptz NULL,
  revocation_reason_code text NULL,

  expired_at timestamptz NULL,

  FOREIGN KEY (session_id)
    REFERENCES engine.sessions(session_id)
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  FOREIGN KEY (
    created_by_binding_id,
    session_id
  )
    REFERENCES access.session_principal_bindings(
      binding_id,
      session_id
    )
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  FOREIGN KEY (
    claimed_binding_id,
    session_id
  )
    REFERENCES access.session_principal_bindings(
      binding_id,
      session_id
    )
    DEFERRABLE INITIALLY DEFERRED
    ON UPDATE RESTRICT
    ON DELETE RESTRICT,

  UNIQUE (token_hash),

  CHECK (expires_at > created_at),

  CHECK (
    claimed_binding_id IS NULL
    OR claimed_binding_id <> created_by_binding_id
  ),

  CHECK (
    (invite_status = 'ACTIVE'
      AND claimed_at IS NULL
      AND claimed_binding_id IS NULL
      AND revoked_at IS NULL
      AND expired_at IS NULL)
    OR
    (invite_status = 'CLAIMED'
      AND claimed_at IS NOT NULL
      AND claimed_binding_id IS NOT NULL
      AND revoked_at IS NULL
      AND expired_at IS NULL)
    OR
    (invite_status = 'REVOKED'
      AND claimed_at IS NULL
      AND claimed_binding_id IS NULL
      AND revoked_at IS NOT NULL
      AND expired_at IS NULL)
    OR
    (invite_status = 'EXPIRED'
      AND claimed_at IS NULL
      AND claimed_binding_id IS NULL
      AND revoked_at IS NULL
      AND expired_at IS NOT NULL)
  )
);
```

---

# 9. Active invite uniqueness

```sql
CREATE UNIQUE INDEX ux_access_active_invite_per_slot
ON access.session_invites(
  session_id,
  target_participant_slot
)
WHERE invite_status = 'ACTIVE';
```

No time-dependent predicate.

---

# 10. Invite lifecycle

INSERT:
- ACTIVE only.

Allowed update:

```
ACTIVE -> CLAIMED
ACTIVE -> REVOKED
ACTIVE -> EXPIRED
```

Terminal thereafter.

Immutable:
- invite_id;
- session_id;
- target_participant_slot;
- token_hash;
- created_by_binding_id;
- created_at;
- expires_at.

A narrow trigger enforces this.

---

# 11. Token contract

Generate server-side:
- 256 random bits;
- URL-safe representation.

Persist:
- SHA-256(raw token) only.

Never persist/log:
- raw token;
- JWT;
- email/phone/profile;
- raw IP for gameplay domain purposes.

---

# 12. Invite issue transaction

1. verified JWT;
2. lock SessionRuntime;
3. lock creator ACTIVE binding FOR SHARE;
4. verify creator canonical permissions/target slot;
5. lock existing ACTIVE invite for target slot if present;
6. evaluate its expiry with database wall clock;
7. mark EXPIRED if already expired;
8. otherwise explicit regenerate/revoke policy;
9. generate/hash/insert new ACTIVE invite;
10. commit;
11. return plaintext token once.

Lost response:
- no automatic blind retry;
- explicit Regenerate Invite terminalizes old ACTIVE invite and creates a new token.

---

# 13. Invite claim transaction

Before transaction:
- hash token;
- non-locking lookup may discover candidate SessionId only.

Inside transaction:

1. lock SessionRuntime;
2. lock invite by exact token hash;
3. revalidate same SessionId and ACTIVE status;
4. capture authoritative expiry-evaluation wall time;
5. require evaluation time < expires_at;
6. verify canonical target slot still claimable;
7. verify no existing binding for `(session_id, auth_subject)`;
8. allocate ParticipantRef;
9. prepare canonical ParticipantBound transition;
10. INSERT ACTIVE binding;
11. UPDATE invite -> CLAIMED with claimed_binding_id;
12. append transition/events and update SessionRuntime;
13. materialize required PlayerView/Presentation/Outbox;
14. commit.

Lifetime UNIQUE constraints independently protect AuthSubject/ParticipantRef even if application checks race or fail.

---

# 14. Invite semantic validator

Required validator:

`verifyClaimedInviteOrigin(invite, claimed binding, canonical transition/resulting state)`

Checks:
- invite and binding Session match;
- binding canonical origin establishes its ParticipantRef;
- target_participant_slot corresponds to the participant slot consumed by that canonical ParticipantBound transition;
- resulting ParticipationState is consistent.

SQL relational evidence cannot replace this semantic validation.

---

# 15. Expiry semantics

Do not use transaction-start `current_timestamp` as the authoritative claim time after lock waits.

After relevant locks:
- capture one current database wall-clock value;
- compare against `expires_at`.

If unexpired at that linearization point, later crossing expiry during the same short transaction does not invalidate the in-flight claim.

Expiration is operational, not canonical gameplay time.

---

# 16. Revocation

Revocation:
- locks binding row;
- ACTIVE -> REVOKED;
- sets revoked_at/reason;
- never rewrites canonical participation.

Because private/write requests hold FOR SHARE on the binding, revocation and authorized requests have deterministic relative ordering.

MVP does not support rebind after revocation.

---

# 17. Session termination

Binding may remain ACTIVE after canonical Session termination for recap/audit/resume-retention policy.

Canonical Session lifecycle rejects illegal gameplay commands.

Do not duplicate Session terminal state into access status.

---

# 18. Realtime caveat

Supabase Realtime authorization is evaluated/cached when a channel is joined and when a new JWT is supplied.

Therefore binding revocation may not immediately evict an already connected channel.

This is acceptable only because Realtime payloads are content-free invalidations.

Rules:
- Realtime is never private-state authority;
- API refetch always revalidates current binding;
- revoked client may temporarily receive a meaningless invalidation for a Session it already knew;
- no private PlayerView/action/secret/narrative payload is broadcast.

---

# 19. Privileges

## engine_runtime

- USAGE on access schema;
- SELECT/INSERT tables;
- column-limited UPDATE on lifecycle/evidence fields;
- no DELETE.

## engine_worker

No access-table DML.

## engine_ops_readonly

Optional explicit SELECT.

## browser roles

No schema/table privilege.

Lifecycle trigger functions are not public APIs.

---

# 20. Trigger requirements

`guard_session_principal_binding`
- INSERT ACTIVE only;
- immutable identity/origin;
- ACTIVE -> REVOKED only;
- REVOKED terminal.

`guard_session_invite`
- INSERT ACTIVE only;
- immutable token/slot/creator/times;
- ACTIVE -> one terminal state only;
- terminal immutable.

Trigger functions:
- narrow;
- no gameplay logic;
- PUBLIC EXECUTE revoked;
- provider-neutral PostgreSQL.

---

# 21. Current-source facts

Current PostgreSQL documentation confirms:
- partial UNIQUE indexes enforce uniqueness over a qualifying subset;
- FOR SHARE blocks concurrent UPDATE/DELETE on the selected row;
- current/transaction timestamps represent transaction start, while clock-time functions can obtain current wall time.

Current Supabase documentation confirms:
- JWT `sub` is the user-id UUID;
- anonymous users have a user ID and authenticated access token;
- Realtime authorization policy is cached for a connection and is refreshed on new JWT/subscription.

These facts are implementation assumptions and should be reverified during deployment spikes.

---

# 22. MVP invariants

1. One lifetime binding per Session/AuthSubject.
2. One lifetime binding per Session/ParticipantRef.
3. No MVP access rebind.
4. Binding origin references canonical transition and passes semantic validator.
5. Private/write requests revalidate and lock current ACTIVE binding.
6. Revocation linearizes against requests through row locks.
7. One ACTIVE invite per Session/target slot.
8. Token plaintext never persists.
9. Claim expiry has explicit locked evaluation time.
10. Claim atomically creates binding + ParticipantBound + consumes invite.
11. Invite creator/claim binding belong to same Session by FK.
12. Claim cannot point to creator binding.
13. Invite semantic target is verified against canonical participation.
14. Terminal invite/binding states never reactivate.
15. Access cannot override canonical ParticipationState.
16. Realtime is never relied upon for revocation/private data security.

---

# 23. Acceptance red-team cases

1. revocation vs pending SubmitAction;
2. revocation vs canonical command;
3. revocation vs PlayerView read;
4. two users claim one token;
5. one user claims two tokens same Session;
6. one participant gets two AuthSubjects;
7. creator revoked during invite issue;
8. invite expires while waiting for SessionRuntime;
9. invite revoked after prelookup before authoritative lock;
10. target slot consumed by another canonical transition;
11. wrong claimed binding attached manually;
12. wrong canonical origin transition;
13. wrong target slot versus ParticipantBound semantics;
14. two invite issuers race;
15. expired ACTIVE invite replacement;
16. raw token leakage;
17. lost invite response;
18. trigger bypass attempt;
19. immutable-column UPDATE grant mistake;
20. Auth deletion with still-valid JWT but revoked binding;
21. Realtime subscriber remains connected after access revocation;
22. terminal Session retains ACTIVE binding;
23. future recovery requirement;
24. provider migration away from UUID AuthSubject.

No executable migrations before this gate passes.


---

# Acceptance record

Access Schema v0.3 was explicitly accepted on 2026-10-02.

Acceptance evidence:
- `RED_TEAM_ACCESS_SCHEMA_v0.3.md`
- `POSTMORTEM_ACCESS_SCHEMA_v0.1.md`
- `POSTMORTEM_ACCESS_SCHEMA_v0.2.md`

This schema is now the binding access-security persistence baseline.

Executable migrations remain gated on the target Supabase/Railway integration spike.
