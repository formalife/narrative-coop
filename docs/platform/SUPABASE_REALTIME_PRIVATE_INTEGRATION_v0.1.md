# SUPABASE REALTIME PRIVATE INTEGRATION v0.1

**Status:** PROPOSED  
**Date:** 2026-10-02  
**Governing architecture:** Architecture Baseline v0.5 — ACCEPTED  
**Governing platform:** Implementation Platform v0.3 — ACCEPTED  
**Governing access schema:** Access Schema v0.3 — ACCEPTED  
**Purpose:** define provider-specific private Realtime authorization and invalidation without exposing private gameplay state.

---

# 1. Scope

This design covers only Supabase-specific Realtime glue:

- private channel topic naming;
- subscription authorization;
- database-originated invalidation send;
- helper-function privileges;
- failure/revocation behavior.

It does NOT own:
- canonical state;
- access binding authority;
- PlayerInteractionView content;
- Session progression.

---

# 2. Topic contract

Private topic:

```
session:<session_id>
```

Only one semantic event is required initially:

```
session_view_changed
```

Payload:

```json
{
  "session_id": "<uuid>"
}
```

No:
- StreamRevision;
- ParticipantRef;
- CharacterId;
- private PlayerView data;
- secret/knowledge data;
- pending action data;
- narrative text.

Reason:
PlayerInteractionView may change without canonical StreamRevision advance, and invalidation should reveal as little as possible.

---

# 3. Browser receive flow

1. Browser authenticates through Supabase Auth.
2. Supabase client sets Realtime auth from current user access token.
3. Browser joins private topic:
   `session:<session_id>`.
4. Realtime evaluates RLS authorization on `realtime.messages`.
5. On `session_view_changed`, browser calls authoritative engine API.
6. API revalidates current ACTIVE access binding and returns current PlayerInteractionView.

Realtime never substitutes for API authorization.

---

# 4. Authorization policy strategy

Do NOT grant all authenticated users access to all Session topics.

The policy must prove:

```
current auth.uid()
  -> ACTIVE access.session_principal_binding
  -> same SessionId encoded in realtime topic
```

Because browser roles must not receive direct SELECT on access tables, use a narrow SECURITY DEFINER boolean helper.

---

# 5. Provider helper

Conceptual helper:

```sql
create function access.can_receive_session_topic(
  p_auth_subject uuid,
  p_session_id uuid
)
returns boolean
language sql
stable
security definer
set search_path = ''
as $$
  select exists (
    select 1
    from access.session_principal_bindings b
    where b.auth_subject = p_auth_subject
      and b.session_id = p_session_id
      and b.binding_status = 'ACTIVE'
  );
$$;
```

Security requirements:

- owner is dedicated migration/privileged role;
- `search_path = ''`;
- fully qualified relations;
- PUBLIC EXECUTE revoked;
- EXECUTE granted only to the role required by Realtime policy evaluation;
- returns boolean only;
- no participant/private row data returned.

The helper is operational authorization glue, not gameplay logic.

---

# 6. Topic parsing in policy

Policy on `realtime.messages` should authorize SELECT only when:

- request role is authenticated;
- `private = true`;
- topic starts with exact `session:` prefix;
- suffix parses as UUID;
- `auth.uid()` is not null;
- helper returns true for parsed SessionId.

Avoid permissive substring matching.

Conceptual policy logic:

```sql
auth.uid() is not null
and realtime.messages.private is true
and realtime.messages.topic ~ '^session:[0-9a-fA-F-]{36}$'
and access.can_receive_session_topic(
  auth.uid(),
  split_part(realtime.messages.topic, ':', 2)::uuid
)
```

Exact UUID parser expression is implementation detail and must fail closed on malformed topic.

---

# 7. Client send permission

Normal gameplay clients do NOT need to send Broadcast messages.

Therefore do not create a normal authenticated INSERT policy for Realtime messages.

Client capability is receive-only.

This reduces spoofed invalidation traffic.

---

# 8. Worker invalidation send

Worker must not hold Supabase secret/service-role key.

Use a narrow database function:

```
platform.notify_session_view_changed(session_id uuid)
```

that internally calls:

```
realtime.send(
  payload := jsonb_build_object('session_id', session_id),
  event := 'session_view_changed',
  topic := 'session:' || session_id::text,
  private := true
)
```

Worker gets EXECUTE on this function only.

---

# 9. Sender helper security

Conceptual:

```sql
create function platform.notify_session_view_changed(
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

Requirements:

- fixed event name;
- fixed private=true;
- topic derived internally;
- payload constructed internally;
- no arbitrary event/topic/payload parameters;
- PUBLIC EXECUTE revoked;
- EXECUTE only for `engine_worker`;
- no direct gameplay table mutation.

---

# 10. Why SECURITY DEFINER is acceptable here

It is allowed only because:

- browser cannot call the sender helper;
- helper exposes one fixed capability;
- helper performs no canonical mutation;
- PUBLIC execute is revoked;
- search_path is empty;
- parameters cannot select arbitrary topic/event/payload.

For subscription authorization helper:
- same narrow boolean-only principle applies.

Do not use SECURITY DEFINER as a general permission workaround.

---

# 11. Realtime authorization cache

Current Supabase Realtime authorization is not re-evaluated for every message.

Therefore:
- a binding revoked after channel join may leave the socket subscribed until reconnect/token refresh;
- this is not acceptable as the primary access boundary.

Mitigation is architectural:
- Realtime payload is content-free;
- authoritative API always revalidates binding;
- no private data is ever broadcast.

Revoked users may transiently receive only:
`session_view_changed(session_id)`
for a Session they already know.

---

# 12. SessionId disclosure

Topic/payload include SessionId.

SessionId is already known to a participant with a valid binding.

Do not treat SessionId itself as a secret.

However:
- malformed/unauthorized topics must fail closed;
- subscription policy must not become an oracle for enumerating Session membership beyond channel join success/failure.

---

# 13. Failure behavior

If Realtime send fails:
- canonical state remains committed;
- worker may retry noncanonical invalidation;
- clients can still refetch/poll API.

If subscription authorization fails:
- client falls back to API polling/resume.
- do not broaden RLS to restore convenience.

Realtime outage does not block canonical command processing.

---

# 14. Provider-specific repository placement

Supabase-specific SQL belongs under:

```
platform/supabase/sql/
  realtime_authorization.sql
  realtime_notifications.sql
```

Not under provider-neutral core semantics.

The functions may reference:
- `realtime.messages`;
- `realtime.send`;
- `auth.uid()`.

They must not redefine:
- `engine` schema;
- canonical event/state contracts;
- access binding semantics.

---

# 15. Required tests

Before acceptance:

1. bound anonymous user may join its Session topic;
2. same user cannot join unrelated Session topic;
3. revoked binding blocks new subscription;
4. already-connected revoked subscriber may remain connected but receives no private content;
5. unauthenticated client cannot join;
6. malformed topic fails closed;
7. authenticated client cannot send broadcast;
8. engine_worker can execute fixed notification helper;
9. engine_runtime cannot execute sender helper unless explicitly required;
10. sender helper cannot emit arbitrary topic/event/payload;
11. PUBLIC cannot execute either SECURITY DEFINER helper;
12. helper search_path is empty;
13. browser roles cannot SELECT access tables directly;
14. notification failure does not affect canonical transaction;
15. worker retry cannot create canonical duplication.

