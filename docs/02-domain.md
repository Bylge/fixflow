# 02 — Domain Model

## Entity map

```
Tenant (company)
 ├─ Membership (user ↔ tenant)      → role, visibility scope
 ├─ Role (tenant-defined)           → set of permissions
 ├─ Status / Priority / Category    → tenant-configurable dictionaries
 └─ Ticket
     ├─ TicketMessage               → public reply | internal note
     ├─ Attachment
     └─ TicketEvent                 → history / audit trail

User (global identity, may belong to several tenants)
Permission (fixed catalogue, defined in code, not in DB rows the client edits)
```

## Tenant

A company. The unit of isolation. Every tenant-owned table carries `tenant_id`, enforced
by a global scope — see `03-architecture.md`.

The super admin (project owner) exists **outside** any tenant and is not a role within
one.

## User and Membership

`User` is a global identity: email, password, name, locale, timezone.

`Membership` binds a user to a tenant and carries everything tenant-specific:
- `role_id` — **exactly one role**, not a set
- visibility scope
- active/inactive

One role per membership. Union-of-roles is configurability nobody has asked for, and the
pivot table is an additive migration on the day someone does.

A user can belong to more than one tenant. This costs nothing now and avoids an ugly
migration later.

**The super admin is not a membership.** It is a boolean on the user record, read by
exactly one thing: the admin panel's access gate. It never appears in tenant authorization
— that is what `03-architecture.md` means by no `if ($user->is_admin)`.

## Roles and permissions

**Permissions** are a fixed catalogue defined in code. Draft list — deliberately short,
because an office manager has to understand it:

| Permission | Meaning |
|---|---|
| `ticket.create` | Report a problem |
| `ticket.comment` | Reply on a ticket |
| `ticket.note` | Write internal notes (invisible to reporters) |
| `ticket.take` | Assign a ticket to yourself |
| `ticket.assign` | Assign a ticket to someone else |
| `ticket.edit` | Change status, priority, category |
| `ticket.delete` | Remove a ticket |
| `workspace.access` | Enter the staff workspace at all |
| `settings.manage` | Statuses, priorities, categories, tenant settings |
| `users.manage` | Invite users, assign roles, create roles |

There is no `ticket.view`. What you may see is answered by visibility scope plus the shell
rule below, never by a permission — merging the two axes is the mistake this model exists
to avoid.

**Roles** are tenant-defined bundles of those permissions. The client creates, renames and
deletes them freely, with one guard: a tenant must always retain at least one active
membership holding `users.manage`, or it locks itself out. Three ship as defaults on tenant
creation:

- **Reporter** — `ticket.create`, `ticket.comment`
- **Agent** — Reporter + `ticket.note`, `ticket.take`, `ticket.edit`, `workspace.access`
- **Admin** — everything

## Visibility scope

A **separate axis** from permissions, stored on the membership. What you may *do* and what
you may *see* are different questions and must not be merged.

- `own` — only tickets I reported or am assigned to
- `all` — every ticket in the tenant

(`team` is deliberately absent — see `04-scope.md`.)

## The shell rule

`workspace.access` alone decides which interface a user lands in:

- **has it** → staff workspace (Filament panel)
- **lacks it** → reporter portal (custom Livewire)

This is the answer to "who is a user and who is a reporter": nobody is either. There is
one permission, and the interface follows from it.

## Ticket

| Field | Notes |
|---|---|
| `tenant_id` | Always |
| `requester_id` | User who reported it |
| `assignee_id` | Nullable |
| `status_id` | FK to tenant's status dictionary |
| `priority_id` | FK to tenant's priority dictionary |
| `category_id` | Nullable FK |
| `subject`, `body` | Free text, tenant data, never translated |
| `number` | Human-readable per-tenant sequence (`#128`), not the DB id |
| timestamps | Created, updated, first responded, closed |

Deliberately **not** present, and why: `client_company_id` (Case B only — nullable column,
cheap to add later), `due_at` (no SLA yet), custom fields, tags, parent/child links, merge.

### Numbering

`tenants.next_ticket_number`, incremented under `SELECT … FOR UPDATE` inside the same
transaction as the insert, with a unique constraint on `(tenant_id, number)` as the
backstop. Not `max(number) + 1` — two people reporting at once is not a rare event in a
system whose whole pitch is that everyone reports in one place.

### `first_responded_at`

Set once, by `PostMessage`, on the first **public reply** authored by someone who is not the
requester. Internal notes never set it. It has no consumer today; it is recorded now because
it is unrecoverable later, and it is the one number any future SLA conversation starts from.

## TicketMessage

Two kinds, distinguished by a flag:

- **Public reply** — visible to the reporter in the portal
- **Internal note** — visible only to users with `workspace.access`

The boundary between these two is the single most important correctness property in the
system. A leaked internal note is the failure mode that loses a client. It is enforced at
the query level, not in the view.

## Status, Priority, Category

Tenant-configurable dictionaries: name, colour, sort order, active flag.

Every **status** additionally carries a fixed `type` the client cannot change:

`new` · `open` · `waiting` · `done` · `cancelled`

The client may create "Czeka na części" and map it to `waiting`. The UI shows their name;
every piece of logic in the system reads the type.

**Lifecycle of a dictionary row.** Hard delete is allowed only while nothing references the
row. Once a ticket has used it, the only operation is `active = false`: it disappears from
every picker, and existing tickets keep it. A ticket may never point at a status that no
longer exists, and history may never be rewritten to say something that did not happen.

**Defaults.** Each tenant flags exactly one status `is_default` (type `new`) and one
`is_default_open` (type `open`). New tickets get the first. The second exists for one rule
below. Neither may be deleted or deactivated while flagged.

**Reopening.** A public reply from the requester on a ticket whose status type is `done`
moves it to the tenant's `is_default_open` status. This is a fixed rule in `PostMessage`,
not the first automation rule — "the customer answered and nobody noticed" is the failure
this product exists to prevent, and leaving it to a human to spot reintroduces it. Type
`cancelled` does not reopen.

## TicketEvent

Append-only history: status changed, assignee changed, priority changed, message added.
Renders as the timeline on the ticket page and doubles as the audit trail.

Kept simple: actor, event type, old value, new value, timestamp. Not a generic
event-sourcing system.

## Notification preferences

Per-user, per-tenant, minimal: an on/off switch for email on the events that matter
(assigned to me, new reply on my ticket). Anything more granular is a later feature.
