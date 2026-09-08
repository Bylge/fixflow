# 06 — Build Plan

The step-by-step sequence `04-scope.md` deferred. What gets built, in what order, and how
we know a step is finished.

## Working a milestone

Milestones are too big to be a unit of work. **The unit is a step**, and the execution rules
for one are in `CLAUDE.md` — read those first; this section only defines the granularity.

A step is: **one branch, one PR, one sitting, one check that either passes or doesn't.**
Steps are numbered inside their milestone — `M2.1`, `M2.2` — and worked strictly in order.

**Before a milestone starts, its step list is proposed and agreed.** Not invented while
coding. A milestone whose steps cannot be listed in advance is not understood well enough
to start, and that is information worth having before the first file is written.

The milestone's exit criterion is checked once, after its last step. Individual steps do not
get to claim the milestone is done.

Example — M2 decomposed:

| Step | Does | Check |
|---|---|---|
| M2.1 | `Tenant`, `User`, `Membership` migrations, models, factories | Factories build a full tenant in one line |
| M2.2 | `BelongsToTenant` trait, global scope, `TenantContext` | Scope applies without an explicit `where` |
| M2.3 | Subdomain resolution middleware, 403 on non-membership | Wrong-tenant host returns 403, not an empty list |
| M2.4 | Isolation test helper, tests for all three models | Every tenant-owned model has a passing isolation test |

Four steps, four PRs, four days of small green diffs — instead of one branch that touches
everything and is impossible to review or revert.

## Rules for the plan itself

1. **A milestone is done when its exit criterion passes.** No partial credit, no "mostly
   M3". Half-finished milestones are how the foundations rot.
2. **Tests ship with the milestone, not after it.** The four categories in
   `03-architecture.md` apply to every milestone that touches a model or an Action.
3. **The API is not a milestone.** Every Action gets its endpoint in the same milestone the
   Action is written — that is what "two doors, one room" costs. M9 is only the parts that
   have no Action behind them: auth, pagination, error shape.
4. **Nothing from "out of MVP" enters without a recorded decision** in `05-open-questions.md`.
5. Sizes are relative (S/M/L), not dates. `04-scope.md` sets the schedule policy: the
   timeline bends, the scope does not.

---

# Phase 1 — Foundations

`01-principles.md:2` — these four cannot be retrofitted. Nothing in Phase 2 starts until
Phase 1 is complete, because everything in Phase 2 is cheap afterwards and unfixable before.

## M0 — Ground truth · S · DEFERRED 2026-09-07 · expires at the end of M4

**Deferred, not dropped and not descoped.** The observation has not happened and nobody is
scheduled to do it. The reason this milestone was written as parallel is the reason it can be
deferred at all: Phase 1 does not depend on the answer. What does depend on it is Phase 2.

What the deferral costs: `02-domain.md` continues to rest on a described workflow rather than
an observed one, and every Phase 2 milestone inherits that. The cost is not paid at M0 — it is
paid at M5, when the first real form goes in front of real people.

The highest-value open item on `05-open-questions.md` is that the domain model rests on a
described workflow, not an observed one. Twenty minutes with the pilot firm: *show me how a
request arrives today and what you do with it.*

Deliberately parallel — the foundations don't depend on the answer. Everything in Phase 2
does.

**Exit (unchanged):** `02-domain.md` either updated or explicitly confirmed against
observation, with a dated note saying which.

**Re-entry — the deferral expires when M4 ends.** M5 is the first milestone that writes the
described workflow into an Action and puts a form in front of a real reporter, and at that
point an unobserved domain model stops being a cheap risk. Either the twenty minutes happen
before M5.1, or the decision to ship the domain model unvalidated is recorded in
`05-open-questions.md`, dated, with a name against it. Starting M5 having done neither is the
failure this note exists to prevent. Whichever comes first also counts: if contact with the
pilot firm happens for any other reason, the twenty minutes are free.

## M1 — Walking skeleton · M

Fresh Laravel, PHP 8.5, PostgreSQL, Pest, Filament installed. CI running the suite on every
pull request, and a red build that actually blocks the merge.

