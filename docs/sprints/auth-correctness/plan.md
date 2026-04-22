# Auth Correctness Sprint — plan

**Drafted:** 2026-04-22
**Sprint type:** Portfolio-wide, multi-PR, sequenced.
**Goal:** Every `/api/*` route in every active Xzibit app verifies a signed JWT (or is explicitly listed as public with reason). Every RLS-enabled table has at least one policy matching its legitimate access pattern. No anon-Supabase-client write paths remain.

---

## Phase status at time of audit

| Phase | Status | ADR |
|---|---|---|
| Phase 1 — RLS policies + TLP service_role fix | **COMPLETE** (Migration 034) | `601cc15d` |
| Phase 2 — JWT signature verification (CP + Milestone) | **QUEUED** | `dc1c4563` |
| Phase 3 — Newly identified gaps (this plan) | **PLANNED** | this doc |

> **Stale ADR notice:** ADRs `cebbc5ff` (Auth Correctness Sprint — queued), `3e8a8729` (TLP incognito bounce), and `e5e04f9b` (TLP 24 non-auth routes) remain in `proposed` state even though Phase 1 resolved the specific DB-level issues they describe. These ADRs should be accepted/closed with a back-reference to `601cc15d`. This is a drift signal — flag to Joel before the sprint closes.

---

## Current state per app

### Capacity Planner (`xzibit-apps/capacity-planner`)

- **Middleware auth:** `middleware.ts` exists at repo root. Matcher: `['/((?!api|_next/static|_next/image|favicon.ico).*)']` — **explicitly excludes `/api/*`**. JWT is decoded only (custom base64url function, no library import) — no signature verification. Development mode bypasses all auth entirely (lines 42–51). Consequence: even after adding signature verification to middleware, `/api/*` routes will still not be covered — fixes must land at the route-handler level.
- **API routes — 20 total, 0 authenticated:**
  - `app/api/projects/route.ts` — GET, POST — **no auth** ← HIGH RISK
  - `app/api/staff/route.ts` — GET, POST — **no auth** ← HIGH RISK
  - `app/api/clear-data/route.ts` — POST — **deletes all rows, no auth** ← CRITICAL
  - `app/api/sheets/route.ts` — POST — no auth
  - `app/api/test-data/route.ts` — POST — no auth
  - `app/api/job-types/route.ts` — GET, POST — no auth
  - `app/api/capacity/weekly/route.ts` — GET — no auth
  - `app/api/staff/[id]/leave/route.ts` — GET, POST — no auth
  - Remaining 12 routes: same pattern — zero auth verification
- **Supabase client:** `lib/supabase.ts:8` — module-level singleton using `SUPABASE_SERVICE_ROLE_KEY` (bypasses RLS). Placeholder fallback strings on lines 9–10 if env vars missing.
- **Environment:** `JWT_SECRET` not declared in `.env.example`. No hardcoded JWT fallback found.
- **Fix needed:** Add JWT verification (`jose` `jwtVerify`) to every route handler. Remove or gate `/api/clear-data` (admin-only at minimum). Add `JWT_SECRET` to `.env.example`.
- **Risk if not fixed:** Any unauthenticated caller can read all projects/staff data and trigger a full database wipe via `/api/clear-data`.

---

### Milestone Calculator (`xzibit-apps/milestone-calculator`)

- **Middleware auth:** `src/middleware.ts`. Matcher identical to CP — **explicitly excludes `/api/*`**. Same custom base64url decode, no signature verification. Same dev-mode bypass (lines 42–51). Same consequence: route-handler fixes required.
- **API routes — 4 total, 0 authenticated:**
  - `src/app/api/milestone/calculate/route.ts` — POST — **no auth, accepts arbitrary JSON** ← HIGH RISK
  - `src/app/api/milestone/save/route.ts` — POST — **no auth, persists to DB** ← HIGH RISK
  - `src/app/api/admin/save-config/route.ts` — POST — no auth
  - `src/app/api/user/role/route.ts` — GET — no auth
- **Supabase client:** `src/lib/supabase-server.ts:32–42` — module-level singleton with `server-only` guard, using `SUPABASE_SERVICE_ROLE_KEY`. Bypasses RLS.
- **Environment:** `JWT_SECRET` not in `.env.example`. No hardcoded fallback.
- **Fix needed:** Add JWT verification to all 4 route handlers (same `jose` pattern as planned for CP). Add `JWT_SECRET` to `.env.example`.
- **Risk if not fixed:** Unauthenticated caller can save arbitrary milestone calculations and reconfigure the admin panel.

