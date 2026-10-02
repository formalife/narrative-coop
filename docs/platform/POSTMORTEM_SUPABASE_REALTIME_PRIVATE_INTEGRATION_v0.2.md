# POSTMORTEM — Supabase Realtime Private Integration v0.2

**Date:** 2026-10-02  
**Reviewed artifact:** `docs/platform/SUPABASE_REALTIME_PRIVATE_INTEGRATION_v0.2.md`  
**Verdict:** READY FOR ACCEPTANCE

## Executive verdict

v0.2 fixes the material issues found in v0.1:

- uses `realtime.topic()` instead of relying on `realtime.messages.topic`;
- keeps browser roles out of the accepted `access` schema;
- moves provider glue into private `platform_supabase`;
- derives AuthSubject internally from `auth.uid()`;
- constrains receive authorization to Broadcast only;
- grants clients no Broadcast send capability;
- keeps worker send capability narrow and fixed;
- explicitly requires Realtime public access to be disabled.

No accepted architecture or access-schema decision needs modification.

---

# 1. Real-target topic helper verification

The staging target's actual definition of:

`realtime.topic()`

was introspected.

It currently reads:

`current_setting('realtime.topic', true)`

and returns the requested topic text.

Result:
**PASS** for using `realtime.topic()` as the authorization input.

This confirms the design matches the current hosted implementation, not only documentation.

---

# 2. Real-target auth identity verification

The staging target's actual `auth.uid()` implementation was introspected.

It currently derives the UUID subject from request JWT settings.

Result:
**PASS** for deriving AuthSubject inside the provider helper rather than accepting a caller-supplied identity argument.

---

# 3. Authorization helper behavioral spike

A temporary provider/access schema was created on the staging target.

The zero-argument authorization helper was tested by simulating request JWT/topic settings.

Observed:

- unauthenticated -> false;
- malformed topic -> false;
- ACTIVE matching binding -> true;
- wrong Session -> false;
- REVOKED binding -> false.

Result:
**PASS**.

All temporary schemas were deleted after testing.

---

# 4. Provider-schema boundary

v0.2 no longer requires authenticated browser roles to receive USAGE on `access`.

Instead:

- `authenticated` gets USAGE only on `platform_supabase`;
- exact EXECUTE only on the boolean helper;
- no access-table SELECT/USAGE is granted.

Result:
**PASS** against Access Schema v0.3 constraints.

---

# 5. Broadcast-only policy

Current Supabase Realtime documentation identifies:

`realtime.messages.extension = 'broadcast'`

as the Broadcast authorization discriminator.

v0.2 includes that constraint.

Result:
**PASS**.

No Presence permission is implied.

---

# 6. Client send capability

v0.2 creates no authenticated INSERT policy on `realtime.messages`.

Normal gameplay clients are receive-only.

Result:
**PASS** by design.

Actual project policy installation remains a deployment step.

---

# 7. Worker sender wrapper

A temporary fixed sender wrapper was created on the staging target.

Verified catalog privileges:

- PUBLIC execute: false;
- worker role execute: true;
- worker provider-schema usage: true.

Calling the wrapper as the administrative connection successfully reached `realtime.send`.

Result:
**PASS** for wrapper SQL/function shape and grant model.

The connector execution role could not `SET ROLE` to the temporary worker role, so an invocation under the exact custom worker login was not possible through this tool path.

That exact-role execution remains an external connection test.

---

# 8. Sender capability scope

The wrapper exposes only:

- SessionId input;
- fixed event `session_view_changed`;
- fixed topic prefix;
- fixed private=true;
- fixed minimal payload.

No arbitrary:
- topic;
- event;
- payload;
- public/private flag

can be selected by the worker caller.

Result:
**PASS**.

---

# 9. Realtime public-access setting

Current Supabase documentation requires disabling **Allow public access** to enforce private-channel authorization as intended.

The current connector does not expose that dashboard/platform setting directly.

Result:
**PENDING DEPLOYMENT CONFIGURATION CHECK**.

This is an implementation/configuration gate, not a design defect.

---

# 10. Live WebSocket authorization

Still not tested end-to-end because the spike has not yet configured:

- anonymous Auth flow;
- actual access binding tables/migrations;
- Realtime policy on production-shaped access data;
- browser/WebSocket client.

Required later:
- bound anonymous user joins Session topic;
- unrelated user fails;
- malformed/unbound user fails;
- revoked user fails on a new join.

Result:
**PENDING**.

---

# 11. Realtime revocation cache

Current documentation confirms authorization is cached for the connection and refreshed on connection/new JWT.

The design already accounts for this:

- broadcasts contain no private gameplay data;
- API access revalidates binding;
- Realtime presence is not an authorization source.

Result:
**PASS**.

---

# 12. Security-definer review

The design uses SECURITY DEFINER only for narrow provider capabilities.

Required implementation properties remain:

- dedicated owner;
- empty search_path;
- fully-qualified relation/function references;
- PUBLIC EXECUTE revoked immediately;
- exact role grants only;
- Supabase security advisors run after installation.

Result:
**PASS AS DESIGN / VERIFY AFTER MIGRATION**.

---

# 13. What changed from v0.1

Root causes corrected:

1. provider API semantics were approximated instead of using the documented `realtime.topic()` helper;
2. function placement accidentally weakened the accepted schema privilege boundary;
3. authorization identity was unnecessarily parameterized;
4. provider-level public-channel setting was omitted.

v0.2 removes all four issues.

---

# 14. Verdict

**SUPABASE_REALTIME_PRIVATE_INTEGRATION_v0.2 is technically READY FOR ACCEPTANCE.**

No v0.3 is justified before applying/testing the provider SQL on the real accepted access schema.

Remaining deployment tests:
- public access OFF;
- exact worker-login execution;
- real anonymous JWT;
- private WebSocket join;
- revoke/rejoin behavior;
- Supabase security advisors.

These are implementation verification tasks, not unresolved architecture design.

