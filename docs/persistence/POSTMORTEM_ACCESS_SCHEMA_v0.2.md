# POSTMORTEM — Access Schema v0.2

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/persistence/ACCESS_SCHEMA_v0.2.md`  
**Verdict:** MODIFY

## Executive verdict

v0.2 fixes the serious revocation race, lock ordering, expiry and invite/binding context problems from v0.1.

The remaining issues are simplification/integrity issues rather than architectural defects.

The largest finding is that v0.2 models `RECOVERY_REBIND` before the product supports recovery. That creates schema states the normal MVP must forbid through application convention.

The safer MVP schema should enforce the stronger fact:

> one AuthSubject and one ParticipantRef each have at most one binding history in a Session, even after revocation.

Recovery can later deliberately relax that rule through a reviewed migration.

---

# 1. RECOVERY_REBIND is premature

## Problem

v0.2 includes:

`binding_kind = INITIAL_CLAIM | RECOVERY_REBIND`

but recovery/rebind UX and authority are explicitly unfrozen.

The database therefore supports a privileged state that no accepted workflow owns.

## Correction

Remove `binding_kind` from MVP schema.

Replace partial ACTIVE uniqueness with lifetime uniqueness:

```
UNIQUE (session_id, auth_subject)
UNIQUE (session_id, participant_ref)
```

Consequences:
- revocation is terminal for that access identity in the Session;
- ordinary role hopping is impossible even if application checks fail;
- future recovery requires an explicit schema/ADR migration.

This matches the accepted MVP limitation around lost anonymous access.

---

# 2. Invite duplicates canonical transition identity unnecessarily

## Problem

CLAIMED invite stores:
- claimed_binding_id;
- claim_transition_key.

But the claimed binding already owns:
- participation_origin_transition_key.

The extra invite transition field is redundant even though v0.2 constrains it with a composite FK.

## Correction

Remove `claim_transition_key` from invite.

Audit chain becomes:

```
Invite
  -> claimed_binding_id
  -> participation_origin_transition_key
  -> canonical transition
```

Fewer duplicated immutable facts.

---

# 3. Creator audit should reference the access binding

## Problem

Invite stores only `created_by_participant_ref`.

That proves a claimed participant identity value, but not which operational authorization binding created the invite.

## Correction

Store:

`created_by_binding_id`

instead.

Add composite FK:

```
(created_by_binding_id, session_id)
  -> session_principal_bindings(binding_id, session_id)
```

ParticipantRef is derivable from binding.

Issue transaction still requires creator binding ACTIVE under lock.

This produces a stronger noncanonical audit chain without adding PII.

---

# 4. Claimed binding/session FK can be simpler

After removing invite transition duplication:

```
(claimed_binding_id, session_id)
  -> binding(binding_id, session_id)
```

is sufficient relational identity.

Add:

```
CHECK (
  claimed_binding_id IS NULL
  OR claimed_binding_id <> created_by_binding_id
)
```

to reject self-consumption at row level.

Semantic target-slot equality still requires the access/canonical validator.

---

# 5. Invite target-slot semantics remain non-relational

PASS with explicit validator requirement.

SQL cannot prove that:
- target_participant_slot;
- ParticipantBound transition;
- resulting ParticipantRef/Role/Character

represent the same canonical slot because canonical state/events are structured bytes/JSON.

Required validator:

`verifyClaimedInviteSemanticOrigin(invite, binding, transition, resulting state)`

This is analogous to binding-origin verification.

---

# 6. Explicit row-lock mode

v0.2 says private/write requests use a shared lock conflicting with revocation.

Freeze it more precisely:

- authorized request: `SELECT ... FOR SHARE` on ACTIVE binding;
- revocation: `SELECT ... FOR UPDATE` / UPDATE same row.

Current PostgreSQL documentation states FOR SHARE blocks concurrent UPDATE/DELETE on the row.

Do not use FOR KEY SHARE: it is too weak because ordinary status UPDATE need not change a key.

This remains a short-lived access lock, not gameplay serialization.

---

# 7. Realtime revocation is not immediate

Current Supabase Realtime authorization caches RLS access for the duration of the connection and refreshes it on subscription/new JWT.

Therefore revoking the access binding does not necessarily eject an already connected Realtime subscriber immediately.

This does **not** invalidate the architecture because Realtime payload is intentionally content-free invalidation only.

Rules:
- API/private PlayerView access uses current binding and revokes immediately under the DB lock policy;
- Realtime payload contains no private/canonical content;
- JWT lifetimes should remain bounded;
- provider-specific Realtime design must document this residual behavior.

Do not make Realtime subscription presence an access-authority signal.

---

# 8. Invite issuance and creator binding order

Strengthen explicit order:

```
session_runtime
-> creator binding FOR SHARE
-> active invite row
```

This prevents invite issuance from succeeding after creator revocation and remains compatible with the extended lock hierarchy.

---

# 9. v0.2 features that pass

Preserve:

- access/canon ownership separation;
- AuthSubject UUID under Supabase baseline;
- no auth.users FK;
- binding lock revalidation inside authorized transaction;
- SessionRuntime-before-access ordering when both are used;
- 256-bit raw invite token + SHA-256 hash only;
- active invite partial UNIQUE index;
- clock_timestamp-style expiry evaluation after lock acquisition;
- explicit lost-response/regenerate policy;
- lifecycle triggers;
- no DELETE runtime privilege.

---

# 10. Verdict

**ACCESS_SCHEMA_v0.2 is NOT ready for acceptance.**

Create v0.3 with:

- no RECOVERY_REBIND state;
- lifetime Session/AuthSubject and Session/ParticipantRef uniqueness;
- created_by_binding_id;
- claimed_binding_id only, no duplicate claim transition key;
- semantic invite-origin validator;
- explicit FOR SHARE vs FOR UPDATE access-lock semantics;
- Realtime revocation-cache caveat documented.

No executable migrations yet.