---

### Team Schedule (`xzibit-apps/team-schedule`)

- **Middleware auth (Rails):** `app/controllers/application_controller.rb` — `before_action :authenticate_user!` applied globally (line 2). Uses `JWT.decode()` with `JWT_SECRET`, `HS256`, `verify_iss: true`, `verify_aud: true` — **full signature verification**. This is the reference implementation the other apps should follow.
- **API routes:** All controllers inherit from `ApplicationController`, so all routes are protected. Write operations additionally gated by `require_manager_or_admin!` / `require_admin!` filters.
- **Supabase client:** Not used — Rails app accesses Postgres directly via `DATABASE_URL`.
- **Environment:** `JWT_SECRET` declared in `.env.example`. No fallback in code (crashes loudly if missing — correct).
- **RLS gap — newly identified:** All `ts_*` tables (9 total: `ts_crew_members`, `ts_shifts`, `ts_venues`, `ts_availability_rules`, `ts_shift_notes`, `ts_open_shifts`, `ts_open_shift_expressions`, `ts_dev_notes`, `ts_skills`) have **`rls_enabled = false`**. Team Schedule's own Rails app uses `DATABASE_URL` directly (bypasses RLS regardless), so this is low-risk for the current app — but if any future client accesses these tables via a Supabase client, they'd have unrestricted access. Flag for Phase 3.
- **Fix needed:** No immediate auth fixes. Enable RLS on `ts_*` tables and add service_role_all + appropriate read policies as a low-risk migration.
- **Risk if not fixed:** Low for current usage; medium if/when Launcher reads team-schedule data via Supabase.

---

### Launcher (`xzibit-apps/launcher`)

- **Middleware auth:** `src/middleware.ts`. Relies on client-side `AuthContext` and role checks — no JWT handling in middleware. `/api/*` routes are in the `publicRoutes` list (explicit early-return). All auth delegated to individual route handlers via `authenticateUser(request)` helper.
- **API routes:** Most routes correctly call `authenticateUser()`. Exception:
  - `src/app/api/auth/login/route.ts` — POST — no auth (correct; this is the login endpoint).
  - `src/app/api/admin/seed-admin-role/route.ts` — POST — **hardcoded secret** `'xzibit-maintenance-2026'` (line 9) instead of `JWT_SECRET` — MEDIUM RISK.
- **Supabase client:** Client-side uses `NEXT_PUBLIC_SUPABASE_ANON_KEY` (`src/lib/supabase/client.ts`). Server-side uses `SUPABASE_SERVICE_ROLE_KEY` (`src/lib/supabase/admin.ts`). Correct split.
- **Environment:** `JWT_SECRET` not declared in `.env.example` (Launcher generates its own JWTs internally; the env var name may differ).
- **Fix needed:** Replace hardcoded maintenance secret in `seed-admin-role/route.ts` with an environment variable. Low priority — endpoint presumably not on a public route but should not contain a secret in source.
- **Risk if not fixed:** Source code leaks a maintenance credential. Anyone with repo access (or who guesses it) can POST to the seed endpoint.

---

### Truck Load Planner (`xzibit-apps/truck-load-planner`) — archived

- **Middleware auth:** `src/middleware.ts`. Matcher excludes `/api/*`. Middleware handles only URL-token-to-cookie redirect. JWT verification deferred to individual API route handlers.
- **API routes:** ~20 routes. Auth-protected routes (`/api/auth/session`, `/api/dashboard/*`) correctly call `authenticateUser()` or `verifyToken()`. `src/lib/jwt.ts` uses `jsonwebtoken` `jwt.verify()` with `JWT_SECRET`, `HS256`, `verify_iss: true`, `verify_aud: true` — **signature verification is implemented**.
- **Supabase client:** `src/lib/supabase/server.ts` — creates fresh service-role client per request. Correct (Phase 1 fix per ADR `601cc15d`).
- **Environment:** `JWT_SECRET` not in `.env.example`. **Hardcoded fallback in `src/lib/jwt.ts:4`:** `process.env.JWT_SECRET || 'your-super-secret-jwt-key-change-in-production'` — if env var is missing in production, the app silently uses the fallback, meaning tokens signed with the real secret will fail verification but tokens signed with the fallback (which is public knowledge from the source) will pass. MEDIUM RISK.
- **Retirement decision (required output of this sprint):** TLP is archived. Three options:
  - (a) **Formally retire** — archive GitHub repo, remove from Launcher app registry, delete Vercel project. Lowest ongoing risk. Preferred if TLP functionality has no near-term roadmap.
  - (b) **Resurrect and fix** — remove hardcoded JWT fallback, add `JWT_SECRET` to `.env.example`, audit remaining unauthenticated routes. Estimated 1–2 days.
  - (c) **Leave as-is** — accepted risk, documented. Not recommended given live Vercel deploy.
