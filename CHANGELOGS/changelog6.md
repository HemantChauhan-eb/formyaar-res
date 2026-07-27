# FormYaar — Changelog 6
**Session date: May 10–11, 2026**

---

## PAN Card Config Completion

### Steps 8–11 mapped (pan_card.json v5)

Completed the remaining NSDL PAN card flow after the payment gateway:

- **Step 8 — `ddSave.html` (Pay Confirm):** Decided NOT to auto-click payment button. Added step with `guidance_only: true` flag so logic is preserved for future overlay implementation but currently just shows verify screen. Selector: `#payConfirm`.

- **Step 9 — `paytmResponseAfterStatus.html` (Payment Receipt):** Auto-clicks Continue after successful Paytm payment. Selector: `#continuePayment`.

- **Step 10 — `continuePayment.html` (Aadhaar Authentication):** Ticks consent checkbox (`#consent`) and clicks Authenticate button (`#submitAadhaarFullForm`). User then enters OTP manually on Aadhaar-linked mobile.

- **Step 11 — `proceedAadhaarDetails.html` (Continue with e-KYC):** Clicks "Continue with e-KYC" button (`#submitEsign`).

- **`authenticateOTP.html` and final eSign page:** User-only. No config steps. Likely opens in new window so extension panel won't be active anyway.

**Skipped pages (no config):**
- `responseAPIRequest.html` — NSDL internal redirect/waiting page
- `secure.paytmpaymens.com` — External Paytm gateway, different domain

**Bumped version:** `pan_card.json` version 4 → 5. Pushed to backend repo, Railway auto-deployed.

**Full flow summary:**

| Step | Page | FormYaar | User |
|------|------|----------|------|
| 1 | `endUserRegisterContact` | Autofills all fields | — |
| 2 | Token page | Selects token, clicks Use | — |
| 3–5 | `endUserLogin` stepy 1–3 | Autofills all fields + AO code | — |
| 6 | `endUserLogin` stepy 4–5 | Autofills proofs + place | — |
| 7 | `fullFormSave` | Fills Aadhaar first 8, clicks Proceed | — |
| 8 | `fullFormSubmit` | Clicks agree, proceeds to payment | — |
| 9 | `ddSave` | Shows verify screen (guidance only) | Clicks Pay Confirm |
| 10 | `paytmResponseAfterStatus` | Clicks Continue | Pays on Paytm |
| 11 | `continuePayment` | Ticks consent, clicks Authenticate | — |
| 12 | `proceedAadhaarDetails` | Clicks Continue with e-KYC | — |
| 13 | `authenticateOTP` | Nothing | Enters OTP, submits |
| 14 | Final eSign | Nothing | Completes |

---

## Architecture Decisions

### B2B Cafe Operator Flow — Finalized

Full architecture locked:

- **Extension panel = action** (queue, review, accept, autofill)
- **Web dashboard = intelligence** (`formyaar.pages.dev/dashboard` — stats, history, QR codes)
- **No separate device needed** — operator reviews on same PC, customer fills on their phone

### Pricing confirmed
- B2C: ₹29/form
- B2B: ₹299/month/device

### Data privacy policy finalized
**Stored:** name, mobile, email, DOB, father/mother name, city, state, PIN code, income source, defence status, proof_of_dob type

**Never stored:** Aadhaar number, PAN number, passport number, TIN, any govt ID numbers

**Lifecycle:** pending → filling → completed → deleted 7 days after `completed_at` via pg_cron. Rejected/failed deleted after 30 days.

### QR URL structure
Each operator gets: `formyaar.pages.dev/user-form?op=OPERATOR_UUID`
UUID = Supabase `operators.id`. Baked into QR once, printed/displayed at cafe.

---

## Supabase Setup

**Project:** `wkubrgktujihesjjxyrk`
**URL:** `https://wkubrgktujihesjjxyrk.supabase.co`

### Schema created
```sql
-- operators table
id uuid, email text, created_at, subscription_status, subscription_expires_at

-- submissions table  
id uuid, operator_id (FK), form_type, status, name, mobile, email, dob,
father_name, mother_name, city, state, pincode, income_source, defence,
proof_of_dob, completed_at, created_at
```

### RLS policies
- Operators see only their own row
- Submissions scoped to operator_id
- Anonymous inserts allowed (customer mobile form — no auth)

### pg_cron cleanup job
Runs daily at 02:00 — deletes completed submissions older than 7 days, rejected older than 30 days.

### Google OAuth configured
- Client ID + Secret added to Supabase Auth → Providers → Google
- Callback URL: `https://wkubrgktujihesjjxyrk.supabase.co/auth/v1/callback`
- Site URL: `https://formyaar.pages.dev`
- Redirect URLs: `https://formyaar.pages.dev/*` and `chrome-extension://*/*`

---

## Extension Changes

