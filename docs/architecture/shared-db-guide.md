# Xzibit Apps — shared Supabase management guide

**Project:** `xzibit-apps` (`rklzgzyqbajhpjvixlkr`), region `ap-southeast-1`, Postgres 17.
**Posture:** One Supabase project, many apps, one schema (`public`), two kinds of tables: **shared** and **app-specific**.
**Source of truth for operational decisions:** `kb_decisions`, `kb_naming`.
**This doc:** how the DB is structured today, how it should be structured, and how to manage it as apps come and go.

---

## The model in one paragraph

Every table in `public.*` belongs to exactly one of two buckets. **Shared tables** represent things the business cares about independent of any one app — projects, staff, venues, public holidays. Shared tables have one **owning app** that is the sole writer; every other app reads only. **App-specific tables** are prefixed by the app's short name (`cp_*`, `ts_*`, `milestone_*`, `load_*`, `kb_*`) and are owned and written by only that app, though they may be read by others via RLS if there's a good reason. Shared tables sit in `public.*` with no prefix. Both kinds live in the same `public` schema — the prefix is the namespace. All cross-table references use real Postgres foreign keys.

That's it. Everything below is elaboration.

---

## Current state — inventory

As of 2026-04-22. Row counts are point-in-time, used to gauge adoption not authority.

### Shared tables (live in `public.*`, no prefix)

| Table | Rows | Owning app | Notes |
|---|---|---|---|
| `apps` | 8 | Launcher | App registry shown on the portal |
| `users` | 1 | Launcher | Doubles as clients — documented oddity, not fixed |
| `roles`, `role_permissions`, `user_app_permissions`, `role_app_permissions` | 2 / 0 / 39 / 16 | Launcher | RBAC |
| `user_sessions` | 161 | Launcher | Cross-app session tracking |
| `projects` | 7 | **TBD post-Whiplash retirement → Launcher Projects admin** | Rich schema (60+ columns), under-populated. ADR to supersede. |
| `project_subtypes` | 13 | Launcher | Reference list |
| `project_notes` | 3 | *unowned — decide* | Notes attached to `projects` |
| `project_dates`, `project_staff` | 0 each | *unowned — dormant* | Planned but unused |
| `venues` | 0 | *unowned — candidate for consolidation* | Shared entity; `ts_venues` is the de facto implementation today |
| `shows`, `contacts` | 0 each | *unowned — dormant* | Planned |
| `quotes`, `quote_line_items` | 0 each | *unowned — dormant* | Planned |
| `labour_types`, `labour_breakdown` | 35 / 0 | *unowned — partially adopted* | |
| `tasks`, `timesheets` | 0 each | *unowned — dormant* | |
| `company_closures` | 87 | **Milestone Calculator** | Per-date, by jurisdiction. Capacity Planner reads from here per Phase 1b. |
| `feedback_submissions`, `feedback_upvotes` | 15 / 0 | Launcher Feedback tab | |
| `app_ideas`, `app_idea_votes` | 2 / 1 | Launcher App Ideas tab | |
| `org_chart_state` | 1 | Launcher Org Chart | |

### App-specific tables (prefixed)

| Prefix | Owning app | Tables | Total rows |
|---|---|---|---|
| `cp_` | Capacity Planner | cp_projects, cp_job_types, cp_curve_libraries, cp_sheet_projects, cp_staff, cp_staff_leave, cp_rows, cp_curves, cp_curve_registry, cp_company_closures | ~5.3K |
| `ts_` | Team Schedule | ts_crew_members, ts_availability_rules, ts_shifts, ts_shift_notes, ts_open_shifts, ts_open_shift_expressions, ts_venues, ts_dev_notes | ~250 |
| `milestone_` | Milestone Calculator | milestone_calculations, milestone_config | 2 |
| `load_`, `truck_`, `tv_` (+ `items`, `inventory_units`, `media`, `movements`) | Truck Load Planner (archived) | load_plans, load_plan_items, truck_settings, tv_models, tv_labels, and the four warehouse-inventory tables | ~7 |
| `kb_` | Launcher Knowledge tab | kb_systems, kb_repositories, kb_deployments, kb_credential_pointers, kb_documents, kb_domains, kb_accounts, kb_open_questions, kb_decisions, kb_naming | ~150 |

### Legacy / artifacts to retire

| Table | Reason |
|---|---|
| `schema_migrations`, `ar_internal_metadata` | Rails leftovers, 0 rows, harmless but misleading. Drop. |

---

## What's wrong today, and how bad

Four patterns create the friction you've felt.

### 1. Duplicate identities across apps
`cp_staff` (71) and `ts_crew_members` (76) are the same people — **52 exact name matches**. Renaming someone means touching both tables. New contractor onboarding is a two-step dance. Fix: one `public.staff`, both apps FK to it.

### 2. Shared-table ownership gaps
`public.projects` is the canonical projects table by design but has 7 rows because its owner (Whiplash) is a POC that's about to retire. Capacity Planner maintains its own `cp_projects` (156 rows) as a result. Same shape issue for `venues` (0) vs `ts_venues` (5). Fix: assign every shared table an active owner; for projects, that's a new Projects admin tab in the Launcher.

