# 04 — Scope

Apply the razor (`01-principles.md`) to everything on this page except the foundations.

## MVP — what the pilot firm sees

**Foundations** (built properly, not cheaply)
- Multi-tenancy with enforced isolation
- Roles and permissions, tenant-editable, three defaults
- Action layer
- i18n, `en` + `pl`, switchable per user

**Tickets**
- Create, view, comment
- Internal notes, separated from public replies
- Attachments
- Assignment: take it yourself, or assign to someone (permission-gated)
- Status, priority, category — tenant-configurable dictionaries
- History timeline

**Interfaces**
- Reporter portal: report a problem, my reports, one report
- Staff workspace: ticket list with filters, ticket view, settings, user management
- Super-admin panel

**Supporting**
- Email notifications with a per-user on/off switch
- Search and filters on the ticket list. A search box and filters, not typeahead
  suggestions — the razor removed the suggestions
- REST API v1
- One-command deploy

## Explicitly out of the MVP

Not "later, probably" — out, until someone asks for them by name.

| Feature | Why it's out |
|---|---|
| Email → ticket, reply-by-email | The largest single piece of work in a helpdesk. The pilot reports via the portal |
| SLA, deadlines, escalation | No client has asked. Adding it later is additive |
| Reports and analytics | Vanity until there is enough data for the numbers to mean anything |
| Custom fields | Configurability we have no evidence anyone needs |
| Automation rules, canned responses | Same |
| Teams / departments | A 20-person firm does not need a second visibility axis |
| Merge, split, linked tickets | Volume-driven features. This is not a high-volume product |
| CSAT surveys | Case B only |
| Billing, subscriptions, self-serve signup | No paying customer exists yet |
| 2FA, SSO | Not asked for. Auth is built so it can be added without pain |
| In-app notification centre | Email is enough. This is polish |
| Client companies on tickets (Case B) | Nullable column, cheap to add the day it's needed |
| Marketing site | Not a product problem |

## Later, in rough order of likelihood

1. Email → ticket, once someone forwards a request instead of using the portal
2. Client companies on tickets, if Case B becomes real
3. Reports, once there are months of data
4. SLA, if a client ever promises response times to someone else
5. 2FA, the first time a client's security policy requires it

## Sequencing intent

The order is constrained by one fact: the four foundations are cheapest at the beginning
and most expensive at any other time.

So the first stretch of work is skeleton, not features: tenancy, auth, permissions,
i18n, action layer, deploy pipeline. A boring app that does almost nothing but does it
correctly for two tenants in two languages. Features are fast once that exists, and
impossible to add cleanly if it doesn't.

The step-by-step plan is `06-build-plan.md`. This page decides *what* is in; that one
decides *when*.

## Timeline

No deadline. Roughly one month of substantial availability, then significantly less. Target
is a working pilot within a realistic fraction of a year, with the explicit understanding
that the schedule bends and the scope does not grow to fill it.
