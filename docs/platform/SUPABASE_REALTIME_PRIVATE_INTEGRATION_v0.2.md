# SUPABASE REALTIME PRIVATE INTEGRATION v0.2

**Status:** ACCEPTED  
**Date:** 2026-10-02  
**Supersedes:** SUPABASE_REALTIME_PRIVATE_INTEGRATION_v0.1  
**Governing architecture:** Architecture Baseline v0.5 — ACCEPTED  
**Governing platform:** Implementation Platform v0.3 — ACCEPTED  
**Governing access schema:** Access Schema v0.3 — ACCEPTED

---

# 1. Provider boundary

All Supabase-specific database glue lives in a private provider schema:

`platform_supabase`

It is not a canonical engine schema and is not exposed through the browser Data API.

Browser role `authenticated` may receive:
- USAGE on `platform_supabase`;
- EXECUTE on one exact boolean authorization function.

It receives no privilege on:
- `engine`;
- `access`;
- generic sender functions.

---

# 2. Private topic contract

Topic:

`session:<session_id>`

Broadcast event:

`session_view_changed`

Payload:

```json
{
  "session_id": "<uuid>"
}
```

No private gameplay content is transmitted.

Realtime is invalidation only.

---

# 3. Receive authorization helper

Use one zero-argument helper:

`platform_supabase.can_receive_current_session_topic() -> boolean`

The helper obtains:
- AuthSubject from `auth.uid()`;
- requested Channel topic from `realtime.topic()`.

The caller cannot supply either identity.

Conceptual implementation:

```sql
create function platform_supabase.can_receive_current_session_topic()
returns boolean
language plpgsql
stable
security definer
set search_path = ''
as $$
declare
  v_auth_subject uuid;
  v_topic text;
  v_session_text text;
  v_session_id uuid;
begin
  v_auth_subject := auth.uid();
  if v_auth_subject is null then
    return false;
  end if;

  v_topic := realtime.topic();

  if v_topic is null or v_topic !~ '^session:[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$' then
    return false;
  end if;

  v_session_text := substring(v_topic from 9);

  begin
    v_session_id := v_session_text::uuid;
  exception
    when invalid_text_representation then
      return false;
  end;

  return exists (
    select 1
    from access.session_principal_bindings b
    where b.auth_subject = v_auth_subject
      and b.session_id = v_session_id
      and b.binding_status = 'ACTIVE'
  );
end;
$$;
```

The exception block is defensive; regex validation should already reject malformed UUIDs.

---

# 4. Receive policy

Create SELECT policy on `realtime.messages`:

```sql
create policy "narrative coop receive private session broadcast"
on realtime.messages
for select
to authenticated
using (
  realtime.messages.extension = 'broadcast'
  and (select platform_supabase.can_receive_current_session_topic())
);
```

Do not create an authenticated INSERT policy.

Normal clients are receive-only.

---

# 5. Schema/function privileges

Immediately after function creation:

```sql
revoke all on function
  platform_supabase.can_receive_current_session_topic()
from public;

grant usage on schema platform_supabase to authenticated;

grant execute on function
  platform_supabase.can_receive_current_session_topic()
to authenticated;
```

Do not grant:
- access schema USAGE;
- access table SELECT;
- platform sender EXECUTE.

The SECURITY DEFINER owner must have only the privileges needed for the lookup.

---

# 6. Worker sender helper

Use a separate fixed-capability function:

`platform_supabase.notify_session_view_changed(p_session_id uuid) -> void`

Conceptual:

```sql
create function platform_supabase.notify_session_view_changed(
  p_session_id uuid
)
returns void
language plpgsql
security definer
set search_path = ''
as $$
begin
  perform realtime.send(
    jsonb_build_object('session_id', p_session_id),
    'session_view_changed',
    'session:' || p_session_id::text,
    true
  );
end;
$$;
```

Hardening:
- PUBLIC EXECUTE revoked;
- schema USAGE + function EXECUTE only for `engine_worker`;
- fixed event;
- fixed private=true;
- topic derived from UUID;
- payload constructed internally.

The worker cannot choose arbitrary topic/event/payload.

---

# 7. Public-channel setting

Deployment requirement:

Supabase Realtime setting:

**Allow public access = OFF**

Client still creates the channel with:

`private: true`

Both are required.

Do not treat RLS policy alone or client private flag alone as the complete deployment control.

---

# 8. Browser flow

1. player authenticates using Supabase Auth;
2. client calls Realtime setAuth/current session integration;
3. client joins `session:<session_id>` with `private: true`;
4. Realtime RLS invokes zero-argument authorization helper;
5. helper derives auth/topic itself and checks ACTIVE binding;
6. client receives only `session_view_changed`;
7. client fetches current PlayerInteractionView from engine-api;
8. engine-api independently revalidates ACTIVE binding under accepted DB lock policy.

---

# 9. Revocation/cache behavior

Realtime permissions are cached for the connection/JWT lifecycle.

Therefore:
- binding revocation blocks new channel authorization;
- an already-authorized socket may remain subscribed until token refresh/expiry/reconnect.

This is safe only because:
- broadcast payload has no private gameplay content;
- API is the actual private-state boundary.

Do not attempt to use Realtime disconnect state as authoritative access revocation.

---

# 10. Failure semantics

Realtime send failure:
- cannot roll back already-committed canonical state;
- worker records/retries noncanonical delivery according to Outbox task semantics.

Realtime join failure:
- client falls back to API refetch/polling;
- never broaden policy as availability workaround.

---

# 11. Provider SQL placement

Canonical repository:

```
platform/supabase/sql/
  001_realtime_private_authorization.sql
  002_realtime_private_notifications.sql
```

These are provider-specific SQL artifacts.

They do not belong to provider-neutral `db/migrations/` if they reference:
- `realtime.*`;
- `auth.*`.

---

# 12. Security tests

Before acceptance:

1. unauthenticated helper returns false;
2. malformed topic returns false;
3. valid topic + no binding returns false;
4. valid topic + wrong AuthSubject returns false;
5. ACTIVE binding returns true;
6. REVOKED binding returns false on new authorization;
7. authenticated role has no access-schema privilege;
8. authenticated role cannot execute sender helper;
9. PUBLIC executes neither helper;
10. engine_worker can execute sender helper only;
11. sender emits only private fixed invalidation;
12. no authenticated INSERT policy exists on Realtime messages;
13. Realtime public access setting is OFF;
14. private channel join succeeds only for bound user;
15. private channel join fails for unrelated user;
16. security advisors report no new material issue.



---

# Acceptance record

Supabase Realtime Private Integration v0.2 was explicitly accepted on 2026-10-02.

Acceptance evidence:
- `POSTMORTEM_SUPABASE_REALTIME_PRIVATE_INTEGRATION_v0.2.md`
- real-target helper/sender spikes on the Formalife Supabase staging project.

Binding provider-specific decisions:
- private Broadcast only;
- receive authorization uses `realtime.topic()` + `auth.uid()`;
- provider helper schema is `platform_supabase`, not `access`;
- authenticated clients receive no Broadcast-send policy;
- worker emits fixed private invalidation through a narrow DB wrapper;
- Supabase Realtime public access must be disabled at deployment.
