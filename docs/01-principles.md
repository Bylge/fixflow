# 01 — Principles

These are the rules that settle arguments. When a decision is unclear, the answer is
whichever option satisfies more of these.

## 1. The razor

If a feature can be removed and the system still does its job, remove it. Ship the most
bare-bones version that works. Features are added when someone actually needs them, not
because they seem obviously useful.

Corollary: features are built to work *correctly*, not to be complete. A working simple
version beats a half-finished sophisticated one.

## 2. Foundations are exempt from the razor

Four things cannot be retrofitted cheaply. They are built properly from day one
regardless of cost:

- **Multi-tenancy** — retrofitting tenant scoping means auditing every query in the app
- **Translation keys (i18n)** — retrofitting means touching every view and every string
- **The action layer** — retrofitting means untangling logic from three UI frameworks
- **The permission model** — retrofitting means rewriting every authorization check

Everything else is a feature and gets no such protection.

## 3. One source of truth for business rules

Every operation that changes state lives in exactly one Action class. Filament resources,
Livewire components and API controllers are entry points: they validate input, call the
Action, and render the result. They never contain rules.

This is what makes the API real rather than decorative, and it is what makes the UI
replaceable.

## 4. Two doors, one room

The UI talks to the domain directly. The API talks to the domain directly. The UI never
routes through the API. They are parallel entry points to the same Actions, and neither
depends on the other.

## 5. Configurable on top of fixed semantics

Clients define their own roles, statuses, priorities and categories. But every
client-defined status maps onto a fixed internal type (`new`, `open`, `waiting`, `done`,
`cancelled`), and every client-defined role is a bundle of permissions from a fixed
catalogue.

Without the fixed layer underneath, nothing in the system can reason about tickets — no
counters, no filters, no future SLA, no reports. Free-form configuration on top, stable
meaning below.

## 6. Buy work where UX doesn't matter, build it where it does

Filament is used for the super-admin panel and the staff workspace: list views, filters,
CRUD, settings, user management. It is mature, consistent and saves an enormous amount of
work in exactly the places where a custom design adds nothing.

The reporter portal is hand-built. It is three screens used by non-technical people who
open it twice a month, and Filament's density and restrictiveness would actively hurt
there.

## 7. Filament must stay evictable

Filament is a UI dependency, not an architecture. If it is ever removed, the cost must be
views and nothing else. That means: no business logic in resources, no domain concepts
that only exist as Filament constructs, no data model shaped by what Filament finds
convenient.

## 8. English in the code, Polish in the interface

All code, identifiers, comments, commits and documentation are English. Polish exists
only as a locale — a translation file. There is no bilingual code.

Client-created content (a status named "W trakcie", a category, a ticket body) is data,
not translatable text. The system never attempts to translate tenant data.

## 9. Deployable from day one

Deployment is one command, from the first week. During the pilot, updates ship daily —
that only works if deploying is boring. Manual deployment steps are a bug.

## 10. Documentation is for us

These docs assume a technical reader who has the full context. No simplification, no
tutorials. Non-technical material, if it is ever needed, is a separate artefact written
for a separate purpose.
