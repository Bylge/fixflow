# 05 — Open Questions

Everything not yet decided. Check here before assuming an answer exists.

## Blocking nothing yet, but needed before launch

**Where the pilot actually runs.** The client's own server was the stated preference; the
architecture makes this a late decision, so it stays open. It determines the handover
mechanism (Docker image vs. a server we control) and who holds the backups.

It no longer blocks the first deploy. `06-build-plan.md` M1 deploys to an owner-controlled
VPS from week one; moving the pilot elsewhere later is a deployment target change, not an
architecture change. Needed by M10.

**Backup ownership.** If it runs on their infrastructure, this must be agreed in writing
before go-live, and a restore tested once.

**Mail sending.** Which transactional provider, on which domain, with SPF/DKIM configured
by whom. Blocks notifications, which are what make people actually use the system —
specifically it blocks `06-build-plan.md` M8, so it must be answered by the end of M7.

**Domain and name.** `fixflow.pl`? Subdomain-per-tenant assumes a domain exists, and
`03-architecture.md` now reserves `admin.` on it. Needed at M2, when subdomain resolution
is built — a placeholder domain works for local and CI, but not for the M1 deploy target
or for mail.

## Product

**Does the pilot firm actually work the way we assumed?** The whole domain model rests on
a described workflow, not an observed one. Twenty minutes of "show me how a request
arrives and what you do with it" would either confirm this or save months. Highest-value
open item on this page.

**AI features in the product.** "AI-heavy" was stated in the context of AI-assisted
*development*, which is covered by `CLAUDE.md`. Whether the product itself should ever do
triage, summarisation or suggested replies is unanswered — and firmly out of the MVP
either way.

**Does Case B ever become real?** If the IT firm resells FixFlow to its pharmacy clients,
the cross-company desk stops being hypothetical. Cheap to add, but it changes the product's
positioning entirely.

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

## Commercial

**Pricing model.** Nothing decided. Per-agent/month is the category standard; a one-off
implementation fee is where solo developers actually earn. Not needed until someone offers
to pay.

**Is the pilot paid or free?** Free buys goodwill and a reference; paid filters for real
need. Undecided.

**Who signs, and for what.** No contract template, no terms, no scope of support.

## Legal

Only becomes real if data for companies other than the pilot ends up on infrastructure the
project owner controls:

- Processor status under GDPR/RODO, and a *umowa powierzenia* per client
- Privacy policy, terms of service
- Business entity: JDG vs. sp. z o.o.
- Liability for data loss, and whether it is capped in a contract

Self-hosting on the client's own server sidesteps most of this — which is an argument for
that model that has nothing to do with technology.

## Technical, deferred by design

- Queue driver (Redis vs. database) — config decision at deploy time
- Docker vs. scripted deploy — decided when the first server exists
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