### 3. Schema inconsistencies in the same column
`cp_projects.truck_load_date` was TEXT with five different formats. Phase 1b added a parsed DATE column. `public.projects.truck_load_date` is a proper `date` from the start. When they finally consolidate, CP reads the clean column and forgets the mess. This is the reward for the consolidation work.

### 4. Missing foreign keys across namespaces
`cp_projects.job_type_mongo_id` stored a text pointer with no FK, which is how we ended up with every project silently falling through to a flat-curve fallback in the Phase 1 engine. The denormalised `cp_projects.job_type` TEXT from Phase 1b fixes the immediate case. More broadly: any text-as-foreign-key pattern should be replaced with a real FK to the owning table.

---

## The target architecture

### Ownership matrix (desired end state)

| Shared table | Owner (writer) | Readers |
|---|---|---|
| `apps`, `roles`, `users`, `user_app_permissions`, `user_sessions`, `feedback_*`, `app_idea_*`, `kb_*`, `org_chart_state` | Launcher | Launcher only |
| `projects`, `project_subtypes`, `project_notes` | **Launcher — Projects admin** (new) | All apps that touch projects: CP, TS, Milestone, Load (if resurrected) |
| `staff` (new) | **Launcher — People admin** (new) | CP, TS, (Milestone when needed) |
| `venues`, `shows`, `contacts`, `quotes`, `quote_line_items` | **Launcher — Shared entities admin** (new sub-section) | Apps that reference them |
| `company_closures` | Milestone Calculator | CP (Phase 1b done), TS (next) |
| `labour_types`, `labour_breakdown`, `tasks`, `timesheets` | Defer until there's a real consumer | — |

### App-specific tables (prefix convention)

| Prefix | App | Purpose |
|---|---|---|
| `cp_` | Capacity Planner | Curves, registry, staff extensions, project extensions (flat hours) |
| `ts_` | Team Schedule | Shifts, availability rules, shift notes |
| `milestone_` | Milestone Calculator | Calculation snapshots, admin config |
| `load_` | Truck Load Planner | Load plans, load items, truck settings |
| `kb_` | Launcher Knowledge tab | System registry, ADRs, naming, repositories, deployments |

Future apps pick a prefix ≤ 4 characters that hasn't been used. Record in `kb_naming` as `thing_type = 'table prefix'`.

### Relationship rules

- **Every FK uses an actual Postgres `REFERENCES` constraint.** No text-based "soft FKs". If a row can be deleted out from under a child, that's a bug, not a pattern.
- **`ON DELETE` defaults to `RESTRICT`** for shared-table → app-table references. The owning app must resolve dependencies before deleting. Prevents surprise data loss.
- **`ON DELETE CASCADE` only for strictly child tables owned by the same app.** E.g. `cp_staff_leave.staff_id` → `cp_staff.id` cascades fine.
- **App-to-shared FKs are named `<entity>_id`**: `cp_projects.project_id → public.projects.id`, `ts_shifts.staff_id → public.staff.id`.
- **App-to-app FKs should be rare and flagged.** If TS needs to reference a CP row, pause and ask whether the thing really belongs in `public.*`.

### RLS posture (the baseline that already mostly exists)

- RLS enabled on every table in `public.*`.
- Default deny. Policies are additive.
- For shared tables: authenticated users can `SELECT`; only the owning app's service role can `INSERT/UPDATE/DELETE` (service role bypasses RLS by design — the guarantee comes from controlling who has the service-role key).
- For app-specific tables: authenticated users can `SELECT` where `user_app_permissions` grants access; service-role writes from the owning app; no client writes.
- The Auth Correctness Sprint is the vehicle for getting this uniform across every app. After it lands, any new table ships with a baseline policy set in the same migration that creates it.

### Naming convention (formalise the existing pattern)

- **Shared tables:** no prefix. Plural snake_case. `projects`, `staff`, `venues`, `company_closures`.
- **App-specific tables:** `<prefix>_<plural_snake_case>`. `cp_projects`, `ts_shifts`, `milestone_calculations`.
- **Columns:** snake_case. Booleans read as assertions: `is_active`, not `active_flag`. Timestamps end with `_at` (`created_at`, `deleted_at`). Dates end with `_date`. FKs end with `_id`.
- **Indexes:** `idx_<table>_<col1>[_<col2>]`. Unique: `uq_<table>_<col1>`.
- **Enums (USER-DEFINED):** lowercase snake_case values; one enum type per concept, reused across tables.
- Document any deviations in `kb_naming` with a reason.

---

## How to manage the DB — operating rules

### Migrations

- **Every schema change lands as a migration.** No ad-hoc DDL in the Supabase Studio for anything that touches `public.*`.
- **Migrations live in the owning app's repo**, in `supabase/migrations/<timestamp>_<name>.sql`. If a migration touches a shared table, it lives in the Launcher repo (which owns shared tables). Other apps pull the resulting column set by re-running `supabase gen types`.
- **Additive first.** Add columns with defaults or nullable. Backfill in a second migration or a script. Drop old columns in a third migration weeks later, once no app reads them.
- **Applying:** `supabase db push` from the owning repo, or `mcp apply_migration` for controlled one-offs from a human session. Both record in `supabase_migrations.schema_migrations`.
- **Reversibility:** every migration has a commented rollback block at the bottom. Not executed routinely; present for emergencies.
- **Sprint-scale DDL uses a Supabase branch.** For any migration that touches > 1 shared table or > 50% of an app's data, create a branch, apply there, point a preview deploy at it, verify, then promote.

