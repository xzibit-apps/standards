# Team Schedule — pre-flight DB audit

**Drafted:** 2026-04-22
**Verdict:** Ready with caveats — two governance findings documented below, neither blocks go-live. An ADR is required to formalise Team Schedule's write access to `public.projects` before Phase C consolidation.

---

## Findings

### 1. DATABASE_URL role

- **Role in use:** `postgres.rklzgzyqbajhpjvixlkr` (Supabase session-mode pooler format: `<role>.<project_ref>`)
- **Actual Postgres role:** `postgres`
- **BYPASSRLS status:** `rolbypassrls = TRUE` — confirmed via `pg_roles` query
- **SSL enforcement:** Yes — `sslmode=require` in connection string
- **Config files:** `config/database.yml` reads `ENV["DATABASE_URL"]` in all environments. `.env` (dev) contains the full pooler URL. `.env.example` shows the Supabase pooler format as a placeholder.
- **Verdict: Safe.** The `postgres` role bypasses RLS at the Postgres level. Auth Correctness Sprint Fix #2 (Migration 040 — RLS enabled on `ts_*` tables, ADR `6a3e1a4b`) does not affect Team Schedule's access path. No changes required.

---

### 2. Cross-app reads from `public.*`

Team Schedule reads from 7 shared tables. All except `public.projects` are explicitly marked `readonly?` in their Rails model:

| Table | Model | Readonly? | Files |
|---|---|---|---|
| `public.projects` | `Project` | **No** — writable (see Finding 3) | `app/models/project.rb`, `app/controllers/projects_controller.rb`, `app/controllers/schedule_controller.rb` |
| `public.users` | `User` | Yes — `readonly?` line 15–16 | `app/models/user.rb` |
| `public.roles` | `Role` | Yes — `readonly?` lines 8–10 | `app/models/role.rb` |
| `public.venues` | `Venue` | Yes — `readonly?` lines 9–10 | `app/models/venue.rb`, `app/models/project.rb:22` (legacy fallback) |
| `public.shows` | `Show` | Yes — `readonly?` | `app/models/show.rb`, `app/models/project.rb:21` |
| `public.project_staff` | `ProjectStaff` | Yes — `readonly?` | `app/models/project_staff.rb`, `app/models/project.rb:33` |
| `public.labour_types` | `LabourType` | Yes — `readonly?` | `app/models/labour_type.rb` |

`public.contacts` and `public.company_closures` — no references found.

---

### 3. Cross-app writes to `public.*`

**Governance finding — not a runtime P0, but requires an ADR before Phase C.**

Team Schedule writes to `public.projects` in two ways:

**A. Active model writes (application code):**
- `app/controllers/projects_controller.rb:66` — `@project.save` (create path)
- `app/controllers/projects_controller.rb:79` — `@project.update(...)` (update path)
- `app/models/project.rb:218` — `update_column(:shifts_stale_since, Time.current)`
- `app/models/project.rb:161` — `update_column(:shifts_stale_since, nil)` (in `regenerate_shifts`)

**B. Historical migrations that added columns to `public.projects`:**

| Migration | Columns added |
|---|---|
| `015_add_project_fields.sql` | `location`, `venue_name`, `bump_in_date`, `bump_out_date`, `show_open_date`, `show_close_date` |
| `019_add_project_display_fields.sql` | `venue_acronym`, `short_name` |
| `021_create_ts_venues_and_project_fk.sql` | `ts_venue_id UUID` + FK → `ts_venues(id)` |
| `023_add_project_datetimes_and_notes.sql` | `bump_in_start_at`, `show_open_at`, `show_close_at`, `bump_out_start_at`, `bump_out_end_at`, `bump_in_notes`, `bump_out_notes` |
| `026_add_shifts_stale_since.sql` | `shifts_stale_since TIMESTAMPTZ` |
| `028_add_project_access_times.sql` | `access_bump_in_start_at`, `access_show_open_at`, `access_show_close_at`, `access_bump_out_start_at`, `access_bump_out_end_at` |
| `029_add_dates_confirmed_with_manual.sql` | `dates_confirmed_with_manual BOOLEAN` |

