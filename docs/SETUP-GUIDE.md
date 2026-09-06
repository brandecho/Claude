# My VIP Clubs — Application Setup Guide

The tasks **you** can do yourself (no code) to get the paid-application flow
ready. Do these and the backend has everything it needs to plug in.

Related: see `BUILD-PLAN.md` §4 for how these pieces fit together.

- **Standard application (free):** existing JotForm →
  https://form.jotform.com/262304451128045  (form id `262304451128045`)
- **VIP Express application ($25):** a shortened copy of that form, reached only
  after payment.

---

## Step 1 — Create a Stripe account

1. Go to https://stripe.com and sign up (use the Brand Echo business email).
2. Fill in business details so you can eventually accept live payments. You can
   build and test everything in **Test mode** first — no bank details needed to
   start.
3. Later, before going live, complete Stripe's business verification.

*Why Stripe:* it supports a hosted **Payment Link / Checkout** with a
**success-redirect URL** and can pass a reference into that URL — which is how we
send a paid applicant straight to the short JotForm and prove they paid.

> If you'd rather use PayPal or Square, tell me — the flow still works, the
> "redirect with a payment reference" wiring just differs. Stripe is the cleanest.

---

## Step 2 — Duplicate the JotForm and make the short version

1. Log into the JotForm account that owns `262304451128045`.
   *(Tell me which email that account is under — I'll need it to enable the
   webhook in Step 4.)*
2. **My Forms → that form → ⋯ menu → Clone Form.**
3. Rename the clone to **"VIP Express Application"**.
4. **Trim it to the ~60-second essentials.** Keep only what you truly need
   upfront; the rest is gathered later via concierge follow-up. Suggested keep:
   - Full name
   - Email
   - Mobile phone
   - City / primary market
   - (optional) How they heard about us / referral
5. **Add one hidden field** named `payment_ref` (Form Elements → hidden field).
   The Stripe redirect fills this so we can verify payment. Leave it empty.
6. Publish and copy the clone's **form URL** and **form id** — send both to me.

*I can't see the current form's fields from here, so if you paste me the field
list (or a screenshot), I'll tell you exactly which to keep vs. drop.*

---

## Step 3 — Create the $25 payment and connect it to the short form

1. In Stripe: **Products → add a product** "VIP Express Application Fee", price
   **$25.00**, one-time.
2. Create a **Payment Link** for it.
3. Under the Payment Link's **"After payment"** setting, choose **"Redirect to a
   page"** and set the URL to your **short JotForm**, appending the session id:
   ```
   https://form.jotform.com/<EXPRESS_FORM_ID>?payment_ref={CHECKOUT_SESSION_ID}
   ```
   Stripe substitutes `{CHECKOUT_SESSION_ID}` automatically, and JotForm reads
   `?payment_ref=...` into the hidden field from Step 2.
4. On the website's **"Apply in 60 Seconds"** button, point it at this Stripe
   Payment Link (replacing the current in-app step).

*Result:* pick Express → pay $25 → land on the short form with proof of payment
attached. No one can reach the short form without paying, because the backend
(Step 4) checks the `payment_ref` against Stripe.

---

## Step 4 — Connect submissions to the backend *(I do this)*

Once Steps 1–3 are done and I have the account details, I'll:
- Turn on a **JotForm webhook** on both forms → our application API.
- Add a **Stripe webhook** so we record paid sessions.
- Verify each Express submission's `payment_ref` against Stripe before accepting.
- Store every submission as an `application` row and route it to review/approval.

---

## What to send me when you're done

- [ ] JotForm account **email** (owner of the forms)
- [ ] **VIP Express** form URL + form id (the clone)
- [ ] Confirmation the Stripe account exists (test mode is fine to start)
- [ ] The **field list** you kept on the short form (so the API maps them right)
- [ ] Whether you're sticking with **Stripe** (or want PayPal/Square instead)
