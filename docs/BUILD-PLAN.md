# My VIP Clubs — Build Plan

A phased plan to take My VIP Clubs from prototype to a real, two-sided app:
a **member app** and a **venue host console**, connected by a
**recognition engine** that greets VIPs before they reach the door.

---

## 1. The product in one paragraph

A member opens the app and sees their standing — tier, lifetime spend, tips,
favorite venues, and the staff who take care of them — and can request access
to rooms they can't get into alone. When that member nears a partner venue,
the venue's host gets a push: *"Obsidian member, 6 min out."* The host taps it,
sees the member's name and tastes, and the member walks past the line into a
drink already being made the way they like it.

Two audiences, one loop:

| Member app | Venue host console |
|---|---|
| Tier & lifetime value | Incoming "VIP nearby" alerts |
| Favorite venues & people | Member profile on tap (tastes, usual table) |
| Request access to venues | Confirm greeting / ready the table |
| Notifications & history | Roster of tonight's expected VIPs |

---

## 2. Recommended stack

Chosen to let a small team ship fast while keeping the hard parts (background
location, cross-platform push) well-supported.

- **Mobile (both apps):** React Native via **Expo**. One codebase, iOS + Android.
  Two apps (or one app with role switching early on).
- **Push notifications + geofencing:** **Pushwoosh** (decided). Its **Geo Zones**
  feature does the geofence detection on-device and its push delivery handles
  both member and host devices, so we don't hand-roll background location.
  Pushwoosh SDK drops into the Expo/React Native app.
- **Backend + data:** **Supabase** (managed Postgres, Auth, Row-Level Security,
  Realtime, Edge Functions). Moves us fast without hand-rolling auth/infra. Hosts
  the **application API** below.
  - Alternative if we outgrow it: Node + TypeScript API on Postgres.
- **Payments:** **Stripe** for the **$25 VIP Express processing fee** at launch
  (and, later, membership billing). The Standard application is free.
- **Web (application + marketing + venue onboarding + demo):** the current
  `index.html` grows into a small static/Next.js site. **A live application flow
  already exists** at `myvipclubs.brandecho.chatgpt.site/apply` — a React/Vite SPA
  with a 4-step wizard (Choose → Apply → Review & fee → Confirmation). Both
  applications are **JotForms**: Standard is the existing full form
  (`form.jotform.com/262304451128045`); VIP Express is a **duplicated, shortened**
  copy reached after the $25 Stripe payment. A JotForm webhook feeds both into the
  application API below.

> The current prototype is intentionally backend-free. Nothing below throws it
> away — the same screens get wired to real data.

---

## 3. Data model (first cut)

```
member        id, name, phone, email, home_city, joined_at,
              location_consent (enum: off / on_near_favorites / on_all),
              tier (derived), photo
venue         id, name, address, geo (lat/lng), radius_m, status, tier_min
staff         id, venue_id, name, role (bartender/host/server/manager), on_duty
membership    member_id, venue_id, status (favorite/requested/approved), since
transaction   id, member_id, venue_id, staff_id, amount, tip, occurred_at
favorite_staff member_id, staff_id
taste         member_id, kind (drink/table/music/avoid), value
access_request id, member_id, venue_id, status, note, created_at
geofence_event id, member_id, venue_id, kind (enter/dwell), at
notification  id, to (staff/member), type, payload, sent_at, opened_at
device        owner_id, owner_kind (member/staff), pushwoosh_hwid, push_token, platform
application   id, applicant_email, type (standard/express), answers (jsonb),
              fee_status (na/unpaid/paid), stripe_payment_id,   -- fee only for express
              jotform_id, jotform_submission_id, stripe_session_id,   -- traceability
              status (submitted/under_review/approved/rejected),
              member_id (set once approved), created_at
```

**Application → profile:** an application comes in (Standard = full/free,
Express = short + $25), lands as an `application` row, and on approval is promoted
into a real `member` profile (its answers seed the member's name, city, tastes,
etc.). Device push tokens are stored per Pushwoosh `hwid`.

