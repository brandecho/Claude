# My VIP Clubs

Give members the hookup — access to rooms they can't get into alone, recognition
before they reach the door, and service tuned to exactly how they like it.

This repo currently holds an **interactive front-end prototype** (`index.html`,
a single self-contained file — no backend, sample data only) that demonstrates
the core concept end to end.

## What the prototype shows

**Member view**
- Membership **tier card** (Obsidian) with member number and tier progress
- **Lifetime spend**, **total tips**, nights out, doors skipped
- **Favorite venues** and the **people who take care of you** (bartenders, hosts, servers)
- **Get the hookup** — browse invite-only rooms and request access; My VIP Clubs makes the call

**Venue Host view**
- A live **GPS "VIP nearby" push** — fires when a member enters the venue's area
- Tap it to open the member's **profile**: tier, ETA, visit history, and *how they like it*
  (drink, table, music, what to avoid) plus the staff member who usually takes care of them
- One tap to **greet at the door** or **ready their table**

Open `index.html` in any browser, or view the hosted version, then use the
**Member / Venue Host** toggle at the top of the phone.

## Roadmap (next steps)

1. Real data model + backend for members, venues, staff, spend/tips ledger
2. Auth for the two roles (member app vs. venue host console)
3. Actual geofencing + push notifications for the nearby-VIP alert
4. Venue onboarding and the access-request workflow
