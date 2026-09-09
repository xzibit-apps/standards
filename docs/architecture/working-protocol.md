# Xzibit Apps — working protocol (Cowork ↔ Claude Code handoff, and the tiered build standard)

**As of:** 2026-09-09
**Status:** Accepted — effective immediately across the portfolio.
**Source of truth for the day-to-day:** `kb_decisions`, `kb_naming`, `kb_systems` in Supabase. This doc is a narrative layer, same as `portfolio-roadmap.md` and `shared-db-guide.md` — if it diverges from `kb_decisions`, `kb_decisions` wins.

---

## Rule 1 — The repo is the record, not the chat

Cowork sessions do strategy and write decisions into a file in the repo. Claude Code sessions read that file, build, and write results back to it. If a decision only exists in a chat session, it does not exist.

In practice:

- **Cowork's job:** think through a problem, land on a decision, and write it down — in the app's `CLAUDE.md`, in a `docs/` note, or in a `kb_decisions` entry for anything cross-app. Not in a chat transcript that nobody will reread.
- **Claude Code's job:** read that file before starting work, build against it, and write the outcome back into the same file (or a linked one) when done — what shipped, what changed, what's still open.
- **`CLAUDE.md` is the handoff point.** Every repo's `CLAUDE.md` must reflect the current, real state of that app — not the state it was in when the repo was scaffolded. If a Cowork session changes a decision that affects how the app should be built, that change lands in `CLAUDE.md` (or the doc it points to) in the same session, not as a follow-up someone might forget.
- This applies to this doc too: any session that changes the working protocol or the tiered standard updates this file and `xzibit-app-template`'s `CLAUDE.md` in the same action — see the closing note.

---

## Rule 2 — Tiered standard

Not every app needs the same level of rigour. The tiers say what's mandatory everywhere versus what scales with how actively an app is being worked on.

### Tier 1 — mandatory for every app, no exceptions

1. **Database access via service role behind a route guard, never a browser key.** All writes (and any read of sensitive data) go through a server-side route that verifies the caller, then uses the Supabase service-role client. The anon/browser key never gets write access, and never gets read access to anything a client shouldn't see directly.
2. **Schema changes as files in `supabase/migrations/`, nowhere else.** No ad-hoc DDL via the Supabase dashboard or Studio. Every schema change is a migration file, additive and reversible, committed to the owning app's repo (per the ownership rules in `shared-db-guide.md`).
3. **`.env.example` listing every variable, no real values.** If the app reads an env var, it's in `.env.example` with a placeholder — not a live secret, not missing. This is the app's env contract; `assertServerEnv()` (or equivalent) validates against it at boot.
4. **A current `CLAUDE.md`.** Reflects the app's actual identity (repo, deploy target, Supabase project), its actual conventions, and its actual state — not a stale scaffold. See Rule 1.

### Tier 2 — for apps in active development

On top of Tier 1:

- **Shared `@xzibit/ui` and `@xzibit/app-kit`.** Chrome and server/auth wiring are packaged dependencies, not forked/copied into the app. Improvements land as version bumps in those packages, not local patches.
- **Tests on anything that produces a number.** Calculators, capacity/demand engines, financial or scheduling outputs — anything a user reads as a figure needs a test proving it's computed correctly. UI and glue code aren't held to this bar; the numbers are.
- **Documented deploy.** How the app ships (Vercel project, branch, env vars, any manual step) is written down in the repo, not held in one person's memory.

### Tier 3 — framework and host: not mandatory

Framework choice, hosting platform, and similar tech-stack decisions are not standardised across the portfolio. **Team Schedule stays on Rails/Railway.** We are not rewriting working apps for tidiness — a working app on an off-standard stack is not a defect. Tier 1 still applies to it regardless of framework (service-role discipline, migrations-as-files, env manifest, current `CLAUDE.md`); Tier 3 is only about *which* framework/host, which is left alone.

---

## What this means for existing apps

- Every app gets audited against the Tier 1 checklist above. Gaps get fixed opportunistically — this isn't a mandate to stop other work and retrofit everything today, but any touch to an app's backend should leave it Tier-1-compliant if it wasn't already.
- Apps in active development (currently: Capacity Planner, Milestone Calculator — see `portfolio-roadmap.md` status table) are held to Tier 2 as they're worked on.
- No app is forced onto a different framework or host to satisfy this doc. That's what Tier 3 explicitly rules out.

---

## How this doc stays honest

Same discipline as `portfolio-roadmap.md`: when the protocol or the tiers materially change, the change lands here (and in `xzibit-app-template/CLAUDE.md`) in the same action, not as a follow-up. If something here contradicts `kb_decisions`, `kb_decisions` wins — flag it and pause.
