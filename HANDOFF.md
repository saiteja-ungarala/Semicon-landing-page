# Semicon Labs — Landing Page Handoff

A **single, static** file: `index.html` (all CSS + JS inline, assets in `images/`).
No build step, no framework, no dependencies. Host it as-is on any static host.

This note is for the **hosting + payment** team. The page's UI is complete —
you only need to (1) deploy it and (2) connect the real ₹99 charge.

---

## 1. What the registration form does

The form (`#regForm`, near the "Reserve your spot" section) has two intents,
chosen by the toggle at the top of the form:

| Intent | Button | What should happen |
|--------|--------|--------------------|
| `pay`    | **Register ₹99** | Charge ₹99, then show the paid-success or payment-failed screen |
| `notify` | **Just notify me** | No charge — instantly shows the "subscribed for updates" screen |

On submit the page validates name / email / phone, then:
- `notify` → `showStatus('subscribed')`
- `pay` → `startPayment(lead)`

## 2. The three outcome screens (already built)

All rendered by `showStatus(key)`; text/icons are final and approved.

| key | When to show it | Screen |
|-----|-----------------|--------|
| `success`    | ₹99 payment **completed** | "Registration Successful! 🎉" |
| `failed`     | Payment **abandoned / failed** | "Payment Not Completed" + **Retry Payment** button |
| `subscribed` | Notify-only submit | "You're Subscribed for Updates!" |
| `pending`    | (fallback) pay chosen but no link configured yet | "Details saved…" |

You do **not** need to touch these — just make sure the right one fires.

## 3. Wiring the ₹99 payment — pick ONE option

Both options are set up at the **top of the `<script>` block** in `index.html`
(search for `INTEGRATION POINTS`).

### Option A — Hosted Easebuzz payment link (simplest)
1. Set `EASEBUZZ_PAYMENT_LINK` to your ₹99 Easebuzz link.
2. In the Easebuzz dashboard, set that link's redirect URLs back to this page:
   - **Success:** `https://<your-domain>/?pay=success`
   - **Failure:** `https://<your-domain>/?pay=failed`
3. Done. On the pay path the page redirects to Easebuzz; on return it reads the
   `?pay=` param and shows `success` or `failed` automatically.
   (It also accepts `?status=` / `?payment=` and common values like
   `success/completed/paid` and `failed/cancelled/pending`.)

> Note: if Easebuzz sends the result as an HTTP **POST** (not a GET redirect),
> a static page can't read it — in that case use Option B with a small backend
> endpoint that receives the POST and 302-redirects to `/?pay=success|failed`.

### Option B — Your own checkout / SDK / backend
Replace the body of `startPayment(lead)` with your flow (Easebuzz seamless SDK,
your API, etc.). When the result is known, call:
```js
showStatus('success');   // payment captured
showStatus('failed');    // payment not completed  (shows Retry)
```
`lead` contains `{ name, email, phone, intent, ts }`. It's also saved to
`localStorage.semiconLead` so it survives a redirect round-trip.

## 4. Lead capture (optional, currently OFF)

`LEAD_ENDPOINT` is blank, so leads only `console.log`. To store them, set it to
a POST URL (your API, Formspree, Google Apps Script, …). Payload is JSON:
```json
{ "name": "...", "email": "...", "phone": "...", "intent": "pay|notify", "ts": 1730000000000 }
```

## 5. Deploy checklist
- [ ] Serve `index.html` + `images/` on the chosen host / domain.
- [ ] Set `EASEBUZZ_PAYMENT_LINK` (Option A) **or** implement `startPayment` (Option B).
- [ ] If Option A: configure Easebuzz success/failure redirect URLs (step 3).
- [ ] **Verify every payment server-side before granting benefits** (see §6).
- [ ] Test: complete a real ₹99 payment → "Registration Successful".
- [ ] Test: start payment, cancel/return without paying → "Payment Not Completed" + Retry (NOT success).
- [ ] Test: retry after a failed/abandoned attempt with the same email → allowed (NOT "already enrolled").
- [ ] (Optional) Set `LEAD_ENDPOINT` to start storing registrations.

---

## 6. Correct behaviour & common mistakes (READ THIS FIRST)

Two real bugs were seen in a previous integration. Both are on this list — do not repeat them.

**Never show success or mark a user "enrolled" until the ₹99 payment is confirmed.**
- `showStatus('success')` renders a *screen only* — it is not proof of payment. Fire it strictly after Easebuzz confirms the charge was captured.
- `?pay=success` in the URL is client-controllable: anyone can type, refresh, bookmark, or share `/?pay=success`. Treat the `?pay=` param as a display **hint**, never as authoritative for money or enrollment. (The template already blocks the trivial case — it only honours `?pay=success` when a payment actually began in the same tab, via `sessionStorage 'semiconPayStarted'` — but that is **not** a substitute for server verification.)
- Verify every payment server-side (Easebuzz webhook or verify/status API, keyed on the transaction/order id) before recording anyone as paid.
- **Bug this prevents:** a user who returned without paying (back button / refresh / shared link) was shown the success screen because the URL param was trusted as truth.

**A lead / pre-registration is NOT a paid enrollment — store them as separate states.**
- `LEAD_ENDPOINT` (§4), the `notify` intent, and `localStorage.semiconLead` all capture *leads*. A lead is written on form submit, before and independent of any payment — it means "form filled in," not "paid."
- Keep `lead` and `paid` as distinct records/statuses. Never promote a lead to "enrolled" without a confirmed payment.
- **Bug this prevents:** the backend recorded the captured lead as "enrolled," so it believed an unpaid user had already paid.

**Unpaid users MUST be able to (re)start payment.**
- The `failed` screen's **Retry Payment** button re-runs the pay path for the same lead — abandoned/failed payments are normal and retry is expected.
- Any duplicate-email / "already registered" check must key on *confirmed payment status* (`status='paid'`), not on the existence of a lead or email. An email with a lead but no confirmed payment must always be allowed to start or retry payment.
- On a repeat submit for a non-paid user, upsert the existing lead and start a **fresh** payment attempt (new txnid) — don't error out.
- **Bug this prevents:** the backend blocked retry with "user already enrolled" for someone who had only submitted the form (a lead) and never paid.

**How success/failure must be signalled back to this static page.**
- The page learns the outcome only via `showStatus('success' | 'failed')`, driven by the `?pay=` return param (Option A) or your explicit call (Option B).
- Do the authoritative "mark as paid / enrolled" write in your **server-side** verification step — never on the client and never from the redirect param alone.
- Option A when Easebuzz returns a POST (see §3 note): verify the payment in your endpoint first, then 302-redirect to `/?pay=success` only on a *verified* success, otherwise `/?pay=failed`.

### Suggested data model
```
Lead        { id, name, email, phone, intent, createdAt }              // written on submit
PaymentAttempt { id, leadId, txnid, status: pending|paid|failed, amount, ... }
```
Block as "already enrolled" only when a `PaymentAttempt` with `status='paid'` exists for that email/phone. Everything else → allow a new attempt.