**Context:** ADR `6ad3b086` designates Launcher as the owner of `public.projects`. However, Team Schedule has been extending `public.projects` with scheduling-domain columns since before that ADR was formalised. All added columns are clearly TS-scoped (`bump_in_*`, `show_*`, `shifts_stale_*`, `access_*`) with no naming conflicts. There are no deletes against `public.projects`.

**Recommendation:** Accept this as the established pattern via a new ADR that formalises the ownership split: Launcher owns the record lifecycle (create/delete/shared fields); Team Schedule may write its own scheduling columns (enumerated list). This ADR should be in place before Phase C, when the shared-DB guide's "one writer per table" principle will be actively enforced.

---

### 4. Empty-state handling for `public.projects`

**Pass.** `app/views/projects/index.html.erb`:
- Line 52: `<% if @projects.any? %>` — guards the project card loop
- Lines 61–65: else clause renders a "No projects found" blank-state card with a create link
- Line 69: footer handles singular/plural project count gracefully

Controller (`app/controllers/projects_controller.rb:6`): `@projects = Project.parent_projects` returns an ActiveRecord collection — empty collection if no rows, never nil. No `.first.id` calls or other assumptions that a project always exists were found.

---

### 5. Dead references to `public.venues`

**No dead references.** The `Venue` model (`app/models/venue.rb`) is intentionally alive as a legacy compatibility layer:
- Marked `readonly?` (lines 9–10) — no writes possible
- `app/models/project.rb:22`: `belongs_to :venue, optional: true` with inline comment "legacy shared `venues` table — read-only"
- `app/models/show.rb:7`: `belongs_to :venue, optional: true`
- `app/controllers/schedule_controller.rb`: eager-loads both `:venue` (legacy) and `:ts_venue` (current) on Project queries

New venue data flows through `ts_venues` (the `TsVenue` model). The `display_city` method on Project reads `ts_venue` first and falls back to the legacy `location` string field — `public.venues` is no longer the primary lookup. The legacy association can be removed once all historical projects have been migrated, but it is not dead code today.

---

### 6. Migrations posture

**Governance finding — same as Finding 3.**

All TS migrations correctly create/alter only `ts_*`-prefixed tables, **with the exception of the 6 migrations** that added scheduling-domain columns to `public.projects` (detailed in Finding 3). No TS migration touches `public.users`, `public.roles`, `public.venues`, `public.shows`, or any other shared table DDL.

The `public.projects` additions are all additive (no column drops, no type changes) and TS-namespaced in everything but the table name.

---

## Go-live checklist

- [x] DATABASE_URL role (`postgres`) bypasses RLS — confirmed via `pg_roles`
- [x] Auth Correctness Sprint Fix #2 (RLS on ts_* tables) does not break TS
- [ ] **ADR required:** Formalise Team Schedule's write access to `public.projects` and enumerate owned columns before Phase C
- [x] No cross-app writes to any table other than `public.projects`
- [x] Empty-state renders cleanly (7 projects or 0)
- [x] No dead references — `public.venues` use is intentional legacy compatibility
- [x] All `public.*` migration changes are additive and TS-namespaced

---

## Recommended actions before production

1. **Open ADR:** "Team Schedule owns scheduling columns on `public.projects`" — enumerate the 23 columns added by migrations 015, 019, 021, 023, 026, 028, 029 as TS-owned. Launcher owns all other columns and record lifecycle (INSERT of new projects, DELETE). This gates Phase C safely.

2. **Optional (non-blocking):** Add a comment or concern to the `Project` model documenting the column ownership split, so future engineers know not to add more columns without the ADR process.

3. **Optional (post-go-live):** Once all historical projects have `ts_venue_id` populated, remove the legacy `belongs_to :venue` association and the eager-loading of `:venue` in schedule_controller.

---

## Related ADRs

| ADR | Title | Relevance |
|---|---|---|
| `6ad3b086` | Launcher owns `public.projects` | TS writes to this table — needs governance clarification |
| `6a3e1a4b` | Auth Correctness Sprint Fix #2 — RLS on ts_* | Confirmed safe for TS's postgres role |
| `1b776834` | Auth Correctness Sprint — plan accepted | Sprint context |
