# Xzibit Apps — portfolio roadmap

**As of:** 2026-04-22
**Posture:** Standalone apps sharing a single Supabase project. No ERP monolith. No Whiplash-as-hub (Whiplash was a POC and will be retired).
**Source of truth for the day-to-day:** `kb_decisions`, `kb_naming`, `kb_systems` in Supabase.
**This doc:** the 6-month arc — sequencing, ownership, and why. It should be re-read every 4-6 weeks and superseded by a `kb_decisions` entry when the plan changes.

---

## Guiding principles

1. **Standalone apps share data, they don't share code.** Every app owns its slice of the DB. Nothing magic — just one Postgres with disciplined naming.
2. **Every shared entity has one owning app.** The owning app is the only writer. Everyone else reads. Shared entities live in `public.*`; app-specific tables use a `<app>_` prefix.
3. **Auth correctness before consolidation.** No schema refactors touch production while any app in the portfolio has unsigned-JWT or unauth'd `/api/*` holes.
4. **Additive migrations, reversible.** No destructive changes. Every migration goes through `supabase migrations` (or the MCP `apply_migration`) and lives in-repo.
5. **Decisions land in `kb_decisions`.** Architecture moves through ADRs, not Slack. Naming renames live in `kb_naming`. Every sprint closes its own kb_decisions entry.
6. **Xzibit App Standard v1.0 is the UI contract.** Every app loads `xzibit-standards.vercel.app/xzibit-design.css` and uses its component classes. No bespoke styling.

These aren't new — they're extracted from the decisions and sprints that have already landed. This doc just makes them explicit for the next 6 months.

---

## Current state — apps, status, and what's next for each

Ordered roughly by strategic weight, not alphabetical.

| App | Status | Deploy | Data ownership | Near-term next step |
|---|---|---|---|---|
| **Launcher** (xzibit-apps) | live | Vercel | `apps`, `users`, `roles`, `user_app_permissions`, `feedback_*`, `app_idea_*` | Extend with **Projects admin** + **People admin** tabs so public.projects and public.staff have an owner (see consolidation plan) |
| **Capacity Planner** | live | Vercel | `cp_*` | Phase 1b bug fix, Phase 1c (curves review + closures admin), then Phase 2 (AV/dept/scenarios) |
| **Team Schedule** | live | Railway | `ts_*` | Consolidate crew_members → core staff; already reads `public.projects.*_date` — stays ahead of the pack |
| **Milestone Calculator** | dev | Vercel | `milestone_*`, **owns** `public.company_closures` | Retrofit to Xzibit Standard v1.0 (done), promote to `live`, wire JWT signature verification |
| **Truck Load Planner** | archived | Vercel | `load_*`, `truck_*` (+ `items`, `inventory_units`, `media`, `movements` all empty) | Decide: resurrect with auth catch-up sprint, or formally retire + drop tables |
| **Whiplash** | dev (POC, to retire) | Manus → Vercel | Nominal owner of `public.projects` in existing ADRs | Retire. Reassign `public.projects` ownership to Launcher's Projects tab. Supersede related kb_decisions. |
| **ERP Overview** | live | Vercel | none (read-only view) | No backend work; visual reference |
| **Brand Guidelines** | live | Vercel | none | Xzibit Standard docs host |
| **LED Screen Calculator** | live | Vercel | none | Calculator-only; retrofit pending |
| **TV Bracket Labels** | dev | Vercel | `tv_models`, `tv_labels` | Keep as-is; small tool |
| **X-Mark** | live | Vercel | none | Keep as-is |
| **Studio Website** | live | Vercel | none | Marketing site |
| **Feedback & Ideas** | live | Launcher-internal routes | `feedback_*`, `app_idea_*` | Keep; occasionally review the backlog |
| **Serpent Cup** | archived | — | — | Dropped |

"Owns" = the writer. Everyone else reads via the Postgres instance (filtered by RLS).

---

## Sequencing — the next 6 months, in phases

Each phase has an exit criterion. Don't start the next phase until the previous phase's exit is met.

