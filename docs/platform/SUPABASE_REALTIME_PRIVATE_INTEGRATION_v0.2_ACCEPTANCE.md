# Supabase Realtime Private Integration v0.2 Acceptance

**Date:** 2026-10-02  
**Decision:** ACCEPTED  
**Accepted integration:** SUPABASE_REALTIME_PRIVATE_INTEGRATION_v0.2  
**Supersedes:** v0.1

## Accepted boundary

- Realtime is content-free invalidation only.
- Browser identity/topic are derived inside provider authorization using auth.uid() and realtime.topic().
- Browser roles receive no access/engine schema privileges.
- Realtime provider glue lives in platform_supabase.
- Normal clients cannot Broadcast-send.
- engine_worker gets only a fixed Session invalidation capability.
- Realtime public access must be disabled.

## Evidence

- postmortem v0.1;
- postmortem/red-team v0.2;
- real target helper and sender SQL spikes.

## Remaining deployment verification

Acceptance does not waive:
- real anonymous JWT test;
- private WebSocket join;
- public-access-off dashboard check;
- exact custom worker LOGIN invocation;
- security advisors after SQL installation.
