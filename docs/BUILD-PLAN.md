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
  - Background geofencing: `expo-location` + `expo-task-manager`
  - Push: `expo-notifications` (FCM + APNs under the hood)
- **Backend + data:** **Supabase** (managed Postgres, Auth, Row-Level Security,
  Realtime, Edge Functions). Moves us fast without hand-rolling auth/infra.
  - Alternative if we outgrow it: Node + TypeScript API on Postgres.
- **Web (marketing + venue onboarding + investor demo):** the current
  `index.html` grows into a small static/Next.js site.
- **Payments / spend capture (later):** POS integrations (Square, Toast) and/or
  Stripe for membership billing.

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
host_device   staff_id, push_token, platform
```

**Tier engine:** tier = f(rolling 12-month spend + tips). Recompute on each new
transaction. Tiers (draft): Silver → Gold → Platinum → Obsidian, with thresholds
we tune. Progress bar in the app reads from this.

---

## 4. The recognition engine (the hard, important part)

This is what makes the app magic — and the part that most needs care.

**Flow**
1. Member opts in to location sharing (granular: off / only near favorites / all).
2. Member app registers **geofences** around eligible venues (favorites +
   approved venues within their city).
3. On **geofence entry**, the app sends an event to the backend
   (member id + venue id + timestamp — *not* a continuous location stream).
4. Backend checks: does this member have a relationship with this venue? Is the
   venue a live partner? Is a host on duty?
5. If yes, push to on-duty host devices with a deep link to the member profile.
6. Host taps → profile opens → greeting / table actions fire notifications to
   the member and the relevant staffer.

**Design principles**
- **Entry events, not tracking.** We react to arriving at a known venue; we do
  not follow members around the city.
- **Relationship-gated.** A venue only ever sees members who favorited it,
  were approved there, or explicitly opted into "all venues."
- **On-duty only.** Profiles surface to staff who are clocked in, and only for
  the current window.

---

## 5. Privacy & trust (non-negotiable, built in from day one)

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

## 6. Phases & milestones

**Phase 0 — Prototype** ✅ *(done)*
Interactive front-end of both views, sample data. Use it to sell the concept to
early venues and validate the flow.

**Phase 1 — Foundations**
Backend + auth + data model, in **one app with a member/host role toggle**
(decided). Member app reads *real* data: profile, tier card, spend/tips,
favorites. **Spend is logged by venue staff after the member leaves** (decided) —
so a minimal staff "log a visit" screen (member + amount + tip) ships in Phase 1
alongside the member view, since spend can't appear until staff can enter it.
*Milestone: a real member logs in and sees standing that a venue actually
entered.*

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

## 7. Decisions & open questions

**Decided**
1. **One app, member/host role toggle.** A single app switches between the
   member view and the venue/host view by role, until the host side is big
   enough to justify its own app.
2. **Spend is logged manually by venue staff after the member leaves.** No POS
   integration at launch. Implication: staff need a lightweight "log a visit"
   screen (member + amount + tip) in Phase 1, and the tier engine recomputes
   from those entries. POS auto-capture stays a Phase 5 upgrade.

**Still open (can settle as we go)**
3. **Tier thresholds** — actual dollar figures for Silver → Obsidian.
4. **Launch market** — one city / a handful of venues to start.
5. **Revenue model** — membership fee, venue subscription, per-recognition, or
   commission on brokered access? Shapes what we instrument.

---

*Phase 1 is unblocked. Remaining open items (3–5) don't block the first code
step and can be decided while Phase 1 is built.*
