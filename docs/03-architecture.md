# 03 — Architecture

## Layers

```
        ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
        │  Filament    │  │  Livewire    │  │  REST API    │
        │  (workspace, │  │  (reporter   │  │  /api/v1     │
        │   admin)     │  │   portal)    │  │              │
        └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
               └─────────────────┼─────────────────┘
                                 ▼
                        ┌─────────────────┐
                        │     Actions     │   all business rules
                        └────────┬────────┘
                                 ▼
                        ┌─────────────────┐
                        │ Models / Domain │
                        └────────┬────────┘
                                 ▼
                           PostgreSQL
```

Three entry points, one domain. The UI does **not** call the API — both call Actions
directly. This is the "two doors, one room" principle from `01-principles.md`.

An Action is a single-purpose class with one public `handle()` method:
`App\Actions\Ticket\CreateTicket`, `AssignTicket`, `ChangeStatus`, `PostMessage`. Input is
validated by the caller; authorization and rules live in the Action.

### The Action contract

Every Action, without exception:

1. **Authorizes first.** Throws on failure — it never returns a boolean the caller might
   forget to check.
2. **Takes a readonly DTO** once it needs more than two inputs (`CreateTicketData`).
   Positional arguments rot as parameters accumulate.
3. **Runs inside one transaction**, covering the write and its `TicketEvent`.
4. **Writes its own `TicketEvent`.** Never a model observer: an observer cannot see who did
   it or why, and half the value of the timeline is the actor.
5. **Dispatches notifications after commit** (`->afterCommit()`). A job that starts before
   the transaction lands reads a row that does not exist yet, intermittently, in production
   only.
6. **Returns the affected model.** Not a bool, not an array.

### Authorization lives in one place, checked in two

`Membership::hasPermission()` is the single source. It is consulted twice, for different
reasons, and both are required:

- **Policies** — so Filament and Blade can hide what you cannot do. Cosmetic. They delegate
  to `hasPermission()`; they never contain a rule of their own.
- **The Action** — so it is actually enforced. The API has no buttons to hide, and a hidden
  button is not a security control.

An Action that trusts its caller's check is a bug even when every current caller checks.

## Multi-tenancy

**Shared database, `tenant_id` column, global scope.** Not schema-per-tenant, not
database-per-tenant — both multiply migration and backup work beyond what one developer
should carry.

- Every tenant-owned model uses a `BelongsToTenant` trait applying a global scope and
  auto-filling `tenant_id` on create
- Tenant resolution: subdomain (`firma.fixflow.pl`) or, in a single-tenant install, the
  logged-in user's only membership
- The super-admin panel is the sole place that may query across tenants, and it does so
  by explicitly removing the scope
- **Tenant isolation is tested.** Every tenant-owned model gets a test asserting that
  tenant A cannot read or write tenant B's rows. This is not optional coverage.

A single-tenant deployment is simply an install with one tenant. There is no separate
code path, and no decision about self-hosting has to be made before launch.

The global scope is the enforcement layer. Whether Filament's own tenancy support is
layered on top of it is still open (`05-open-questions.md`) and changes nothing below it.

### Tenant context outside HTTP

The predictable failure of this design: a queued job, a console command or the scheduler
runs with no host to resolve a tenant from. The global scope then either filters everything
out or, worse, writes a row belonging to nobody.

- `tenant_id` is `NOT NULL` on every tenant-owned table. The database refuses the ambiguous
  write rather than storing it.
- **`Queue::createPayloadUsing()` stamps `tenant_id` onto every job payload** at dispatch,
  and job middleware restores the context before `handle()` runs. Stamping centrally rather
  than per-job matters: the job someone writes in a hurry six months from now is stamped
  too, and remembering to add a property is not a control.
- **Context is torn down after every job**, pass or fail. A worker is a long-lived process
  handling many tenants in sequence; context left standing leaks into the next job on that
  worker, which produces a cross-tenant write that no test of a single job will ever catch.
- Console commands that touch tenant data take a tenant argument. There is no "current
  tenant" default outside a request.
- Two tests, not one: a job dispatched under tenant A still resolves tenant A when it runs,
  **and** a job for tenant B running immediately after one for tenant A on the same worker
  sees only B.

## Panels and surfaces

| Surface | Built with | Audience |
|---|---|---|
| Super-admin panel | Filament | Project owner only. Tenants, users, diagnostics |
| Staff workspace | Filament + custom pages | Anyone with `workspace.access` |
| Ticket conversation view | Custom Filament page | Inside the workspace, same nav and styling |
| Reporter portal | Livewire + Blade + Tailwind | Everyone else. Three screens |

The reporter portal screens: *report a problem*, *my reports*, *one report*. Nothing else.

### Hosts and routing