### Ownership and write discipline

- **Each table comments itself.** `COMMENT ON TABLE public.staff IS 'Owned by: Launcher People admin. Writers: Launcher service role only. Do not write from other apps.'` The comment is authoritative.
- **Service-role keys are per-app.** If an app writes to shared tables through the service role, either (a) the app is the owner, or (b) there's a specific, logged exception in `kb_decisions`. Option (b) should be rare and short-lived.
- **Read access is granted via `user_app_permissions`.** An app without permission to read a shared table gets `SELECT` failures — a real signal, not a silent success.

### Audit and observability

- Add `public.core_audit_log` in Phase B of the consolidation sequence: `{ id, app_id, user_id, table, operation, row_pk, diff jsonb, happened_at }`.
- Every service-role write through an owning app emits one row. Supabase has triggers for this if you want it at the DB layer; or the app layer does it in the write path. Pick one, document it, don't split.
- Launcher Knowledge tab grows a "Audit" view over this table.

### Types

- Generate once per week from the Launcher repo: `supabase gen types typescript --project-id rklzgzyqbajhpjvixlkr --schema public > types/db.ts`.
- Commit into a shared types package (`@xzibit/db-types`) — published privately or a git submodule.
- All apps consume that package for shared-table types. App-specific types stay in the owning repo.
- A column rename in `public.*` forces a type error in every consumer repo — which is the feature.

### What to stop doing

- **Stop duplicating shared entities in app-specific tables.** When you see `ts_venues` + `venues` + `projects.venue_name`, that's three sources of truth for one concept. Pick one and FK the rest.
- **Stop using TEXT columns for categorical values that belong to a shared reference.** `cp_projects.status` as free text when `project_status` is an enum → pick the enum.
- **Stop writing to shared tables from non-owning apps.** Even if it works today, it breaks the ownership contract and creates invisible coupling.
- **Stop creating tables without a clear owner.** If the owning app isn't obvious, the table probably shouldn't exist yet.

### What to do when you find something broken

1. Check `kb_decisions` for a relevant ADR.
2. If none exists, draft one — with context, decision, alternatives, consequences. Status = `proposed`.
3. Share in chat or the appropriate channel, discuss, update status to `accepted` or `superseded`.
4. Only then: write the migration.

This is the workflow the team's already using (47 entries deep). Keep doing it.

---

## Consolidation sequence (mirrors the portfolio roadmap)

In order, because each step unblocks the next:

1. **Auth correctness first** — no consolidation touches production until `/api/*` auth holes and JWT-signature gaps are closed. See `kb_decisions` "Auth Correctness Sprint — queued" + Phase 2.
2. **Staff** — `public.staff` + Launcher People admin; FK both `cp_staff` and `ts_crew_members` into it; keep per-app tables for app-specific attributes (utilisation, quality_score, etc.).
3. **Projects** — supersede the "Whiplash owns public.projects" ADR; Launcher gains a Projects admin; migrate `cp_projects` rows into `public.projects` and FK the extension table; TS already reads the right place.
4. **Reference tables** — `venues`, `shows`, `contacts`, `project_subtypes`; one at a time, small migrations.
5. **Retire Whiplash** — superseded ADRs, archived repo, dropped from apps registry.
6. **Observability** — `core_audit_log` + Launcher Audit view.
7. **Types package** — publish once the shared schema is stable for a sprint.

Each step has an exit criterion in the roadmap.

---

## Open questions for the team

- **Should `public.users` and `public.staff` be the same table?** Currently `users` doubles as clients. Splitting gives: `users` = login-capable humans, `clients` = billing entities, `staff` = people on a roster (overlapping with `users` but not 1:1). Worth an ADR before Phase C runs.
- **`public.company_closures` has `type = 'public_holiday'` — does `'company_closure'` need to be written there too**, deprecating `cp_company_closures`? My read is yes (one source of truth for all non-working days), but it depends on whether the Milestone Calculator UI is ready to edit ranged workshop shutdowns. If not: keep `cp_company_closures` for ranges until MC can take them.
- **Where do sales pipeline concepts live** (opportunity, lead, stage)? Not covered in today's inventory. If sales tooling comes into scope, needs its own ADR on whether it's a new app or an extension of the Launcher.
- **TLP's fate** — resurrect with an auth catch-up sprint or drop. The answer changes whether `load_*` tables stay or get archived.

---

## TL;DR

One DB. Two kinds of tables — shared (`public.*`) and app-specific (`<prefix>_*`). One owner per table, documented in a comment. FKs for everything. Migrations in-repo, additive, reversible. Auth correctness before consolidation. Staff first, projects second, the rest opportunistic. `kb_decisions` is the operating record; this doc is a narrative layer on top of it.