**Tier engine:** tier = f(rolling 12-month spend + tips). Recompute on each new
transaction. Tiers (draft): Silver → Gold → Platinum → Obsidian, with thresholds
we tune. Progress bar in the app reads from this.

---

## 4. Application flow & API

Membership starts with an application. A live 4-step wizard already exists at
`myvipclubs.brandecho.chatgpt.site/apply` (Choose → Apply → Review & fee →
Confirmation). It needs a backend to receive, store, and act on submissions.

**Two application types — both JotForms (decided)**
- **Standard Membership Application** — **free ($0)**, the *full* application
  (complete profile, VIP preferences, referral/qualification info upfront). The
  existing JotForm (`form 262304451128045`).
- **VIP Express Application** — **$25 non-refundable processing fee**, the *short*
  path (~60 seconds, essentials only, concierge follow-up after). Built by
  **duplicating the Standard JotForm and trimming it to the essentials.** The fee
  does **not** guarantee approval.

> The naming is worth noting: the **short** form is the paid one, the **full**
> form is free. You pay for speed and white-glove follow-up, not for the effort.

**The flows (both end at a JotForm → our webhook)**
- **Standard:** pick Standard → straight to the full JotForm → submit.
- **VIP Express:** pick Express → **pay $25** → on success, redirect to the
  **short JotForm** → submit.

This keeps *all* form-building in JotForm — no custom form to maintain. The only
custom piece is the payment gate in front of the Express JotForm.

**Payment gate (simplest that works)**
- Use a **Stripe Payment Link / Checkout** for the $25 fee whose **success URL is
  the short JotForm**. Pass a reference (the Stripe session id) into the JotForm
  as a **prefilled field** so the submission carries proof of payment.
- The JotForm **webhook** then posts the submission (including that reference) to
  our API, which verifies the payment with Stripe before marking the Express
  application `paid` + `submitted`. This stops anyone reaching the short form URL
  without paying.

**Endpoints (Supabase Edge Functions / REST)**
```
POST /pay/express               create Stripe Checkout for the $25 fee
                                (success_url = short JotForm, prefilled w/ session id)
POST /webhooks/stripe           record a paid Express session
POST /webhooks/jotform          ingest a submission (standard or express) → application row;
                                for express, verify the Stripe session before accepting
GET  /applications/:id          status (submitted / under_review / approved / rejected)
POST /applications/:id/approve  (admin) promote application → member profile
GET  /me                        the app pulls the logged-in member's profile + tier
POST /visits                    staff log a visit (member, amount, tip) → transaction
POST /webhooks/pushwoosh        geo-zone entry events (see §5)
```

**Rules of the road**
- Express is payment-gated: an Express application isn't "submitted" until
  `fee_status = paid`. Standard has no fee (`fee_status = na`).
- Never store raw card data — Stripe holds it; we keep only the payment id
  (keeps us out of PCI scope).
- Approval is the gate that turns an `application` into a `member` and seeds the
  profile from the answers.

---

## 5. The recognition engine (the hard, important part)

This is what makes the app magic — and the part that most needs care.

Built on **Pushwoosh Geo Zones** (geofence detection) + **Pushwoosh push**
(delivery). We add the relationship/consent logic between the two.

**Flow**
1. Member opts in to location sharing (granular: off / only near favorites / all).
2. Each partner venue is a **Pushwoosh Geo Zone**; the member app is subscribed
   to the zones for eligible venues (favorites + approved venues in their city).
3. Pushwoosh detects **zone entry** on the member's device and calls our backend
   webhook (member id + venue id + timestamp — *not* a continuous location stream).
4. Backend checks: does this member have a relationship with this venue? Is the
   venue a live partner? Is a host on duty?
5. If yes, backend sends a **Pushwoosh push to on-duty host devices** with a deep
   link to the member profile.
6. Host taps → profile opens → greeting / table actions fire notifications
   (also via Pushwoosh) to the member and the relevant staffer.

> Note: Pushwoosh's built-in geo push would notify the *member* who entered the
> zone. Our twist is that entry must alert the *host* — so we route the entry
> event through our backend and fan out to host devices, rather than using a
> plain member-facing geo push.

