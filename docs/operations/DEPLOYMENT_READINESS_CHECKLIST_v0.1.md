# DEPLOYMENT READINESS CHECKLIST v0.1

**Status:** ACTIVE GATE  
**Date:** 2026-10-02  
**Architecture:** v0.6 — ACCEPTED  
**Purpose:** distinguish manual provider configuration from agent-executable integration work.

## A. Manual owner actions

### A1 — Railway organization workspace

Required:
- create or expose a Railway Workspace representing Formalife;
- connected Railway user `formalife-personal` must be a Workspace **Admin** or **Member**;
- workspace must appear in Railway workspace listing.

Why:
- Member/Admin can create projects/services/variables;
- current connector sees only the personal workspace;
- Narrative Co-op infrastructure must not be created under personal ownership.

Acceptance evidence:
- Railway connector lists a workspace named/owned for Formalife with type `team`.

### A2 — Supabase anonymous sign-ins

Project:
- `narrative-coop-staging`
- ref: `ylqipkcqypmtsnbflkpv`

Required:
- enable Anonymous Sign-Ins.

Acceptance evidence:
- a real `signInAnonymously()` succeeds with publishable key.

### A3 — Supabase CAPTCHA / Turnstile

Required:
- create/configure Cloudflare Turnstile (preferred current proposal) or hCaptcha;
- enter provider secret in Supabase Auth Bot and Abuse Protection;
- enable CAPTCHA protection.

Acceptance evidence:
- anonymous sign-in without valid CAPTCHA fails when protection applies;
- sign-in with a valid challenge succeeds.

### A4 — Supabase Realtime public access

Required:
- Realtime setting **Allow public access = OFF**.

Acceptance evidence:
- public/private unauthenticated channel cannot bypass private authorization policy.

### A5 — PostgreSQL SSL enforcement

Required:
- enable Supabase Database SSL Enforcement.

Acceptance evidence:
- non-SSL external connection fails;
- TLS verified external connection succeeds.

## B. Agent-executable after A1

Once Formalife Railway workspace is visible, ChatGPT may:

1. create Railway project `narrative-coop-staging`;
2. create `engine-api`;
3. create `engine-worker`;
4. configure restart/healthcheck/service boundaries;
5. create temporary custom DB login credentials in Supabase;
6. inject runtime credential only into API;
7. inject worker credential only into worker;
8. confirm variable-name separation;
9. connect from Railway to Supabase session pooler;
10. run TLS / DB transaction integration tests;
11. run two-connection concurrency tests;
12. measure region/transaction latency;
13. test password rotation/reconnect;
14. remove temporary credentials/artifacts if spike-only.

## C. Agent-executable after A2–A4

ChatGPT may then:

1. create temporary anonymous test identities through application/test client path;
2. verify JWT claims/signature path;
3. apply provider-specific Realtime SQL on staging only after migration artifact is reviewed;
4. test private channel authorization;
5. test fixed worker Broadcast sender;
6. run Supabase security/performance advisors;
7. record results and postmortem.

## D. Still blocked until this checklist passes

- production-ready executable migration rollout;
- implementation runtime deployment;
- production secrets;
- public playtest environment;
- product UI implementation depending on real auth/session backend.

## E. Security rule

Never paste into chat:
- database passwords;
- Supabase secret/service-role keys;
- Turnstile secret keys;
- Railway tokens.

Configure secrets in provider dashboards or connector variable APIs only.
