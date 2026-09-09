# FixFlow V2 — Agent Context

Compact entry point for AI sessions. Read this first, then load only the doc you need.

## What this is

Multi-tenant ticket / issue-reporting system for small companies (≤ ~30 people).
Primary use case: an employee reports a problem in one place instead of scattering it
across Messenger, phone and email. Staff pick it up, work it, close it.

Status: **planning. No code exists yet.** Fresh repository — V1 is not ported.

## Stack

Laravel 13 · PHP 8.5 · Livewire 4 · Filament 5 · Blade · Tailwind · PostgreSQL · Pest 5

Re-verified 2026-09-08 against packagist and php.net, replacing the August figures.

**PHP 8.5**, because 8.4's active support ends 2026-12-31 while 8.5's runs to 2027-12-31.
That reason stands alone. **Not PHP 8.3**, but not for the reason previously recorded here:
Laravel 13 declares `php ^8.3` and every `symfony/*` constraint in it reads `^7.4 || ^8.0`,
so 8.3 does not fail — it silently resolves the older Symfony 7.4 line, because the 8.x line
Composer prefers (`symfony/console` 8.1) requires `>= 8.4.1`. Nobody ran the 8.3-pinned
resolution, so this is a claim about the *preferred* resolution, not an absolute floor.

**Pest 5**, not 4. 5.1.4 is current, `pest-plugin-laravel` 5.x targets Laravel 13, and the
jump costs nothing on a repository that contains no tests yet — taken now precisely because
later it stops being free.

Re-verify at install (`06-build-plan.md` M1); never take a version number from an agent's
memory, including mine.

## Hard rules (do not violate without an explicit decision)

1. **Business logic lives in Actions**, never in Filament resources, Livewire components
   or controllers. Those are entry points only.
2. **Two doors, one room.** UI and REST API both call the same Actions. The UI never
   calls the API.
3. **Everything tenant-scoped.** No query may cross tenant boundaries except in the
   super-admin panel.
4. **No user-facing string is hardcoded.** Translation keys only. Code and docs are
   English; Polish exists solely as a locale.
5. **The razor.** If a feature can be cut and the system still works, cut it. This does
   *not* apply to the four foundations (tenancy, i18n, action layer, permissions).
6. **Filament must stay evictable.** Deleting Filament should cost views, never domain.

## How to work — one segment at a time

The plan is segmented deliberately. **Building ahead of the current step is the failure
this section exists to prevent**, and it outranks any urge to be helpful. A step delivered
exactly is worth more than three steps delivered approximately.

1. **Agree the step before writing code.** State what it includes, what it explicitly
   excludes, and what will prove it done. Wait for a yes.
2. **One step, then stop.** When the step meets its check, report and stop. Do not begin the
   next one. Do not "while I'm here" into it. The next step starts when the user says so.
3. **Scope is a contract, not a starting point.** Code outside the current step is wrong
   even when it is good code.
4. **Found work goes in a GitHub issue, not into the diff.** One issue, naming the
   milestone it belongs to, then carry on with the step.
5. **No speculative structure.** No stubs, no placeholder files, no abstraction for a step
   that has not started, no config for a feature that is out of MVP. This is the razor
   (`01-principles.md` §1) applied to execution.
6. **A doc decision is not yours to change quietly.** If a doc is wrong, or the step is
   impossible as written, stop and say so. Never improve the design mid-step.
7. **A step that turns out too big gets split, out loud, before more code** — not silently
   absorbed.

Steps are defined in `docs/06-build-plan.md`. If the current step is unclear, ask; do not
pick one.

## How to report — certainty, walkthrough, proof

The rules above govern *what* gets built. These govern what you are told about it, and when
the work stops.

1. **Ambiguity with a consequence stops the work.** Reversible details — a variable name, a
   test's wording, where a helper sits — get decided and listed in the recap. Anything
   touching schema, domain rules, dependencies, a public contract, or a decision a doc
   already owns: stop and ask. Guessing at one of those costs more than the round trip.
2. **Blocked on one thing is not blocked on everything.** Finish every part of the step the
   answer does not affect, then stop and name exactly what is waiting. Nothing downstream
   of an open question gets written on a guess.
3. **Plan, then recap.** Before code: what the step includes, what it excludes, what will
   prove it done, and a plain explanation of any concept or service it introduces — a
   global scope, a queue worker, Sanctum. A system nobody can explain is a system nobody
   can maintain. After code: what changed and what to look at. No commentary in between.
4. **Never take an API from an agent's memory either.** `Stack` above says it about version
   numbers; it is equally true of method signatures. Laravel 13, Filament 5, Livewire 4 and
   Pest 5 are partly newer than any model's training. Read `vendor/` or the official docs
   before using something uncertain, and never invent a signature because it looks
   plausible.
5. **"Done" means the check was run.** Not reasoned about, not expected to pass — run, with
   the real output shown. A failing check is reported as failing, a skipped one as skipped.
   `07-conventions.md` defines done mechanically precisely so this is never a judgement call.
6. **Irreversible things ask first.** Adding a dependency, deleting or overwriting an
   existing file, `migrate:fresh` or anything that drops data, any destructive git
   operation. Approval for one is not approval for the next.
7. **Docs get the decisions that were actually made.** When something a doc owns is agreed
   in conversation, it is written into that doc as part of the step and the diff appears in
   the recap. Changing a decision unilaterally is a different act, and `How to work` rule 6
   forbids it.

## Doc index — load on demand

| File | Read when |
|---|---|
| `docs/00-overview.md` | You need context: who this is for, why it exists |
| `docs/01-principles.md` | You're making a design trade-off |
| `docs/02-domain.md` | You're touching models, permissions, tickets, statuses |
| `docs/03-architecture.md` | You're touching layers, tenancy, API, i18n, deploy |
| `docs/04-scope.md` | You're deciding whether something belongs in the MVP |
| `docs/05-open-questions.md` | Something seems undecided — check here before assuming |
| `docs/06-build-plan.md` | You're asking what to build next, or whether a step is done |
| `docs/07-conventions.md` | You're about to write code — naming, layout, keys, tests, done |
| `docs/08-environment.md` | You're touching local setup, CI, deploy or rollback |
| `docs/private/*` | Anything about money, legal exposure, or who the pilot is |

`docs/private/` is **gitignored** — readable locally, never pushed — so the repository went
public at M1.1 without a rewrite. Never move its contents into a tracked file, and never restate
them in one; a public doc may point at it, nothing more.

All docs are **living**. New facts about clients, workflows and constraints arrive
continuously and get folded into the relevant file — the docs are never "finished".

## Conventions

Full set in `docs/07-conventions.md` — read it before writing code. The load-bearing few:

- English identifiers, comments and commits
- Actions: `App\Actions\Ticket\CreateTicket` — one public method, `handle()`
- Tests: Pest. Tenant-isolation tests are mandatory for every tenant-owned model
- Migrations are forward-only once the pilot is live (`08-environment.md`)
- `composer check` (Pint + Larastan level max + Pest) passes before anything is pushed
- Translation keys are `file.context.item`; a key missing from `pl` fails CI
