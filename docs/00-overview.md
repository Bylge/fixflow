# 00 — Overview

> Living document. Facts about clients and their workflows are added as they are learned;
> nothing here is final until it is contradicted by observation.

## What FixFlow V2 is

A multi-tenant ticket system for small companies. An employee reports a problem; it lands
in one place; someone with the right permissions picks it up, works it, and closes it.

The problem it replaces is not "we lack a ticket system" — it is **requests scattered
across Messenger, phone calls, and personal inboxes**, where things get forgotten and
nobody knows what state anything is in.

## Where it came from

V1 was a school project (Laravel + Blade + Livewire, single-tenant, hardcoded
Admin/Agent/Client roles). It worked, but it was not built to be run in production and
the codebase is not worth carrying forward.

**V2 is a fresh repository.** V1 stays available only as a reference for domain
decisions — what statuses existed, what fields mattered, what turned out to be useless.

## Who it is for

- **Segment:** companies up to roughly 20–30 people
- **First pilot:** a small services company inside the target size range, running its own
  *internal* desk — their own staff reporting and resolving. Who they are, and the shape of
  their client relationships, is in `docs/private/01-pilot-and-commercial.md` (not in git).
- **Second wave (if the pilot works):** similar small firms, reached through direct
  introductions rather than through marketing.

The competitive position is deliberately narrow: Zendesk, Jira Service Management and
the MSP tools (ConnectWise, Atera, Syncro, HaloPSA) are all heavy, expensive and built
for larger operations. FixFlow's wedge is **small, cheap, and installable in an
afternoon**. Feature parity with any of them is neither achievable nor desirable.

## The two shapes of the same product

| | Reporter | Resolver | Status |
|---|---|---|---|
| **Case A — internal desk** | employee of company X | staff of company X | **This is what we build** |
| **Case B — cross-company desk** | employee of a client company | staff of the desk's company | Possible future, not built |

Both are the same core with one difference: whether the requester belongs to the same
company as the desk. The domain is modelled so Case B is an additive change later
(nullable client-company on a ticket), not a rewrite. **We do not build Case B now.**

## Goals

1. A genuinely production-grade system, exposed to real users, that survives daily use
2. A portfolio piece that demonstrates production engineering, not coursework
3. Side income if it turns out anyone will pay — opportunistic, not the driver

## Non-goals

- Competing on features with established helpdesk products
- Self-serve SaaS signup, billing, subscriptions
- Marketing site, content, SEO
- Anything that requires the author to do sales

## Roles in the project

- **Developer / owner:** full technical responsibility, all decisions
- **Client contact:** a second participant who brings the pilot and any subsequent client
  introductions, and may or may not contribute code. Distribution, not development.

## Definition of "it worked"

The pilot firm uses FixFlow daily for three months without going back to Messenger.
Everything beyond that is upside.