**Dependency floor**, re-verified 2026-09-08 — Laravel 13, PHP 8.5, Filament 5, Livewire 4,
Pest 5, plus Sanctum, Pint, Larastan. Filament 5 requires Livewire 4; the two move together,
and Livewire arrives transitively with Filament rather than as a separate require.

Re-verify all of it at install and **record the resolved versions here**. Do not take a
version number from an agent's memory, including mine — this list was wrong within three
months of being written, which is exactly why the rule exists. Nothing else added without a
reason written down; see the packages we deliberately skip in `03-architecture.md`.

If PHP 8.5 blocks a package at install, drop to 8.4 and record why. Not 8.3 — though the
mechanism is softer than first recorded: Laravel 13 declares `php ^8.3` and its `symfony/*`
constraints all read `^7.4 || ^8.0`, so 8.3 resolves the older Symfony 7.4 line silently
rather than failing. The reason to be on 8.5 is support dates, not a hard floor
(`CLAUDE.md`).

PHP 8.5 is not in stock Ubuntu 24.04, which tops out at 8.3. It comes from `ppa:ondrej/php`.
Two package-level traps, both verified: `php8.5-opcache` **does not exist** — OPcache is
compiled into the core packages and naming it aborts the whole apt transaction — and
`ext-intl` is a hard `composer require` of `filament/support` that appears on neither
Laravel's nor Filament's stated requirements list.

**CI and local both run PostgreSQL, never SQLite.** The tempting in-memory shortcut diverges
from production on exactly the three things this system leans on: global scopes over JSON
columns, `SELECT … FOR UPDATE` behind ticket numbering, and constraint timing. A green suite
that proves nothing about production is worse than a slow one.

**Deployment moved out of M1 on 2026-09-08 — a deliberate departure from
`01-principles.md:78`, recorded rather than glossed.** The principle asks for deployable from
week one, and this plan previously refused to defer it. The owner's decision is that FixFlow
is primarily a portfolio project until a client exists, and paying for a server to host
something with no users buys no feedback. Deployment is now **M11**, and its cost of being
late is accepted.

What M1 still owes M11, so that lateness stays cheap: configuration comes from the
environment and never from a hardcoded path, migrations carry the schema, and the `/up`
health route exists from the skeleton onward. The expensive thing to retrofit is an
application that assumes it runs on a laptop — not a CD workflow, which is a day's work
against an app already shaped for it.

Local setup, the four CI jobs and the deploy steps are specified in `08-environment.md`;
`composer check` and the definition of done in `07-conventions.md`. M1 is where both stop
being documents and start being enforced. **The repo moves into the WSL2 filesystem here** —
doing that once branches are in flight is needless friction.

**Exit:** all four CI jobs green on a pull request, and a deliberately failing test turns the
build red *and* leaves the pull request unmergeable. The live-URL clause moved to M11 with
the rest of deployment.

### Steps — agreed 2026-09-08

| Step | Does | Check |
|---|---|---|
| M1.1 | Repository public + all-rights-reserved `LICENSE`; ruleset on `main` requiring a pull request | A direct push to `main` is **rejected**, and `gh api repos/Bylge/fixflow/rulesets` returns one `active` ruleset |
| M1.2 | WSL toolchain: PHP 8.5 + extensions, Composer, Node 24, git, gh, Docker | One chained command reports PHP 8.5, every required extension, and a reachable `docker` |
| M1.3 | Repo re-cloned into `~/code/fixflow`; Windows copy archived; session moves | `git -C ~/code/fixflow log` shows the same three commits; the Windows copy is renamed, not deleted |
| M1.4 | Compose: `postgres` and `mailpit`, with the PostgreSQL 18 volume path proven | `docker compose up -d` reports healthy, and a row survives `down` then `up` |
| M1.5 | Laravel 13 skeleton on PostgreSQL with Pest 5; SQLite eradicated | `artisan migrate` succeeds against `pgsql` and no `sqlite` reference survives anywhere |
| M1.6 | The gate: `pint.json`, `phpstan.neon`, `composer check` | `composer check` exits 0 from a clean tree and leaves it clean |
| M1.7 | Filament 5 installed as a package — no panel, no resources | `composer show --locked filament/filament` reports 5.x and `artisan about` exits 0 |
| M1.8 | Frontend toolchain and first asset build | `npm ci && npm run build` produces `public/build/manifest.json` |
| M1.9 | Resolved versions recorded back into this file | A script asserts every version recorded here matches the lockfiles |
| M1.10 | CI: `lint`, `static`, `test` against a PostgreSQL service container, then added to the ruleset as required checks | Those three jobs conclude `success` on a pull request, and the ruleset lists all three |
| M1.11 | CI: `i18n` — `en`/`pl` key parity, added as the fourth required check, red path proven | Four jobs green and all four required; a deliberately unpaired key turns `i18n` red, then is reverted |
| M1.12 | Exit proof: red blocks the merge | A failing test leaves the PR unmergeable; removing it makes it mergeable |

