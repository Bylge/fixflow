# 07 — Conventions

How the code is written. `03-architecture.md` decides the shape of the system; this decides
the shape of a file.

**If a tool can enforce a rule, it is not prose here.** Everything below is either enforced
by Pint, Larastan or a test, or it is a naming decision no tool can make. Style opinions
that are neither do not belong in this document.

## The gate

`composer check` runs three things, in this order, and all three must pass before anything
is pushed:

| Tool | Setting |
|---|---|
| **Pint** | `laravel` preset, plus `declare_strict_types`. Not negotiable per-file |
| **Larastan** | **Level max, no baseline.** A greenfield project that generates a baseline on day one has a baseline forever |
| **Pest** | The full suite, against PostgreSQL |

Level max hurts for about a week and then stops hurting. Adopting it later never happens.

## Layout

Flat Laravel, not DDD modules. One bounded context and thirty users do not justify
`src/Domain/Ticketing/`. If a genuine second context ever appears, that is the moment to
split, and not before.

```
app/
  Actions/{Aggregate}/VerbNoun.php   CreateTicket, AssignTicket, PostMessage
  Data/                             readonly DTOs
  Enums/                            Permission, StatusType, TicketEventType
  Models/
  Policies/                         delegate to hasPermission(), never a rule of their own
  Support/                          TenantContext, BelongsToTenant
  Filament/  Http/                  entry points only — no rules (01-principles.md)

resources/views/
  components/                       Livewire 4 components live here, not app/Livewire
  pages/                            full-page components — the reporter portal's three
  layouts/
```

`Http/` stays thin: form requests validate, controllers call an Action, API resources
serialise. A controller with an `if` about domain state is a bug.

**Livewire 4 components are single-file by default** — PHP, Blade and any scoped CSS in one
file, which is what `make:livewire` now produces. Convert to multi-file
(`livewire:convert --mfc`) when a component passes roughly 150 lines, and expect the
reporter portal's three screens to stay single-file for a long time.

If a component is large enough that the multi-file question feels close, check first whether
logic has leaked in that belongs in an Action. Usually it has.

## Naming

| Kind | Convention | Example |
|---|---|---|
| Model | Singular | `TicketMessage` |
| Table | Plural snake | `ticket_messages` |
| Action | `VerbNoun`, one public `handle()` | `AssignTicket` |
| DTO | `…Data`, `final readonly` | `CreateTicketData` |
| Enum | Singular, backed by string | `StatusType` |
| Job | Verb first | `SendTicketAssignedMail` |
| Test | Mirrors the class under test | `tests/Feature/Ticket/AssignTicketTest.php` |
| Route name | Dotted, resourceful | `tickets.show`, `api.v1.tickets.store` |

Three rules the tools won't catch:

- **`final` by default.** Drop it only when something actually extends the class and you
  meant it to.
- **Return types everywhere.** `mixed` in a signature needs a comment saying why.
- **Enums over string constants, and over a boolean that will grow a third state.**

## Translation keys

The hard rule in `CLAUDE.md` — no user-facing string in code — only survives if the key
convention is decided before the first view. It is decided here.

- **One file per surface:** `lang/{locale}/ticket.php`, `user.php`, `settings.php`,
  `common.php`
- **Shape:** `file.context.item` — `ticket.status.updated`, `ticket.form.subject_label`
- **Three levels maximum.** A fourth means the file should have been split
- **Always `__()`.** No `@lang`, no inline default as a second argument — a default is a
  hardcoded string wearing a hat
- **Keys name the role, not the copy.** `ticket.form.subject_label`, never
  `ticket.form.what_is_the_problem`. Rewording the Polish must never rename a key
- **`en` is authoritative. A key present in `en` and missing in `pl` fails CI.** Silent
  fallback to English is how half a product ends up untranslated without anyone noticing
- **Tenant data is never a key** (`01-principles.md`). Status names, categories and ticket
  bodies are data in whatever language the client typed

The M4 lint is a Pest test: it scans Blade, Livewire and Filament resources for literal
user-facing strings and for key parity between locales. It runs in CI as its own job.

## Tests

The four mandatory categories are in `03-architecture.md`. These are the mechanics:

- **Feature tests by default.** `tests/Unit` only for pure logic that touches no database
- **Never mock the database, never mock an Action inside a feature test.** The thing being
  proven is that the real pieces fit together
- **Every model gets a factory**, and making a second tenant with overlapping data is one
  line — because an isolation test nobody can write cheaply is an isolation test nobody
  writes
- **A shared helper asserts scoping** so per-model isolation tests stay one-liners
- **Names describe behaviour:** `it('refuses to assign a ticket without ticket.assign')`

Every Action ships with three tests minimum: happy path, permission denied, tenant
isolation. Anything touching messages adds the internal-note leak test.

## Commits and branches

- **Conventional Commits**, short subset: `feat` `fix` `refactor` `test` `docs` `chore`
- Imperative mood, English, subject ≤ 72 characters
- Branch per change, short-lived, rebased on main: `feat/ticket-assignment`
- **One logical change per PR.** A PR carrying a migration and a UI redesign gets split
- Squash merge, so main reads as one commit per change

## Comments

Explain *why*. The *what* is the code, and a comment restating the line gets deleted.

Non-obvious domain rules get a comment pointing at the doc that decided them — the reopen
rule in `PostMessage` reads as an accident to anyone who has not read `02-domain.md`.

## Definition of done

No reviewer exists, so the gate is mechanical. A change is done when:

1. `composer check` passes locally
2. CI is green, all jobs
3. The Action's three tests exist and are named for behaviour
4. No new user-facing string lacks keys in **both** locales
5. `.env.example` carries any new variable, added in the same commit
6. Any decision that changed is written into the doc that owns it, in the same PR
