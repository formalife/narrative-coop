# RED-TEAM — Access Schema v0.3

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/persistence/ACCESS_SCHEMA_v0.3.md`  
**Governing architecture:** Architecture Baseline v0.4 — ACCEPTED  
**Verdict:** READY FOR ACCEPTANCE

## 1. Ownership boundary

PASS.

The schema stores operational authorization/invite state only.

It does not duplicate:
- Character;
- Role;
- capabilities;
- canonical Session lifecycle;
- canonical ParticipationState.

AuthSubject never becomes gameplay identity.

---

# 2. Lifetime binding uniqueness

PASS for MVP.

```
UNIQUE(session_id, auth_subject)
UNIQUE(session_id, participant_ref)
```

is stronger and safer than partial ACTIVE uniqueness while recovery/rebind is not implemented.

Consequences are explicit:
- revoked access cannot be rebound under current schema;
- role hopping cannot be enabled accidentally through normal API;
- future recovery requires deliberate migration/review.

This matches the accepted MVP anonymous-access limitation.

---

# 3. Revocation versus write commands

PASS.

Participant-authorized write transactions acquire ACTIVE binding FOR SHARE.

PostgreSQL FOR SHARE conflicts with concurrent UPDATE/DELETE on the row.

Revocation therefore has a clear linearization point.

The extended order preserves all previously accepted engine relative lock ordering.

---

# 4. Revocation versus private reads

PASS.

A private PlayerInteractionView request holds a short authorization transaction/lock.

A request authorized before revocation may complete.

A request after revocation rejects.

No cached middleware-only authorization is treated as sufficient.

---

# 5. Invite claim concurrency

## Two users claim same invite

PASS.

Both serialize through:
- SessionRuntime;
- then exact invite row.

First successful claim terminalizes invite.

Second revalidation sees non-ACTIVE state.

## Same AuthSubject claims two different invites in same Session

PASS.

SessionRuntime serializes claims and lifetime UNIQUE on Session/AuthSubject independently protects the invariant.

## Two AuthSubjects target same ParticipantRef

PASS.

Canonical target-slot validation + lifetime UNIQUE Session/ParticipantRef protect the result.

---

# 6. Invite issuance concurrency

PASS.

Order:

```
SessionRuntime
-> creator binding FOR SHARE
-> active invite row
```

serializes:
- creator revocation;
- concurrent regenerate/issue;
- target-slot canonical changes.

Partial UNIQUE active-invite index remains an additional database safeguard.

---

# 7. Expiry race

PASS.

The schema explicitly avoids transaction-start time as the claim authority after lock waits.

Expiry is evaluated after authoritative locks using current database wall time.

Policy is precise:

> if the invite is valid at the locked evaluation point, later wall-clock expiry during the same short transaction does not cancel the accepted in-flight claim.

This is operational timing, not canonical logical time.

---

# 8. Prelookup race

PASS.

Token prelookup only discovers a candidate SessionId.

After locking SessionRuntime, the transaction re-locks/revalidates the exact token hash row.

Revocation/claim between prelookup and authoritative lock therefore causes rejection/revalidation, not stale authority.

---

# 9. Invite relational audit chain

PASS.

Creator:

```
invite.created_by_binding_id
-> binding
```

Claim:

```
invite.claimed_binding_id
-> binding
-> participation_origin_transition_key
-> canonical transition
```

No duplicate transition key is stored on invite.

The row-level check rejects creator binding as claimed binding.

---

# 10. Semantic origin verification

PASS as explicit non-SQL invariant.

SQL cannot inspect canonical event semantics through a normal FK.

The required validators cover:

- binding ParticipantRef versus ParticipantBound transition/resulting state;
- invite target slot versus claimed ParticipantBound semantics.

The schema correctly does not pretend referential integrity alone proves domain meaning.

---

# 11. AuthSubject representation

PASS under accepted platform.

Current Supabase Auth JWT reference defines `sub` as the user-id UUID.

Using PostgreSQL UUID for AuthSubject is valid for the accepted Supabase baseline.

A future Auth-provider migration that cannot preserve UUID subjects is an operational access-schema migration; canonical ParticipantRef remains unchanged.

---

# 12. Auth deletion / stale JWT

PASS at engine authorization layer.

Binding alone grants no authority.

Valid JWT + ACTIVE binding are both required.

If engine-managed deletion/recovery revokes binding first, engine access stops immediately under the binding-lock model even if an already-issued upstream JWT has not expired.

Out-of-band Auth deletion can leave stale binding history but cannot create new authority by itself.

---

# 13. Realtime revocation cache

PASS because of payload boundary.

Current Supabase Realtime documentation states private-channel authorization is cached for the connection and refreshed on connection/new JWT.

Therefore an already-connected revoked user can temporarily remain subscribed.

That would be unacceptable if Realtime carried PlayerView/private game data.

It does not.

Only content-free invalidation is sent, and authoritative API refetch rechecks the current binding.

Do not use Realtime connection state as access authority.

---

# 14. Trigger/privilege model

PASS at design level.

Runtime:
- no DELETE;
- column-limited lifecycle UPDATE;
- lifecycle triggers protect immutable identity/token fields.

Trigger functions remain provider-neutral and are not public APIs.

Concrete GRANT/trigger DDL still requires execution testing on the selected target.

---

# 15. Lost invite response

PASS as explicit MVP trade-off.

Raw token cannot be reconstructed.

No reversible token storage is introduced.

The UI must use an explicit Regenerate action rather than invisible automatic retry.

---

# 16. Terminal Session + ACTIVE binding

PASS.

Access authorization and gameplay legality are separate.

Binding may remain for recap/resume-retention purposes.

Canonical Session state rejects illegal gameplay operations.

No duplicate terminal truth is introduced.

---

# 17. Future recovery

EXPECTED SCHEMA CHANGE.

Recovery/rebind is intentionally impossible under v0.3 lifetime uniqueness.

Implementing it later requires:
- explicit recovery authority model;
- replacement of lifetime UNIQUE constraints with a historical/active design;
- audit semantics;
- abuse/recovery threat model;
- migration/ADR review.

This is preferable to shipping an unused privileged state now.

---

# 18. Current-source verification

Current PostgreSQL documentation supports:
- partial UNIQUE subset indexes;
- FOR SHARE blocking concurrent row UPDATE/DELETE;
- distinction between transaction-start timestamps and current wall-clock functions.

Current Supabase documentation supports:
- JWT `sub` as user-id UUID;
- anonymous authenticated users;
- cached Realtime authorization per connection/JWT lifecycle.

These are deployment assumptions and remain part of the later integration spike.

---

# 19. Residual risks

No material architecture defect remains.

Residual implementation risks:
- incorrect GRANT mapping;
- trigger implementation bug;
- missing semantic validator call;
- accidentally logging raw invite token;
- too-long Auth JWT lifetime increasing stale Realtime invalidation exposure;
- future product pressure for account recovery.

All have explicit tests/revisit paths.

---

# 20. Required tests after acceptance

Before migration rollout:

1. binding FOR SHARE blocks revocation UPDATE;
2. revocation first makes later authorization query return no ACTIVE binding;
3. two invite claims produce one winner;
4. lifetime UNIQUE blocks second binding history;
5. two issue/regenerate transactions produce one ACTIVE invite;
6. expiry check occurs after lock wait;
7. lifecycle triggers reject immutable-field mutation;
8. CLAIMED invite must point to same-Session binding;
9. semantic validator detects wrong ParticipantRef/origin;
10. semantic validator detects wrong target slot;
11. browser roles cannot access `access`;
12. raw invite token never appears in structured logs;
13. Realtime invalidation contains no private content.

---

# 21. Verdict

**ACCESS_SCHEMA_v0.3 is technically READY FOR ACCEPTANCE.**

No v0.4 is justified before executable DDL/integration testing.

After explicit acceptance:

1. mark Access Schema v0.3 accepted;
2. design the Supabase-specific private Realtime authorization/send helpers;
3. perform Supabase/Railway connectivity/security spikes;
4. materialize provider-neutral + provider-specific migration files;
5. run DDL/concurrency integration tests;
6. only then create the implementation skeleton and resolver/replay harness.
