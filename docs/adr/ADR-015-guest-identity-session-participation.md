# ADR-015 — Guest Identity and Session Participation

**Status:** PROPOSED  
**Date:** 2026-10-02

## Context

The MVP must support two-player guest play without mandatory PII while preserving a strict separation between authentication identity and canonical gameplay identity.

Anonymous guest access also introduces:
- invite-claim races;
- abuse;
- stale/deleted auth subjects;
- device-loss limitations;
- cleanup/retention concerns.

The accepted Domain Model already separates ParticipantRef from account identity. The implementation platform must now define the operational authorization boundary without putting provider identity into canon.

## Decision

Use three distinct identities:

1. **AuthSubject** — Supabase Auth JWT subject/user id. Operational security identity.
2. **ParticipantRef** — opaque internal gameplay participant identity used by canonical ParticipationState.
3. **CharacterEntityId** — canonical in-world character identity.

Invariant:

```
AuthSubject != ParticipantRef != CharacterEntityId
```

### Anonymous guest authentication

Use Supabase anonymous sign-in for first-play guests.

Requirements:
- Turnstile/CAPTCHA enabled;
- no mandatory email/name/phone;
- application quotas on Session/invite operations;
- anonymous users may later link a permanent identity.

### Internal access schema

Persist noncanonical authorization state in internal `access` schema:

- `access.session_principal_bindings`;
- `access.session_invites`.

Neither table is canonical world state or browser-exposed.

### Session principal binding

An ACTIVE binding maps:

```
(session_id, auth_subject) -> participant_ref
```

MVP uniqueness:

- one ACTIVE AuthSubject controls at most one ParticipantRef per Session;
- one ParticipantRef has at most one ACTIVE AuthSubject.

Authorization always starts from a valid verified JWT and then resolves the binding.

No hard FK to Supabase `auth.users`.

### Atomicity

Initial access binding and the matching canonical ParticipantBound transition must commit atomically.

Preferred creator Session transaction may create revision-0 bootstrap and revision-1 ParticipantBound in the same PostgreSQL transaction.

### Invitations

Use one-time high-entropy bearer invite:

- 256 random bits;
- cryptographic hash stored;
- token hash unique;
- expiration required;
- terminal CLAIMED/REVOKED/EXPIRED states;
- at most one ACTIVE invite per target participant slot by default;
- raw token excluded from logs/analytics.

Prefer raw token in SPA URL fragment and scrub it immediately after reading.

### Cleanup

Do not delete an anonymous AuthSubject while it has an ACTIVE binding to a nonterminal/recoverable Session.

Anonymous cleanup begins only after retention policy is accepted.

### Recovery

MVP does not promise access recovery after:
- sign-out;
- browser storage deletion;
- device switch.

Supported upgrade path is linking a permanent identity to the current anonymous user.

Merging/rebinding to a different existing account is future explicit access-recovery work and does not rewrite canonical character history.

## Alternatives Considered

1. Auth user id directly as ParticipantRef.
2. Mandatory permanent accounts.
3. Invite bearer token without authenticated user.
4. Anonymous Auth + separate operational binding.

## Why Rejected

### Auth id as ParticipantRef

Provider identity leaks into canonical gameplay and complicates provider migration/auth merging.

### Mandatory accounts

Adds PII/friction before demonstrated need.

### Bearer-only authorization

Weakens durable authorization, revocation and resume semantics.

## Consequences

Benefits:
- minimal PII;
- canonical identity remains provider-independent;
- permanent identity linking does not rewrite gameplay;
- one-Session/one-role default is enforceable.

Costs:
- new operational `access` schema;
- anonymous cleanup policy;
- no initial cross-device recovery.

## Risks

- anonymous-account abuse;
- invite leakage;
- stale bindings;
- overly broad "authenticated" authorization;
- guest access loss after local credential loss.

## Revisit Conditions

Revisit if:
- cross-device recovery becomes material;
- accounts become mandatory;
- multi-device/multi-controller participation becomes a product feature;
- Supabase Auth is replaced;
- one human controlling multiple roles becomes an intentional game mode.
