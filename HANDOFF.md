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
- [ ] Run one real ₹99 test transaction and confirm the correct screen shows.
- [ ] (Optional) Set `LEAD_ENDPOINT` to start storing registrations.