Twelve steps, worked in order.

**Why the required status checks arrive at M1.10 and M1.11 rather than M1.1.** A ruleset that
requires a check no workflow produces leaves every pull request permanently unmergeable —
M1.1 would wedge the milestone it opens. So M1.1 turns on the pull-request requirement alone,
and each check becomes required in the step that creates the job behind it. The gating is not
weakened and does not slip out of the milestone; it is attached to the thing it gates. M1.12
proves the whole mechanism, which is where the exit criterion is actually met.

**Public, not paid.** Merge gating needs rulesets, which are free on a public repository and
a paid feature on a private one. The repository was made public at M1.1 under an
all-rights-reserved `LICENSE` — readable, not open source, and no commercial right is
granted. Commercial and pilot-identifying material had already moved to `docs/private/`. The
side benefits are unlimited Actions minutes and secret-scanning push protection, both of
which this plan leans on.

M1.2 still needs things only the owner can supply: a sudo password typed interactively, and
the Docker Desktop WSL-integration toggle.

## M2 — Tenancy and identity · L

`Tenant`, `User`, `Membership`. The `BelongsToTenant` trait: global scope plus `tenant_id`
auto-fill. Subdomain resolution, with the single-membership fallback. Factories that make a
second tenant free to create, because otherwise nobody writes the isolation tests.

**Exit:** two tenants exist with overlapping data; the isolation test for every tenant-owned
model passes; a logged-in user hitting another tenant's host gets 403, not an empty list.

## M3 — Permissions and the shell rule · M

The fixed permission catalogue in code. `Role` as a tenant-owned bundle, three defaults
seeded on tenant creation. `Membership::hasPermission()` as the single check everything
reads. Three shells wired but empty: super-admin panel, staff workspace, reporter portal.

**Exit:** a user with `workspace.access` lands in the workspace, one without lands in the
portal, and neither can reach the other's routes. Super admin reaches the admin panel and
no tenant surface.

## M4 — Internationalisation · S

`en` and `pl` translation files, per-user locale and timezone, UTC storage, locale-aware
date formatting decided once. A check in CI that fails on literal user-facing strings in
Blade.

Small, and permanently expensive to skip — `01-principles.md:20`.

**Exit:** the entire Phase 1 skeleton renders in both languages with no literal strings, and
the CI check catches a deliberately hardcoded one.

---

# Phase 2 — Product

## M5 — First vertical slice · M

`CreateTicket` end to end: reporter portal form → Action → ticket with an allocated number →
visible in the staff list. Plus the `POST /api/v1/tickets` endpoint calling the same Action.
Status/priority/category dictionaries seeded with defaults.

This milestone's real output is **the pattern** — Action contract, authorization split,
event write, transaction boundary, test shape. Every later Action copies it, so it is worth
getting slowly right.

**Exit:** a ticket created through the portal and one created through the API are
indistinguishable in the database. All four test categories exist for this slice.

## M6 — Ticket lifecycle · L

`PostMessage` (public reply and internal note), `AssignTicket`, `TakeTicket`,
`ChangeStatus`. `TicketEvent` timeline. Attachments through the authorized controller route.
Endpoints alongside.