### Phase A — Auth correctness sprint (2-3 weeks)
**Why first:** No data-model refactor touches production while any app can be trivially scraped. Already queued in `kb_decisions` ("Auth Correctness Sprint — queued", "Phase 2 of Auth Correctness Sprint queued — signed-JWT verification gap").

**Scope:**
- Close `/api/*` auth holes in Capacity Planner and Milestone Calculator (JWT signature verification with `jose` or `jsonwebtoken`, matching the Team Schedule pattern).
- Fix the 24 non-auth routes in Truck Load Planner that still use the anon Supabase client — or, if TLP is retiring, shut them off.
- RLS policies on `user_sessions` + `users` covering legitimate anon-role access patterns (per the TLP incognito-bounce ADR).
- Service-role usage audit: every write path goes through `createServiceClient`, not the anon client.

**Exit:** Every deployed `/api/*` endpoint either requires a valid JWT or is explicitly listed in a `public-endpoints.md` register with a reason. No anon-client write paths remain.

### Phase B — Capacity Planner closeout (2-3 weeks, in parallel with A's tail)
**Why now:** The operational goal ("Sam can plan next quarter") is within reach. Don't stall it waiting on portfolio-level work.

**Scope:**
- Phase 1b bug fix: `ambiguous_probability` detection (active).
- Phase 1c: curves review + closures admin UIs (prompt drafted).
- Phase 1d: data cleanup working session with Sam — fill the 78 zero-week projects, the 38 missing truck dates, the 11 ambiguous probabilities.

**Exit:** Capacity Planner's demand chart shows real curves + real capacity for 90%+ of projects. Warnings pill row counts under 20 rows across all tags.

### Phase C — Staff unification (3-4 weeks)
**Why this first among consolidations:** 52 of 71 cp_staff match ts_crew_members by name — the overlap is already there, the pain of maintaining both is already felt, and the migration path is short.

