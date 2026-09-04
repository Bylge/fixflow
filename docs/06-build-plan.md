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

## M0 — Ground truth · S · runs in parallel, blocks nothing

The highest-value open item on `05-open-questions.md` is that the domain model rests on a
described workflow, not an observed one. Twenty minutes with the pilot firm: *show me how a
request arrives today and what you do with it.*

Deliberately parallel — the foundations don't depend on the answer. Everything in Phase 2
does.

**Exit:** `02-domain.md` either updated or explicitly confirmed against observation, with a
dated note saying which.

## M1 — Walking skeleton · M

Fresh Laravel, PHP 8.3+, PostgreSQL, Pest, Filament installed. CI running the suite on push.
One-command deploy to a real server, reaching a real health-check URL.

**Dependency floor**, verified August 2026 — Laravel 13, PHP 8.5, Filament 5, Livewire 4,
Pest 4, plus Sanctum, Pint, Larastan. Filament 5 requires Livewire 4; the two move together.

Re-verify all of it at install and **record the resolved versions here**. Do not take a
version number from an agent's memory, including mine — this list was wrong within three
months of being written, which is exactly why the rule exists. Nothing else added without a
reason written down; see the packages we deliberately skip in `03-architecture.md`.

If PHP 8.5 blocks a package at install, drop to 8.4 and record why. Not 8.3: Laravel 13
claims to support it, but Symfony 8 arrives transitively and requires 8.4.

**CI and local both run PostgreSQL, never SQLite.** The tempting in-memory shortcut diverges
from production on exactly the three things this system leans on: global scopes over JSON
columns, `SELECT … FOR UPDATE` behind ticket numbering, and constraint timing. A green suite
that proves nothing about production is worse than a slow one.

The deploy target being an open question does **not** defer this. `01-principles.md:78`
requires deployable from week one; what is deferred is Docker-vs-scripted and whose
hardware the *pilot* eventually runs on. Deploy to an owner-controlled VPS now, move it
later if the pilot lands elsewhere.

Local setup, the four CI jobs and the deploy steps are specified in `08-environment.md`;
`composer check` and the definition of done in `07-conventions.md`. M1 is where both stop
being documents and start being enforced. **The repo moves into the WSL2 filesystem here** —
doing that once branches are in flight is needless friction.

**Exit:** one command ships a commit to a live URL; all four CI jobs green; a deliberately
failing test turns the build red *and* blocks the merge.

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

## M10 — Hardening and go-live · M

Backup running and one restore actually tested. Error tracking and uptime monitoring. Rate
limits verified. Seed the pilot tenant with real users. Whatever the M0 observation said we
got wrong, if it is small.

**Exit:** the go-live checklist passes on the production host, including a restore from
backup performed start to finish.

---

## Triggers to stop and re-plan

- **M0 contradicts the domain model.** Re-plan before M5, not after.
- **Pilot users keep using Messenger.** The problem was never tooling; email intake stops
  being deferred (`05-open-questions.md`).
- **Case B becomes real before go-live.** Additive by design, but it changes positioning —
  a decision, not a ticket.
- **Any milestone doubles its size estimate.** Cut scope inside it rather than let it run;
  that is the razor applied to work rather than features.