- **Fix needed:** Decision first. If resurrecting: remove hardcoded fallback, add `JWT_SECRET` to `.env.example`. If retiring: GitHub archive + Launcher registry update + Vercel teardown.
- **Risk if not fixed:** Live Vercel deploy with a public-knowledge fallback JWT secret — any caller who knows the fallback can forge tokens if `JWT_SECRET` is unset in the Vercel env.

---

## Shared-DB state

### Tables with RLS enabled but zero policies (needs confirmation)
The following tables have `rls_enabled = true` but were not covered by Migration 034. A follow-up policy audit query should confirm each. Based on the RLS table list:
- `company_closures`, `contacts`, `cp_*` tables (10), `feedback_submissions`, `feedback_upvotes`, `milestone_calculations`, `milestone_config`, `app_ideas`, `app_idea_votes`, `org_chart_state`, `project_dates`, `project_notes`, `project_staff`, `project_subtypes`, `projects` (public), `quotes`, `quote_line_items`, `role_app_permissions`, `shows`, `tasks`, `timesheets`, `venues`, `labour_breakdown`, `labour_types`

> **Action required:** Run `select tablename, count(*) from pg_policies where schemaname = 'public' group by tablename` and cross-reference with the full RLS-enabled table list to identify any tables with RLS on but no policies. This is a separate Sprint item.

### Tables with RLS explicitly disabled
- `ts_*` tables (9): `ts_crew_members`, `ts_shifts`, `ts_venues`, `ts_availability_rules`, `ts_shift_notes`, `ts_open_shifts`, `ts_open_shift_expressions`, `ts_dev_notes`, `ts_skills` — RLS off. Low risk for current Team Schedule Rails access patterns; needs enabling before any Launcher/Supabase-client reads.
- `ar_internal_metadata`, `schema_migrations` — system tables; intentionally no RLS.

### Tables with confirmed policies (from audit query)
All 6 tables queried have `anon_deny` + `service_role_all` policies (2 each): `apps`, `role_permissions`, `roles`, `user_app_permissions`, `user_sessions`, `users`.

### Policy approach note — ADR drift flag
ADR `3e8a8729` explicitly said "Do NOT fix by swapping to `createServiceClient()` — that bypasses RLS rather than configuring it." Migration 034 (ADR `601cc15d`) resolved the incognito bounce by adding `*_anon_deny` policies (explicit block) and routing everything through `service_role`. The original ADR's preferred approach (anon-friendly RLS policies) was not implemented. The outcome is correct for the current use case (all access is service_role), but the decision rationale in `3e8a8729` is now stale. Flag to Joel: should `3e8a8729` be closed as "resolved differently"?

---

## Proposed fix sequence

Ordered by least-risky-first, dependency order, and live-user impact.

| # | Fix | Repo | Scope | Risk | Rough effort |
|---|---|---|---|---|---|
| 1 | Remove hardcoded JWT secret fallback in `src/lib/jwt.ts:4` | `truck-load-planner` | 1-line code change + `.env.example` update | **Low** — TLP is archived; no live user impact | 0.25 day |
| 2 | Enable RLS on all `ts_*` tables + add `service_role_all` policies | `launcher` (migration) | SQL migration only — additive, reversible | **Low** — Team Schedule uses `DATABASE_URL` directly, unaffected | 0.5 day |
| 3 | Add JWT verification (`jose` `jwtVerify`) to all CP route handlers; remove or gate `/api/clear-data` | `capacity-planner` | ~20 route handlers + 1 dangerous endpoint | **Medium** — can lock out live users if `JWT_SECRET` mismatch; needs coordinated deploy with env-var check | 1–2 days |
| 4 | Add JWT verification to all Milestone Calculator route handlers | `milestone-calculator` | 4 route handlers | **Medium** — same risk as CP | 0.5–1 day |
| 5 | Update CP + Milestone middleware to include `/api/*` in matcher | `capacity-planner`, `milestone-calculator` | Matcher config change | **Low** — defence-in-depth only; route handlers are the primary fix | 0.25 day each |
| 6 | Replace hardcoded maintenance secret in Launcher `seed-admin-role/route.ts` | `launcher` | 1-line change + new env var | **Low** | 0.25 day |
| 7 | TLP retirement decision + execution | `truck-load-planner` | Depends on decision: GitHub archive + Launcher registry + Vercel teardown (retire) OR full auth fix (resurrect) | **Low if retire** / **High if resurrect** | 0.5 day (retire) or 2–3 days (resurrect) |
| 8 | Close stale ADRs: `cebbc5ff`, `3e8a8729`, `e5e04f9b` with back-reference to `601cc15d` | — | `kb_decisions` updates only | **None** | 0.5 day |
| 9 | Full RLS policy audit — confirm no RLS-enabled table has zero policies | `launcher` (migration) | SQL audit + policy fills | **Low per migration** | 1 day |

