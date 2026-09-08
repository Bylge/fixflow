# 05 — Open Questions

Everything not yet decided. Check here before assuming an answer exists.

## Blocking nothing yet, but needed before launch

**Where the pilot actually runs.** The client's own server was the stated preference; the
architecture makes this a late decision, so it stays open. It determines the handover
mechanism (Docker image vs. a server we control) and who holds the backups.

It no longer blocks anything early: deployment is **M11** as of 2026-09-08, and moving the
pilot elsewhere later is a deployment-target change, not an architecture change. Needed by
M11, which is itself needed by M10.

**Backup ownership.** If it runs on their infrastructure, this must be agreed in writing
before go-live, and a restore tested once.

**Mail sending.** Which transactional provider, on which domain, with SPF/DKIM configured
by whom. Blocks notifications, which are what make people actually use the system —
specifically it blocks `06-build-plan.md` M8, so it must be answered by the end of M7.

**Domain and name.** `fixflow.pl`? Subdomain-per-tenant assumes a domain exists, and
`03-architecture.md` now reserves `admin.` on it. Needed at M2, when subdomain resolution
is built — enumerated `*.fixflow.test` hosts work for local and CI. A real domain is not
needed until M11 and mail.

## Product

**Does the pilot firm actually work the way we assumed?** The whole domain model rests on
a described workflow, not an observed one. Twenty minutes of "show me how a request
arrives and what you do with it" would either confirm this or save months. Highest-value
open item on this page.

> **Status 2026-09-07: deferred as a milestone.** `06-build-plan.md` M0 is the work; it is not
> scheduled, and the deferral expires at the end of M4. What changed is not the answer but the
> plan for getting one — the question stays open, stays the highest-value item here, and
> becomes blocking when M5 starts (see the re-plan triggers in `06-build-plan.md`). Deferring
> it did not make it smaller.

**AI features in the product.** "AI-heavy" was stated in the context of AI-assisted
*development*, which is covered by `CLAUDE.md`. Whether the product itself should ever do
triage, summarisation or suggested replies is unanswered — and firmly out of the MVP
either way.

**Does Case B ever become real?** If the pilot firm resells FixFlow to the client companies
it already serves, the cross-company desk stops being hypothetical. Cheap to add, but it
changes the product's positioning entirely.

**Is the portal enough as an intake channel?** If pilot users keep messaging on Messenger
anyway, the problem was never the tooling, and email intake becomes urgent rather than
deferred.

## Design and frontend

The *stack* is decided (`03-architecture.md`); the *design* is not. Those are different
questions, and the docs currently answer only the first — which is what makes the gap
invisible rather than open.

Not a gap at all: **the Filament surfaces need no design work.** Buying the admin panel and
the staff workspace is `01-principles.md` §6 working as intended.

The real gap is the reporter portal: three hand-built screens with no visual decision behind
them. **It comes due at M5**, not at launch — M5 puts a working report form in front of real
eyes.

Genuinely undecided, and the author's call:

- **Phone or desk?** Someone reporting a broken machine may well be standing next to it.
  This changes the layout fundamentally; it is not "add breakpoints later"
- **Visual identity.** Tied to the name and domain question above — no logo, palette or type
  choice exists
- **How much design at all.** Three screens, used twice a month, by non-technical people, is
  a brief that argues for plain and obvious over designed

Answerable now by the razor, absent an objection:

- Filament panels ship stock. Per-tenant theming is a feature nobody has asked for
- Email templates are Laravel's default markdown mail
- Accessibility floor: real labels, visible focus, keyboard-operable forms, passing contrast.
  Cheap now, expensive retrofitted, and this audience is precisely who suffers without it
- Tailwind's major version is verified at install, like every other dependency

This graduates to `09-frontend.md` the day it is decided. Deciding it mid-step inside M5 is
the failure mode — it is a conversation of its own, held before M5 starts.

## Commercial and legal