The largest milestone and the one carrying the system's most important correctness
property.

**Exit:** the internal-note leak test passes at query level, not view level — a reporter
fetching their own ticket through both the portal and the API receives zero internal notes.
Full lifecycle drivable from either door.

## M7 — Configuration · M

Dictionary CRUD with the lifecycle rules from `02-domain.md`, role CRUD, user invitations,
tenant settings. Filament, mostly.

**Exit:** an office manager sets up a tenant from scratch — statuses, roles, users — without
a developer and without a seeder.

## M8 — Notifications · M

Supervised queue worker and scheduler in the deploy. Transactional mail provider with
SPF/DKIM on the real domain. Two notifications: assigned to me, new reply on my ticket.
Per-user on/off.

Depends on the mail decision in `05-open-questions.md`. If that is still open when M7 ends,
it is blocking by then — flag it at M5.

**Exit:** both emails arrive in a real inbox from the production domain, dispatched after
commit, with correct tenant context inside the job.

## M9 — API surface and finding tickets · M

Sanctum tenant-scoped tokens, cursor pagination, the error shape, rate limits. Ticket list
filters and search in the workspace.

**Exit:** the full lifecycle from `M6` drivable by `curl` alone, against a documented
`/api/v1`. Staff can find a three-month-old ticket in under ten seconds.

---

# Phase 3 — Pilot

Deployment lives here rather than in M1 — the decision, and what it costs, is recorded under
M1. **M11 runs before M10**, because M10's exit is checked on a production host that M11
creates. It is numbered after M10 anyway: milestone numbers are identity, cited from other
docs and from git history, and are never renumbered to reflect order.

## M11 — Deployment · M

A server exists, and one command ships a commit to it. **Scripted, not Docker** — decided
2026-09-08 (`03-architecture.md`): one box, one application, no orchestration and no
registry, which is what `01-principles.md` §9 means by boring. Revisit only if the pilot
lands on hardware we do not control.

An owner-controlled VPS, provisioned with PHP 8.5, PostgreSQL, nginx and php-fpm. TLS from
Let's Encrypt on the host itself. A dedicated `deploy` user and an SSH keypair used for
nothing else, its private half in Actions secrets. Assets are built **in CI** and shipped
with the release, so the server never needs Node. The CD sequence, the queue-worker restart
and the rollback rules are already specified in `08-environment.md`.

The production database password is generated on the host and lives only in the server's
`.env` — never in git, and not in Actions secrets either, because the deploy does not need
it.

**Exit:** one push to `main` ships that commit and `curl https://<host>/up` returns 200
without `-k`; a failed health check fails the deploy loudly.

## M10 — Hardening and go-live · M

Backup running and one restore actually tested. Error tracking and uptime monitoring. Rate
limits verified. Seed the pilot tenant with real users. Whatever the M0 observation said we
got wrong, if it is small — and if M0 never ran, M10 is the last honest opportunity to run it,
now against the pilot's real usage rather than a description of it. Go-live is the last moment
the answer is free.

Depends on M11: there is no production host to harden until deployment exists.

**Exit:** the go-live checklist passes on the production host, including a restore from
backup performed start to finish.

---

## Triggers to stop and re-plan

- **M0 is still deferred when M4 ends.** The deferral was granted because Phase 1 does not
  need the answer; at the end of Phase 1 that basis is spent. Before M5 planning starts,
  either run M0 or record the accepted risk, dated, in `05-open-questions.md`. Not both blank.
- **M0, whenever it runs, contradicts the domain model.** Re-plan before the next Phase 2
  milestone begins — before M5 if M5 has not started, and at the end of the current step if it
  has. Mid-step re-planning is the other failure. The trigger does not expire because the
  milestone was deferred.
- **Pilot users keep using Messenger.** The problem was never tooling; email intake stops
  being deferred (`05-open-questions.md`).
- **Case B becomes real before go-live.** Additive by design, but it changes positioning —
  a decision, not a ticket.
- **Any milestone doubles its size estimate.** Cut scope inside it rather than let it run;
  that is the razor applied to work rather than features.
