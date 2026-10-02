# Access Schema v0.3 Acceptance

**Date:** 2026-10-02  
**Decision:** ACCEPTED  
**Accepted schema:** ACCESS_SCHEMA_v0.3  
**Supersedes:** ACCESS_SCHEMA_v0.2

## Accepted invariants

- AuthSubject remains operational/noncanonical.
- One lifetime binding per Session/AuthSubject in MVP.
- One lifetime binding per Session/ParticipantRef in MVP.
- No MVP recovery/rebind.
- Authorized private/write transactions revalidate and lock ACTIVE binding.
- Revocation linearizes through row-lock conflict.
- Invite plaintext is never persisted.
- Invite claim atomically creates binding + canonical ParticipantBound.
- Expiry is evaluated at the locked wall-clock claim point.
- SQL relations prove context/identity; semantic validators prove canonical ParticipantBound/slot meaning.
- Realtime never carries private access-sensitive state.

## Evidence

- `RED_TEAM_ACCESS_SCHEMA_v0.3.md`
- postmortems v0.1 and v0.2.