---

## Acceptance criteria (sprint exit)

- [ ] Every deployed `/api/*` route in each live app either verifies a signed JWT or is listed in `docs/sprints/auth-correctness/public-endpoints.md` with a reason.
- [ ] No Supabase anon-client writes remain in any live app.
- [ ] Every `public.*` table with RLS enabled has at least one policy, OR is in an explicit "intentionally locked" list with a reason.
- [ ] All `ts_*` tables have RLS enabled with appropriate policies.
- [ ] TLP is either resurrected with clean auth or formally retired (Launcher registry + GitHub archive + Vercel teardown).
- [ ] No hardcoded secrets remain in any source file (JWT fallback in TLP, maintenance secret in Launcher).
- [ ] Stale ADRs `cebbc5ff`, `3e8a8729`, `e5e04f9b` are closed/resolved.
- [ ] A new ADR `Auth Correctness Sprint — closed` marks the outcome in `kb_decisions`.

---

## Out of scope for this sprint

- JWT secret rotation (separate concern; see ADR `e89707ca` — `JWT signing secret NOT rotated in Sprint C`).
- `jwt.ts` duplication between Launcher and TLP (see ADR `b71cf76e` — `truck-load-planner/src/lib/jwt.ts duplicates launcher's jwt.ts`).
- Phase B/C/D consolidation work (blocked by this sprint landing).
- Sprint C Phase C cleanup — dropping legacy `trucker-*` iss/aud values (see ADR `4362d6c9`).

---

## Top findings summary

1. **CP `/api/clear-data` is unauthenticated and deletes all rows** — the most severe finding in the audit. Not previously documented in any ADR. Must be gated or removed as part of fix #3.

2. **CP and Milestone middleware explicitly exclude `/api/*`** — this means the planned "add signature verification to middleware" fix (ADR `931fe6d7`, `dc1c4563`) is insufficient on its own. Fixes must land at the route-handler level in both apps. The middleware matcher must also be updated as a separate defence-in-depth step (fix #5).

3. **All `ts_*` tables have RLS disabled** — not previously documented. Low risk today (Team Schedule uses direct DB), but a latent risk as Launcher expands to read staff/schedule data via Supabase client. Should be addressed before Phase C consolidation work begins.

---

## Links

- Shared-DB guide: https://raw.githubusercontent.com/xzibit-apps/standards/main/docs/architecture/shared-db-guide.md
- Portfolio roadmap: https://raw.githubusercontent.com/xzibit-apps/standards/main/docs/architecture/portfolio-roadmap.md
- Relevant `kb_decisions` ADRs:
  - `cebbc5ff` — Auth Correctness Sprint — queued (STALE — see `601cc15d`)
  - `601cc15d` — Auth Correctness Sprint Phase 1 complete — RLS policies + TLP service_role fix
  - `dc1c4563` — Phase 2 queued — signed-JWT verification gap
  - `931fe6d7` — CP + Milestone do not verify JWT signatures
  - `3e8a8729` — TLP incognito bounce (STALE — see `601cc15d`)
  - `e5e04f9b` — TLP 24 non-auth routes (STALE — see `601cc15d`)
  - `09cebb17` — Sprint C — JWT iss/aud migrated from trucker-* to xzibit-*
  - `b71cf76e` — jwt.ts duplication (out of scope)
  - `e89707ca` — JWT signing secret NOT rotated in Sprint C (out of scope)
