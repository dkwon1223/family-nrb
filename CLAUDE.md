# CLAUDE.md

Project instructions for Claude Code. Keep under 200 lines.

## Project

Marketing site plus booking system plus admin panel for a Korean bar and private-room
karaoke (noraebang) in Aurora, CO. Solo unpaid maintainer. English only.

**Status: greenfield.** No application code exists yet. Full rationale for every decision
below is in `ARCHITECTURE.md` — read it before proposing changes to the stack, hosting,
data model, or payments scope. Do not restate its contents here.

## Writing Style

For chat responses reporting to me, use the following points to guide your writing style. Code comments follow the surrounding code's style instead.

- Be extremely extremely concise. Sacrifice grammar for the sake of concision.
- Second person, present tense, active voice. When reporting your own actions, use first person.
- Sentence case headings.
- Standard American spelling.
- Code font for code, bold for UI elements.
- When a list is warranted, number it for sequences and bullet it otherwise.

## Stack

- Next.js (App Router, TypeScript), pinned to the **Active LTS** line
- Payload 3, installed into the same `/app` folder, generating the admin panel
- PostgreSQL
- TanStack Query, scoped to the booking route only
- Sevalla (app pod, Postgres pod, S3-compatible object storage)
- Resend for transactional email, Sentry for errors

## Commands

Fill these in as they are established. Until then, ask rather than guess.

```bash
pnpm dev                          # local dev server
pnpm build                        # production build
pnpm payload generate:types       # regenerate payload-types.ts after schema changes
pnpm payload migrate:create       # generate a migration; required for production
```

## Data model invariants

These are expensive to retrofit. Do not simplify them "for v1."

- **Reservation status is an enum**: `pending | confirmed | cancelled | completed | no_show`.
  Never a boolean, never a nullable `confirmed_at` standing in for status.
- **Booking is two-phase.** Create the row and hold the slot, then confirm as a separate
  step. Do not write a single synchronous handler that creates a confirmed booking and
  renders success. Phase two is instant today and becomes a webhook later.
- **Slots are discrete 30-minute rows.** A two-hour booking is four rows sharing a
  `booking_group_id`. Do not switch to `tstzrange` with an exclusion constraint; Payload
  has no range field type and the custom migrations are not worth it at this scale.
- **Double-booking is prevented by the database**: a unique constraint on
  `(room_id, time_slot)` inside a serializable transaction. Do not add application-level
  locking, a queue, or a reservation service.
- **Pricing is time-windowed rate rules**, not a price column. Rates vary by room size,
  day of week, and time of day, and happy hour patterns must be expressible as rules.
- **Money is integer cents.** No floats, no decimals-as-strings.
- **Store UTC, render `America/Denver`.** The operating day runs past midnight: a 1am
  Saturday booking belongs to Friday night in every UI surface, including admin lists and
  the booking calendar.
- **The expired-`pending` sweep ships in v1.** It runs in-process on the long-lived Node
  server, not as an external cron trigger.

## Payments: out of scope for v1

v1 ships with no online payments, by decision. The no-show rate is unmeasured.

**Do not add**, even if it looks like an obvious improvement: a Stripe dependency, a
webhook route, a `payments` table, a `payment_status` column, or a payment-provider
abstraction layer. These are additive migrations later against a schema that already
accommodates them.

If asked to add payments, implement Stripe Checkout (hosted) per `ARCHITECTURE.md` §6.
The non-negotiables there: the webhook is the source of truth and the success redirect is
UX only; verify signatures against the raw request body, not parsed JSON; deduplicate on
`event.id`.

## Rendering and client JavaScript

The site's commercial value is ranking for local karaoke searches and loading fast on a
phone. Protect that.

- Public pages are Server Components. Default to no client JavaScript.
- `"use client"` is confined to the booking calendar, the booking form, and the mobile nav.
  Adding it anywhere else needs a stated reason.
- Read Payload content through the **Local API** (`payload.find(...)`) inside Server
  Components. Do not fetch your own REST endpoints over HTTP from server code.
- TanStack Query belongs only under the booking route. Mount `QueryClientProvider` in that
  route's layout, never the root layout.
- Images go through `next/image` with `sharp` installed. Optimization burns the app pod's
  own CPU, so unoptimized hero photos are the single biggest page-speed risk.

## Booking calendar

- The availability endpoint returns **taken** intervals (status `pending` or `confirmed`)
  for a room and date range. The client derives free slots from operating hours.
- Server-render the first visible week and hydrate with `HydrationBoundary`.
- Query config: `staleTime` ~30s, `placeholderData: keepPreviousData`, leave
  `refetchOnWindowFocus` on.
- **Cached availability is optimistic.** The booking mutation must handle a constraint
  rejection by invalidating `['availability', roomId]`, refetching, and telling the user
  the slot was taken. This path will fire on Saturday nights; do not leave it as a generic
  error toast.

## Payload conventions

- `payload-types.ts` is generated output. Never hand-edit it; run `generate:types`.
- Dev mode auto-pushes schema changes. Production requires generated migrations. Do not
  deploy a schema change without one.
- Media uses the S3 storage adapter against Sevalla object storage. The local-filesystem
  default does not survive redeploys — never fall back to it, even temporarily.
- Collections are many rows (rooms, reservations, menu items). Globals are singletons
  (business hours, site settings). Put owner-editable content in one of these, not in
  hardcoded JSX.

## Deployment

- Target is a standard Next.js standalone container. Write no Sevalla-specific code; the
  app must stay movable to Railway, Fly, or a VPS with a Dockerfile and an env file.
- Do not enable hibernation on the production pod.
- The database pod has 0.25GB RAM. Payload's schema push can be memory-hungry — flag
  migrations that may thrash it.

## Scope guard

Before adding a dependency, service, or abstraction, check it against the maintenance
budget: 1–2 hours per month from one unpaid engineer. Say so explicitly when a suggestion
would raise that. Specifically, do not introduce Kubernetes, a message queue, a second
deployable, a microservice split, SMS notifications (A2P 10DLC registration overhead), or
Korean localization. All were considered and rejected.

## Open questions

Blocking schema work. Ask the owner rather than inventing values: actual room types and
capacities, current rate structure, any existing happy hour pattern, and the current
no-show rate (which gates the payments work).