**Design principles**
- **Entry events, not tracking.** We react to arriving at a known venue; we do
  not follow members around the city.
- **Relationship-gated.** A venue only ever sees members who favorited it,
  were approved there, or explicitly opted into "all venues."
- **On-duty only.** Profiles surface to staff who are clocked in, and only for
  the current window.

---

## 6. Privacy & trust (non-negotiable, built in from day one)

Location + identity is sensitive. This has to feel like a concierge, not
surveillance — for members *and* for the venues' liability.

- **Explicit, granular consent** for location sharing, changeable anytime.
- **Data minimization:** store geofence *entry* events, not location history.
- **Member controls what venues see** (tastes are opt-in per field).
- **Staff access controls:** on-duty scope, audit log of profile opens.
- **Right to delete** account + data; clear retention windows.
- **Regional compliance:** location-privacy and consent law (varies by state /
  country); PCI scope if/when we touch card data (prefer POS/Stripe so we don't).
- A short **privacy policy + venue data agreement** before the first real venue.

---

## 7. Phases & milestones

**Phase 0 — Prototype** ✅ *(done)*
Interactive front-end of both views, sample data. Use it to sell the concept to
early venues and validate the flow.

**Phase 1 — Foundations + application intake**
Backend + auth + data model, in **one app with a member/host role toggle**
(decided). Scope:
- **Application API** (Supabase): a **JotForm webhook** ingests both the Standard
  and (payment-gated) VIP Express submissions, plus a **Stripe Checkout** for the
  $25 Express fee whose success URL is the short JotForm. Stores each as an
  `application` row; on approval promotes it into a `member` profile.
- Member app reads *real* data: profile, tier card, spend/tips, favorites.
- **Spend is logged by venue staff after the member leaves** (decided) — so a
  minimal staff "log a visit" screen (member + amount + tip) ships here too,
  since spend can't appear until staff can enter it.

*Milestone: someone applies + pays on the web, gets approved into a member
profile, logs in, and sees standing that a venue actually entered.*

**Phase 2 — Venue & staff side**
Full venue onboarding, staff accounts & roles, the richer host console, linking
members↔venues, access-request workflow. Builds out the staff side beyond the
Phase 1 spend-logging screen. *Milestone: a host can pull up a member profile.*

**Phase 3 — The recognition engine**
Geofencing + push + consent controls. *Milestone: walk near a partner venue →
the host gets the alert → taps → sees the profile.* This is the demo that sells.

**Phase 4 — Concierge & access**
"Request access" becomes a real workflow (member asks → we broker → approval →
venue added). Notifications, visit history.

**Phase 5 — Spend automation & scale**
POS integrations to auto-capture spend & tips, membership billing, analytics for
venues (VIP traffic, value), multi-city.

---

## 8. Decisions & open questions

**Decided**
1. **One app, member/host role toggle.** A single app switches between the
   member view and the venue/host view by role, until the host side is big
   enough to justify its own app.
2. **Spend is logged manually by venue staff after the member leaves.** No POS
   integration at launch. Implication: staff need a lightweight "log a visit"
   screen (member + amount + tip) in Phase 1, and the tier engine recomputes
   from those entries. POS auto-capture stays a Phase 5 upgrade.
3. **Two application types, both JotForms.** **Standard** = full profile, **free**
   (`form 262304451128045`). **VIP Express** = a **duplicated, shortened** copy of
   that JotForm, reached only after paying the **$25 fee via Stripe** (Checkout
   success URL → the short form). A JotForm webhook feeds both into our API;
   approval promotes them into a member profile.
4. **Pushwoosh** for push notifications *and* geofencing (Geo Zones). Replaces the
   earlier build-it-ourselves approach for background location.

**Still open (can settle as we go)**
5. **Tier thresholds** — actual dollar figures for Silver → Obsidian.
6. **Launch market** — one city / a handful of venues to start.
7. **Revenue model beyond the $25 application fee** — recurring membership, venue
   subscription, per-recognition, or commission on brokered access?

---

*Phase 1 is unblocked. Remaining open items (5–7) don't block the first code
step and can be decided while Phase 1 is built.*
