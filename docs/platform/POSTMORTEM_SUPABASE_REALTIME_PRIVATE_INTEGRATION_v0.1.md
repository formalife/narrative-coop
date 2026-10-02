# POSTMORTEM — Supabase Realtime Private Integration v0.1

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/platform/SUPABASE_REALTIME_PRIVATE_INTEGRATION_v0.1.md`  
**Verdict:** MODIFY

## Executive verdict

The high-level design is correct:

- private Broadcast only;
- content-free invalidation;
- no client send capability;
- worker emits through a narrow DB function;
- API remains authoritative for private PlayerView access.

However v0.1 contains two material integration mistakes and one missing provider setting.

No accepted core architecture or Access Schema decision must change.

---

# 1. Topic source in RLS policy is wrong

## v0.1

The policy logic referenced:

`realtime.messages.topic`

## Current Supabase contract

Supabase Realtime Authorization documents the helper:

`realtime.topic()`

as the source of the Channel topic being joined.

Realtime evaluates authorization by running an RLS check against `realtime.messages`, but the request topic should be read through `realtime.topic()`.

## Correction

Use:

`select realtime.topic()`

for topic authorization.

Do not depend on a synthetic/current row value in `realtime.messages.topic` during the authorization probe.

---

# 2. Helper placement conflicts with accepted access-schema privilege boundary

## v0.1

Authorization helper was proposed in:

`access.can_receive_session_topic(...)`

For role `authenticated` to invoke a function in schema `access`, it normally requires schema USAGE.

But Access Schema v0.3 explicitly says browser/authenticated roles receive no direct schema/table privilege on `access`.

## Correction

Create a separate provider-specific, non-exposed schema:

`platform_supabase`

Place Realtime helper functions there.

Grant:
- USAGE on `platform_supabase` to `authenticated`;
- EXECUTE only on the specific boolean authorization helper.

Do NOT grant any privilege on `access` to browser roles.

The helper runs as SECURITY DEFINER and reads `access.session_principal_bindings` internally.

This preserves the accepted access boundary.

---

# 3. Authorization helper should not accept caller-supplied AuthSubject

## v0.1

Helper accepted:

`p_auth_subject uuid`

Even though policy would pass `auth.uid()`, a callable helper should avoid accepting an identity argument when request identity can be derived internally.

## Correction

Use:

`platform_supabase.can_receive_session_topic(p_session_id uuid)`

Internally:

```
auth.uid()
-> ACTIVE access binding
-> p_session_id
```

This removes identity spoofing from the helper contract.

The function returns false when `auth.uid()` is NULL.

---

# 4. Broadcast extension must be constrained

Supabase documents `realtime.messages.extension = 'broadcast'` for Broadcast authorization.

## Correction

SELECT policy must include:

`realtime.messages.extension = 'broadcast'`

Do not accidentally authorize Presence.

No INSERT policy is created for normal clients.

---

# 5. Public-channel platform setting is missing

Current Supabase Realtime Authorization documentation states that private-channel enforcement requires disabling:

**Allow public access**

in Realtime settings.

## Correction

Deployment checklist must explicitly verify:
- Realtime public access disabled;
- client channel created with `private: true`;
- RLS SELECT policy installed.

Do not assume `private: true` alone is sufficient platform hardening.

---

# 6. Topic parsing

Regex-only validation of a UUID-shaped suffix is unnecessary complexity if the policy can parse deterministically.

Preferred helper contract:

```
platform_supabase.session_id_from_realtime_topic(topic text) -> uuid/null
```

or inline guarded parsing.

But avoid exceptions from malformed topics.

## Correction

Do not cast arbitrary topic suffix directly to UUID in the RLS predicate if malformed input can raise an exception.

Use a fail-closed parser helper that:
- checks exact `session:` prefix;
- validates UUID syntax;
- returns NULL for malformed topic.

Then authorization helper receives the parsed UUID.

This is provider glue, not domain logic.

---

# 7. SECURITY DEFINER hardening

Preserve and strengthen:

- owner is migration/privileged role;
- `SET search_path = ''`;
- fully-qualified `auth.uid()` and access-table references;
- PUBLIC EXECUTE revoked immediately;
- only exact required roles receive EXECUTE;
- no arbitrary SQL/topic/payload arguments.

Run Supabase security advisors after applying the provider-specific SQL.

---

# 8. Sender wrapper

v0.1 sender direction is sound.

Keep:

`platform_supabase.notify_session_view_changed(session_id uuid)`

It constructs:
- fixed event `session_view_changed`;
- fixed topic `session:<uuid>`;
- fixed private=true;
- minimal payload.

Worker only gets EXECUTE on this wrapper.

Do not grant worker INSERT on `realtime.messages` directly if the wrapper can encapsulate the capability.

---

# 9. Realtime policy cache

PASS.

Revocation is not immediately re-evaluated on every message.

This remains acceptable only because:
- payload contains no private state;
- authoritative API refetch revalidates binding;
- JWT expiry eventually forces disconnect if token is not refreshed.

This residual behavior must remain documented.

---

# 10. Verdict

**SUPABASE_REALTIME_PRIVATE_INTEGRATION_v0.1 is NOT ready for acceptance.**

Create v0.2 with:

- `realtime.topic()`;
- provider-specific `platform_supabase` schema;
- no browser privileges on `access`;
- helper derives AuthSubject internally via `auth.uid()`;
- Broadcast-only extension check;
- fail-closed topic parser;
- explicit Realtime public-access-off deployment setting;
- narrow fixed sender wrapper.