### New file: `entrypoints/content/supabase.ts`
- Exports `supabase` client, `getOperatorSession()`, `signInWithGoogle()`, `signOut()`
- Tried three approaches for sign-in:
  1. Direct `signInWithOAuth` — blocked by Google (content script context)
  2. `window.open(data.url, "_blank")` — still blocked
  3. `browser.runtime.sendMessage({ type: "OPEN_URL" })` → background opens tab — still blocked
- Root cause: Google treats all Chrome extension contexts as untrusted, regardless of how tab is opened

### New operator screens in `panel.ts`
Three new render functions added to `renderPanelHTML()`:
- `renderOperatorLoginScreen()` — Google sign-in button, shown when no session
- `renderOperatorQueueScreen()` — live queue of pending submissions with form icons
- `renderOperatorReviewScreen()` — full customer data display with Accept & Fill / Reject buttons

### New exports in `panel.ts`
- `showOperatorPanel()` — checks session, shows login or queue
- `loadQueue(operatorId)` — fetches pending submissions from Supabase, renders tiles
- `showReviewScreen(sub)` — displays submission data, wires Accept/Reject buttons
- Temporary "Cafe operator? Sign in here" link added to home screen footer

### `autofill.ts` additions
- `runAutofillFromSubmission(sub)` — maps Supabase submission record to UserData format, sets `window.__fy_operator_userdata`, calls `runAutofill()`
- `guidance_only` check in `runAutofill()` — if step has this flag, show verify screen and return instead of filling

### `userData.ts` change
- `getUserData()` checks `window.__fy_operator_userdata` first — operator override mechanism for B2B autofill

### `background.ts` addition
- `OPEN_URL` message handler added — calls `browser.tabs.create({ url })` from trusted background context

### `types.ts` addition
- `{ type: "OPEN_URL"; url: string }` added to `ExtensionMessage` union

### Supabase Realtime subscription
Added to `loadQueue()` — subscribes to `postgres_changes` INSERT events on submissions table filtered by `operator_id`. Queue auto-refreshes when new customer submission arrives. Unsubscribes before re-subscribing to prevent duplicate channels.

---

## Cloudflare Pages Changes

### New file: `functions/api/submit.js`
Pages Function at `/api/submit`. Receives customer mobile form POST, validates required fields, writes to Supabase submissions table via REST API with anon key. Returns `{ success, id }`.

### New file: `operator-auth-callback.html`
Handles Google OAuth redirect. Imports Supabase JS via CDN, calls `exchangeCodeForSession()`, upserts operator record into `operators` table, closes tab.

### Updated: `user-form.html` (renamed from `formyaar-cafe.html`)
- Submit handler made `async`
- Reads `op` param from URL: `new URLSearchParams(window.location.search).get('op')`
- POSTs to `/api/submit` with full payload
- Shows reference ID from Supabase response: `FY-` + first 8 chars of UUID
- Button disables during submission, re-enables on error
- Fixed: duplicate `return` statement removed
- Fixed: removed stale `// TODO` comment

---

## Known Issues / Pending

### Google OAuth blocking (UNRESOLVED)
Google OAuth consistently fails with "Couldn't sign you in — This browser or app may not be secure" regardless of how the tab is opened from the extension. Suspected causes:
1. Chrome extension context treated as untrusted by Google
2. Current Chrome profile may be flagged (user reported CAPTCHA issues on regular Google searches)

**Options to resolve:**
1. Test on a fresh Chrome profile (2 min test — rules out profile flagging)
2. Build `formyaar.pages.dev/operator-login` — operator signs in on website, gets token, pastes once into extension
3. Use `chrome.identity.launchWebAuthFlow()` — Chrome's own OAuth API that Google explicitly trusts

**Recommended next step:** Try fresh Chrome profile first. If still fails, go with website-based login (Option 2).

### Other pending items
- End-to-end test: mobile form submit → appears in panel queue (blocked by auth)
- Supabase `chrome-extension://*/*` redirect URL may need exact extension ID
- Web dashboard at `formyaar.pages.dev/dashboard` not yet built
- Subscription enforcement (check `subscription_expires_at`) not yet implemented
- Razorpay subscription flow for ₹299/month not yet connected
- QR code generation per operator not yet built (manual URL construction for now)

---

## Key File Paths
- `formyaar-extension/entrypoints/content/supabase.ts` — new
- `formyaar-extension/entrypoints/content/panel.ts` — major additions
- `formyaar-extension/entrypoints/content/autofill.ts` — `runAutofillFromSubmission`, `guidance_only` check
- `formyaar-extension/entrypoints/content/userData.ts` — operator override in `getUserData()`
- `formyaar-extension/entrypoints/background.ts` — `OPEN_URL` handler
- `formyaar-extension/entrypoints/content/types.ts` — `OPEN_URL` type
- `formyaar-backend/configs/pan_card.json` — v5, steps 8–11 added
- `formyaar-website/functions/api/submit.js` — new
- `formyaar-website/operator-auth-callback.html` — new
- `formyaar-website/user-form.html` — updated (was formyaar-cafe.html)