**Both moved out of git on 2026-09-08.** Pricing, whether the pilot pays, contract terms,
GDPR/processor status, business entity and liability now live in
`docs/private/01-pilot-and-commercial.md`. Still open, still unanswered, still read by agents
locally — simply not published. Making the repository public is the fallback if the project
finds no clients, and that fallback stays cheap only while this separation holds
(`docs/private/README.md`).

Nothing technical moved. If a commercial or legal answer ever constrains the architecture,
the *constraint* is recorded here in engineering terms and the reasoning stays private — for
example, self-hosting on a client's own server is already argued for in `03-architecture.md`
on its technical merits alone.

## Technical, deferred by design

- Queue driver — `database` through M7; the choice is which engine replaces it at M8, and
  `08-environment.md` records Valkey over Redis on licence grounds when that day comes
- Whether Filament's built-in tenancy is layered over the global scope, which enforces
  regardless — decided at M3, when the panels are wired
- Error tracking and uptime monitoring — needed before real users, not before code
- Attachment size and MIME allowlist — a number to pick at M6, not a design question

## Recently closed

Recorded so nobody reopens them by accident. Full reasoning lives in the linked doc.

| Question | Decision | Where |
|---|---|---|
| Roles per membership | Exactly one; pivot is an additive migration if ever needed | `02-domain.md` |
| How the super admin is modelled | Boolean on the user, read only by the admin panel gate | `02-domain.md` |
| Per-tenant ticket numbering | Counter on the tenant row, `FOR UPDATE`, unique `(tenant_id, number)` | `02-domain.md` |
| Status of a new ticket | Tenant flags one `is_default` status of type `new` | `02-domain.md` |
| Deleting a status in use | Deactivate only; hard delete while unreferenced | `02-domain.md` |
| Requester replies to a closed ticket | Auto-reopens to the tenant's default `open` status | `02-domain.md` |
| Policies or Actions for authorization | Both — Policies hide, Actions enforce, one shared check | `03-architecture.md` |
| Who writes `TicketEvent` | The Action, never an observer | `03-architecture.md` |
| Tenant context in queued jobs | Explicit `tenant_id` plus job middleware; `NOT NULL` in the schema | `03-architecture.md` |
| One login or two | One per tenant; Filament's login disabled, shell rule redirects | `03-architecture.md` |
| Session shared across subdomains | No. Per-host cookie; multi-tenant users log in twice | `03-architecture.md` |
| Tenancy / permission packages | Neither. Trait + global scope; enum + JSON column | `03-architecture.md` |
| Test database | PostgreSQL everywhere, never SQLite | `06-build-plan.md` |
| Local dev environment | WSL2 + Docker Compose for backing services; repo inside WSL | `08-environment.md` |
| Local wildcard subdomains | Enumerated hosts entries, not dnsmasq | `08-environment.md` |
| How changes reach main | Branch + PR, four CI jobs gate the merge, squash | `07/08` |
| Static analysis strictness | Larastan level max, no baseline | `07-conventions.md` |
| Translation key convention | `file.context.item`; missing `pl` key fails CI | `07-conventions.md` |
| Rollback | Code redeploys; schema is forward-only, destructive changes split over two releases | `08-environment.md` |
| Stack versions | Laravel 13, PHP 8.5, Filament 5, Livewire 4, Pest 4 — verified Aug 2026, re-verify at install | `CLAUDE.md` |
| Unit of work | The step, not the milestone: one branch, one PR, one sitting, agreed in advance | `06-build-plan.md` |
| Whether to run M0 before Phase 1 | Deferred 2026-09-07, not cancelled; expires at the end of M4 | `06-build-plan.md` |
| Docker vs. scripted deploy | Scripted. One box, one app, no registry | `03-architecture.md` |
| When deployment happens | M11, not M1. A departure from `01-principles.md` §9, accepted and recorded | `06-build-plan.md` |
| Pest major version | Pest 5, not 4 — taken while the repo has no tests | `CLAUDE.md` |
| Cache/queue container at M1 | None. `database` drivers until M8, then Valkey not Redis | `08-environment.md` |
| Commercial and legal notes | Moved to gitignored `docs/private/` so the repo can go public | `05-open-questions.md` |