**Scope:**
- Add `public.staff` table (new, not an extension of `users` — they're different things; `users` has company_name oddity per ADR).
- Add `crew_type` discriminator (workshop, onsite, contractor, etc.).
- Backfill from union of cp_staff + ts_crew_members, matched by name; Joel resolves name ambiguities (~4 needed).
- Launcher gains a **People** admin tab — the only writer to `public.staff`.
- Add FKs: `cp_staff.staff_id` and `ts_crew_members.staff_id` → `public.staff.id`.
- Migrate CP and TS to read display name + crew_type from `public.staff`, keep their own tables for module-specific fields (utilisation, skills, quality_score, etc.).
- `ts_crew_members` and `cp_staff` become module-specific extensions, not independent identities.

**Exit:** One canonical person per human. Renaming a staff member in the People admin propagates instantly to both CP and TS dashboards without a data migration.

### Phase D — Projects spine (4-6 weeks)
**Why after staff:** Projects are the biggest shared entity and the messiest. Staff consolidation proves the pattern first.

**Scope:**
- Supersede the "Whiplash owns `public.projects`" ADR with a new one: "Launcher's Projects admin owns `public.projects`".
- Extend `public.projects` schema if needed (it's already well-populated with columns but only 7 rows — most of the work is data, not schema).
- Launcher gains a **Projects** admin tab: full CRUD on `public.projects`.
- Migrate cp_projects rows into `public.projects` by mapping `job_number` ↔ `project_number`. Flag collisions.
- `cp_projects.public_project_id UUID REFERENCES public.projects(id)` — CP keeps cp_projects as a module-specific extension holding the flat-hour columns (cnc, build, paint, etc.), FK-linked.
- Team Schedule already reads `public.projects`; no change needed there.
- Google Sheets sync now writes to `public.projects` via the Projects admin API, not to cp_projects directly.
- Retire Whiplash.

**Exit:** Every project has exactly one authoritative row in `public.projects`. CP and TS render the same project name, show dates, and venue for a given job number without reconciling.

### Phase E — Capacity Planner Phase 2 (4-6 weeks)
**Why deferred:** AV sub-buckets, department views, and scenario planning all depend on a stable projects spine. Building them on top of cp_projects would mean rebuilding them when projects consolidate.

**Scope:** Port the Whiplash-reference engines — avDemandEngine, avCapacityEngine, departmentDemandEngine, departmentCapacityEngine, scenarioProcedures — onto the consolidated data model.

**Exit:** Sam can produce a next-quarter capacity scenario with AV sub-buckets and department-level views.

### Phase F — Reference table hygiene (2-3 weeks, can interleave)
Smaller consolidations that can happen alongside C/D/E when there's spare capacity:

- `venues` (0 rows) vs `ts_venues` (5 rows) vs the `projects.venue_name` TEXT column — pick one, backfill, FK the rest.
- `project_subtypes` (13 rows) — confirm adoption; add FKs from `cp_projects` and `public.projects`.
- `cp_job_types` (12 rows) vs the job types implied by `cp_curve_registry` — reconcile (already half-done in Phase 1b).
- `labour_types` (35 rows) — confirm adoption; wire into TS and CP where relevant.
- Drop Rails artifacts (`schema_migrations` with zero rows, `ar_internal_metadata`).

**Exit:** No two tables represent the same entity. `kb_naming` has no outstanding "canonical name" column mismatches.

### Phase G — Shared TypeScript types package (1-2 weeks)
Once schemas stabilise:

- Generate types from Supabase with `supabase gen types typescript --schema public`.
- Publish as a private npm package (`@xzibit/db-types`) or as a git-submodule — whichever is cheapest given team size.
- All apps consume the same types for shared tables. App-specific types stay local.

**Exit:** A rename of a column in `public.staff` forces a type error in every app that reads it until they update.

### Cross-cutting work (runs through all phases)

- **Xzibit App Standard v1.0 adoption** — every live app loads xzibit-design.css and passes the 10-point checklist. Whiplash retires, LED Screen Calculator retrofits. Standard v1.1 dark-theme variant lands as a separate ADR.
- **Observability** — add a `public.core_audit_log` table. Every write endpoint logs `{ app_id, user_id, table, operation, row_pk, diff, at }`. Dashboard tab in Launcher for visibility.
- **Supabase branching** — treat the shared project as production. For any migration that touches > 1 app's tables, use a Supabase branch, test, then promote.
- **CLAUDE.md drift check** — every app repo keeps its CLAUDE.md in sync with the shared standards; build a CI step that fails if xzibit-design.css link is missing.

---

## What "success" looks like in 6 months

- Sam plans next quarter on Capacity Planner with confidence. Warnings pill row is mostly empty.
- A new staff member is added once (in Launcher's People tab) and appears in TS and CP within seconds.
- A new project is added once (in Launcher's Projects tab) and appears in CP demand, TS shifts, and Milestone calcs without duplication.
- Every `/api/*` route is either authed or publicly listed with a reason.
- Whiplash is dead and nobody misses it.
- `kb_decisions` has ~20 more entries and `kb_naming` has zero pending renames.

## What "success" does NOT require in 6 months

- A microservices split. One DB, one schema per app, is fine.
- A custom ORM or a shared backend. Each app keeps its own Next.js / Rails / whatever.
- A full rewrite of any app. Consolidation is additive and backwards-compatible at every step.
- Any downtime. Every migration is reversible and deployable without maintenance windows.

---

## Open questions to resolve in the next month

- Does **staff** also cover sales-only people (for reporting), or just people who get scheduled/planned? Decides whether `public.staff.crew_type` includes `sales` as a value.
- Should `public.users` and `public.staff` merge? Currently `users` doubles as clients (documented oddity). Would be cleaner to have `users` = real humans with logins, `clients` = companies being billed, `staff` = real humans on payroll (some of whom are also `users`). Deserves its own ADR.
- Projects admin UI — minimal table + form, or something richer like a kanban? If richer, does it merit being a standalone app instead of a Launcher tab?

---

## How this doc stays honest

This doc is not a `kb_decisions` entry; it's a narrative layer on top of them. When any phase's scope or sequencing materially changes, the change lands in `kb_decisions` first (as an ADR) and this doc gets updated to reference it. If they diverge, `kb_decisions` wins.
