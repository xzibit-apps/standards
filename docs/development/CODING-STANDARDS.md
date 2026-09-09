# Xzibit coding standards

Canonical. Last reviewed 2026-09-09.

Every app CLAUDE.md points here. If this file and a chat session disagree, this
file wins.

---

## 1. The repo is the record

Cowork sessions do strategy and write decisions into a file in the repo. Claude
Code sessions read that file, build, and write results back to it.

**If a decision only exists in a chat session, it does not exist.**

CLAUDE.md is the handoff point. Keep it current in every repo you touch — same
commit as the work, not a follow-up. A CLAUDE.md that describes last month's
architecture is worse than none, because the next session will believe it.

Consequences worth stating plainly:

- Decisions belong in a tracked repo. A decision in an untracked local folder is
  not written down — no other machine and no other session can read it.
- If you change how apps should be built, update this file and
  `xzibit-app-template` **in the same action**. A standard nobody propagated is
  a preference.
- Cross-repo references must be links that resolve. Check them when you move
  files; a 404 in a CLAUDE.md silently disables the instruction for every
  session that follows.

---

## 2. Tiered standard

Three tiers. Know which one applies before you argue about a rule.

### Tier 1 — mandatory, every app, no exceptions

1. **Database access via service role behind a route guard. Never a browser
   key.** The service-role client is server-only, and every route that uses it
   sits behind an auth guard. Do not ship a Supabase anon/publishable key to the
   browser and rely on RLS as the only boundary.
2. **Schema changes as files in `supabase/migrations/`, nowhere else.** Not
   applied by hand in the dashboard, not in an ad-hoc `db/` folder. The file is
   the record; applying it is a separate step.
3. **`.env.example` listing every variable, no real values.** Every var the app
   reads, with a one-line comment on what breaks without it. Placeholders only —
   never a real key, not even a dev one.
4. **A current CLAUDE.md.** See §1.

Tier 1 is the floor. An app below it is a liability regardless of how well it
works today.

### Tier 2 — apps in active development

- Shared `@xzibit/ui` and `@xzibit/app-kit` rather than per-app copies.
- Tests on anything that produces a number. Money, dates, capacity, durations,
  totals, anything a person will act on. Rendering and layout can go untested;
  arithmetic cannot.
- Documented deploy — how it ships, where it ships to, how to verify it landed.

### Tier 3 — framework and host

**Not mandatory.** Team Schedule stays on Rails/Railway.

We are not rewriting working apps for tidiness. Tier 3 choices are revisited
when there is a reason beyond consistency.

---

## 3. Where things live

| What | Where |
|---|---|
| This standard | `xzibit-apps/standards` → `docs/development/CODING-STANDARDS.md` |
| New app starting point | `xzibit-apps/xzibit-app-template` |
| Shared UI | `xzibit-apps/xzibit-ui` |
| App scaffolding / env manifest | `xzibit-apps/xzibit-app-kit` |
| Design tokens + CSS | `xzibit-apps/standards` (`tokens.json`, `xzibit-design.css`) |
| Per-app context and history | that app's `CLAUDE.md` |

---

## 4. Applying this to an existing app

Do not big-bang a repo into compliance. Order of work:

1. Add `.env.example` and fix CLAUDE.md — additive, zero risk, do it now.
2. Audit database access for browser keys — this is the security item.
3. Move schema files to `supabase/migrations/` when you are next touching
   migrations anyway. Moving a large existing folder is a real change with real
   blast radius (tests and tooling reference those paths); it needs its own
   commit and its own verification, not a drive-by.

Record known deviations in the app's CLAUDE.md with a one-line reason. A
documented gap is manageable; an undocumented one is a trap.