| Host / path | Serves |
|---|---|
| `admin.fixflow.pl` | Super-admin panel. Reserved subdomain; no tenant may claim it |
| `{tenant}.fixflow.pl/` | Reporter portal |
| `{tenant}.fixflow.pl/app` | Staff workspace, behind `workspace.access` |
| `{tenant}.fixflow.pl/api/v1` | REST API, tenant resolved from the host exactly as the UI does |

A logged-in user requesting a tenant they have no active membership in gets **403**, not a
login screen and not an empty list — the difference between the three leaks whether that
tenant exists.

## REST API

- Versioned: `/api/v1`
- Token auth via Laravel Sanctum; tokens are tenant-scoped
- Covers what the UI covers, because both call the same Actions
- MVP surface: authenticate, list tickets, read ticket, create ticket, comment, change
  status, assign
- JSON only, standard HTTP semantics, cursor pagination
- One error shape everywhere — `{ "message": …, "errors": { field: […] } }`, Laravel's
  validation envelope, used for every non-2xx including authorization failures

The API is a first-class citizen from the first week, not something bolted on later. It is
also the cheapest possible integration story for any future client.

## Internationalisation

- All system strings are translation keys. `en` and `pl` from day one
- Locale is per user, switchable, stored on the user record
- **Tenant-created content is never translated.** Status names, categories, ticket bodies
  are data in whatever language the client typed
- Timezone per user; all timestamps stored UTC
- Date and number formatting follows locale, decided once at the start

## Notifications

Queued mail. Redis if available, database queue otherwise — the queue driver is a config
decision, not an architectural one. A supervised worker and the scheduler are part of the
deployment from the beginning, because retrofitting a queue into a synchronous app is
tedious.

Mail goes through a real transactional provider with SPF/DKIM configured. `mail()` and
unauthenticated SMTP are not acceptable for a system people rely on.

## Storage

Attachments on the local filesystem behind Laravel's storage abstraction, so an
S3-compatible backend is a config change. Uploads are validated by MIME type and size,
stored outside the web root, and served through an authorized controller route — never by
direct URL.

## Authentication and security

- Session auth for UI, Sanctum tokens for API
- Rate limiting on login, password reset and the API
- Password reset, email verification, user invitations
- 2FA: not in the MVP, but the auth flow must not make it painful to add
- Authorization always goes through the permission catalogue — no `if ($user->is_admin)`
  anywhere

### One login, two destinations

A tenant has a single login page, hand-built, on the tenant host. Filament's own login is
disabled; the staff panel authenticates against the same `web` guard and redirects
unauthenticated visitors to that page. After login the shell rule decides where you land:
`workspace.access` → `/app`, otherwise `/`.

Two login pages for one company is a support burden nobody signed up for, and "which URL do
I use" is exactly the confusion this product replaces.

The super-admin panel on `admin.` keeps its own login. Different host, different audience,
and no reason for those sessions to touch.

### Sessions are per-host

The session cookie is **not** shared across subdomains — no `SESSION_DOMAIN=.fixflow.pl`. A
wildcard cookie means any tenant host, including one we later hand to a client, sits inside
the same session boundary as every other tenant and as `admin.`.

The cost is that a user belonging to two tenants logs in twice. That user is hypothetical
today (`02-domain.md`), and the isolation is not.

## Packages we deliberately do not use

Both of these solve our problem in general and would cost more than they save here. Written
down so nobody helpfully installs one later.

**No tenancy package** (`stancl/tenancy`, `spatie/laravel-multitenancy`). They are built
around database- and schema-per-tenant, which `03` already rejected. Our enforcement layer
is a trait, a global scope and a context singleton — small enough to read in one sitting,
which matters more than features for the thing that stops tenant A seeing tenant B.

**No permission package** (`spatie/laravel-permission`). It models globally-scoped roles and
guard names; ours are tenant-owned rows over a catalogue fixed in code. Bending it to that
shape costs more than the alternative:

- `Permission` is a PHP enum. That *is* the catalogue `02-domain.md` requires — it cannot
  drift into client-editable rows, because it is not rows.
- `roles.permissions` is a JSON array of enum values, tenant-owned.
- `Membership::hasPermission(Permission $p)` is the one function everything reads.

## Deployment

- One command, from week one. Daily updates during the pilot depend on it
- Docker or a scripted deploy — decided when the first server exists, not before
- Environments: local, and the pilot server. No staging until there is something to stage
- Forward-only migrations once the pilot has real data
- Backups: whoever owns the server owns the backups, agreed in writing before go-live. A
  restore is tested once before the pilot starts, not after the first incident

## Testing

Pest. The tests that must exist:

1. Tenant isolation per model
2. Permission enforcement per Action
3. Internal notes never appear in reporter-facing queries
4. Happy path for each Action

Coverage targets are not a goal; those four categories are.
