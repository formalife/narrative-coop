# ADR-015 — Guest Identity and Session Participation

**Status:** PROPOSED  
**Date:** 2026-10-02

## Context

The MVP must support two-player guest play without mandatory PII while preserving a strict separation between external authentication identity and canonical gameplay identity.

The accepted Domain Model already distinguishes canonical ParticipantRef/Character state from account identity, but the runtime still needs a durable way to authorize an authenticated browser to act for one ParticipantRef.

Anonymous guest access also introduces invite-claim, abuse, recovery and privacy concerns.

## Decision

Use three distinct identities:

1. **AuthSubject** — external Supabase Auth subject/JWT user id; operational security identity.
2. **ParticipantRef** — opaque internal gameplay participant identity used by canonical ParticipationState.
3. **CharacterEntityId** — canonical in-world Character identity.

Invariant:

```
AuthSubject != ParticipantRef != CharacterEntityId
```

### Guest authentication

Use Supabase anonymous sign-in for MVP guest play.

- no mandatory email/name/phone;
- CAPTCHA/Turnstile required for anonymous account creation;
- rate-limit Session creation/invite claims;
- anonymous identity may later link a permanent authentication method.

### Access binding

Persist a noncanonical internal mapping:

```
session_principal_bindings
  session_id
  participant_ref
  auth_subject
  binding_status
  created_at
  optional revoked_at
```

This relation is security authority for who may authenticate as a ParticipantRef.

It is not canonical world state.

### Atomic participant claim

Initial binding of an AuthSubject to a ParticipantRef and the corresponding canonical ParticipantBound transition must commit atomically in PostgreSQL.

No AuthSubject is stored in canonical gameplay state/hash.

### Invitations

Guest partner invitations use a high-entropy one-time bearer token:

- 256 random bits;
- only token hash stored;
- expiration required;
- single claim;
- raw token never logged.

Prefer invite token in SPA URL fragment and exchange it over HTTPS.

### Recovery

MVP does not promise anonymous identity recovery after:
- browser storage deletion;
- sign-out;
- device switch.

Players may later link a permanent auth method.

Cross-device/rebind recovery is a future explicit access workflow, not a canonical Character rewrite.

## Alternatives Considered

1. Use Auth user id directly as ParticipantRef.
2. Require permanent account/email before play.
3. Use only invite bearer tokens without authenticated users.
4. Anonymous AuthSubject + separate ParticipantRef binding.

## Why Rejected

### Auth id as ParticipantRef

Leaks provider/account identity into canonical gameplay and makes later auth migration/account linking harder.

### Mandatory account

Adds friction and PII before demonstrated product need.

### Bearer token only

Weakens resumable authorization/session security and makes access revocation/device state harder to manage.

## Consequences

Benefits:
- minimal PII;
- canonical game identity remains provider-independent;
- Auth upgrade does not rewrite gameplay history;
- invite claims can be transactional and auditable.

Costs:
- requires operational access-binding tables;
- anonymous-user cleanup policy;
- guest device-loss limitation.

## Risks

- anonymous account abuse;
- invite token leakage;
- stale access bindings after auth deletion;
- accidental authorization using only "authenticated" status;
- user confusion after clearing browser data.

## Revisit Conditions

Revisit if:
- cross-device guest recovery becomes a key conversion/retention issue;
- permanent accounts become mandatory;
- Supabase Auth is replaced;
- multi-participant/multi-device control becomes a product feature.
