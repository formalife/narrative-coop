# POSTMORTEM — Access Schema v0.1

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/persistence/ACCESS_SCHEMA_v0.1.md`  
**Governing architecture:** Architecture Baseline v0.4 — ACCEPTED  
**Verdict:** MODIFY

## Executive verdict

The v0.1 ownership model is correct:

- access state is noncanonical;
- AuthSubject is separate from ParticipantRef;
- initial access binding and canonical ParticipantBound are atomic;
- invite tokens are high-entropy and hash-only;
- active uniqueness is database-enforced.

The red-team found several material concurrency/integrity gaps that should be fixed before migrations.

No accepted architecture/ADR/domain/persistence decision must be superseded.

---

# 1. Binding revocation versus gameplay command

## Problem

If API authorization does:

1. SELECT ACTIVE binding;
2. later starts gameplay transaction;

a concurrent revocation can commit between those operations.

The command may then commit canonical gameplay after access was revoked.

## Correction

Participant-authorized transactions must lock/revalidate the ACTIVE binding inside the authoritative transaction.

Extend the lock hierarchy while preserving all previously accepted relative ordering:

```
session_runtime, when needed
-> access authorization rows
-> command_processing
-> window_input_gates
-> window_slot_inputs
```

For pending Submit/Replace operations that do not lock SessionRuntime:

```
active binding
-> command_processing
-> gate
-> slot
```

Revocation locks the same binding row.

Linearization:
- if command locks ACTIVE binding first, that already-authorized command may finish before revocation;
- if revocation locks first, command later observes REVOKED and rejects.

No command may rely only on a pre-transaction binding read.

---

# 2. Invite claim lock order conflicts with accepted Session ordering

## Problem

v0.1 locks invite first, then SessionRuntime.

A future transaction that locks SessionRuntime then invite can deadlock.

## Correction

Invite token lookup may first read the row without lock only to discover SessionId.

Then authoritative claim transaction locks:

```
1. session_runtime
2. invite row
3. command_processing if used
```

and revalidates token/status/expiry after both relevant locks are held.

This preserves SessionRuntime as the highest canonical lock when present.

---

# 3. Invite -> claimed binding context is underconstrained

## Problem

`claimed_binding_id` only references a BindingId.

`claim_transition_key` independently references any transition in the same Session.

A malformed row can therefore combine:
- one real binding;
- another real transition.

## Correction

Binding should expose a composite referenced key:

```
(binding_id, session_id, participation_origin_transition_key)
```

Invite CLAIMED state should use one composite FK:

```
(claimed_binding_id, session_id, claim_transition_key)
  -> binding(binding_id, session_id, participation_origin_transition_key)
```

This proves the claimed invite points to the same binding/transition context.

---

# 4. participation_origin_revision is redundant/driftable

## Problem

Binding stores both:
- transition key;
- origin revision.

The FK verifies only transition key.

Revision may drift from the referenced transition's resulting revision.

## Correction

Remove `participation_origin_revision`.

Read resulting revision from canonical_transitions when needed.

Do not duplicate immutable canonical metadata without a relational enforcement reason.

---

# 5. FK cannot prove that transition binds this ParticipantRef

## Problem

Even a composite binding -> transition FK only proves that the transition exists.

Canonical event content is stored as bytes; SQL cannot assert that the referenced ParticipantBound event actually contains the same ParticipantRef.

## Correction

Clarify authority:

- FK is historical transition identity evidence;
- application transaction constructs ParticipantRef + ParticipantBound transition together;
- post-write/replay validator must verify the referenced canonical transition establishes that ParticipantRef;
- access binding never overrides canonical ParticipationState.

Do not describe the FK as sufficient semantic proof.

---

# 6. Normal invite flow after revoked binding

## Problem

Partial uniqueness permits an AuthSubject with a REVOKED historical binding to claim a different participant slot later.

That can become role-hopping through the public guest flow.

## Correction

Normal `ClaimInvite` rejects if **any** prior binding exists for that AuthSubject in the same Session.

A future privileged AccessRecovery/Rebind workflow may explicitly create a new binding after revocation.

Add:

```
binding_kind:
  INITIAL_CLAIM
  RECOVERY_REBIND
```

MVP public flows create only INITIAL_CLAIM.

RECOVERY_REBIND remains privileged/future.

---

# 7. Invite expiry linearization

## Problem

PostgreSQL `current_timestamp` is transaction-start time.

A claim can wait on locks and then evaluate against a timestamp from before the wait.

## Correction

After acquiring SessionRuntime + invite lock, evaluate expiry using a current wall-clock function such as `clock_timestamp()` and capture one `claim_evaluated_at`.

Policy:

> An invite is claimable if it is unexpired at the authoritative locked claim evaluation point.

If accepted at that point, later crossing of expires_at during the same short transaction does not cancel the in-flight claim.

`claimed_at` uses the same captured evaluation time or later commit-side operational timestamp.

---

# 8. Expired ACTIVE invite blocks replacement

v0.1 recognized this.

Strengthen rule:

Invite issuance always serializes through SessionRuntime for the Session, even if eligibility appears static.

Under that lock:
- lock existing ACTIVE invite for slot;
- mark expired one EXPIRED;
- then apply replacement policy.

This makes concurrent invite issuance deterministic and avoids relying on unique-violation recovery as normal control flow.

---

# 9. Lost invite creation response

## Problem

Only the client response contains raw token; DB stores only its hash.

If the response is lost, the same invitation URL cannot be reconstructed.

## Correction

Accept this security/usability trade-off for MVP.

Do not store reversible invite plaintext.

Client must not blindly auto-retry invite creation.

On uncertain/lost response, explicit user "regenerate invite" action:
- revokes existing ACTIVE invite;
- creates a new token.

This may invalidate a previous URL and should be explicit UX.

---

# 10. AuthSubject UUID storage

PASS for accepted platform.

Current Supabase JWT documentation defines `sub` as user ID UUID.

Using PostgreSQL `uuid` is therefore appropriate for the accepted Supabase baseline.

Provider replacement that cannot map subjects to UUID requires access-layer migration/ADR review; canonical ParticipantRef remains unaffected.

---

# 11. Token hash/index design

PASS.

PostgreSQL partial UNIQUE indexes support active-subset uniqueness.

A 32-byte SHA-256 hash over a 256-bit random token is appropriate for lookup/uniqueness.

No time-dependent partial predicate is used.

---

# 12. Terminal Session with ACTIVE binding

PASS.

An ACTIVE access binding may remain for recap/resume/audit during retention.

Canonical application validation rejects gameplay commands for a terminal Session.

Do not make access binding status a duplicate Session lifecycle truth.

---

# 13. Root causes

## A. Authentication check was treated as request middleware rather than transaction authority

For canonical/pending writes, authorization must remain valid at the transaction's linearization point.

## B. Relational references proved existence but not shared context

Composite FKs should bind invite, access binding and transition identity as one audit chain.

## C. Operational wall time needs explicit linearization semantics

Expiry is not canonical time, but still needs a precise authoritative check point.

---

# 14. Verdict

**ACCESS_SCHEMA_v0.1 is NOT ready for acceptance.**

Create v0.2 with:

- transaction-level binding lock/revalidation;
- extended lock hierarchy;
- SessionRuntime-before-invite claim ordering;
- composite invite -> binding/transition FK;
- remove duplicated origin revision;
- semantic validator requirement for ParticipantRef origin;
- binding_kind and public role-hopping protection;
- explicit expiry evaluation point;
- deterministic invite issuance serialization;
- explicit lost-response/regenerate policy.

No executable migrations yet.
