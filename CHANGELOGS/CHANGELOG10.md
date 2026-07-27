# CHANGELOG10 — FormYaar
**Session date:** May 18, 2026  
**Session duration:** ~3 hours (00:00 – 03:00 IST)  
**Maintainer:** Hemant Chauhan

---

## Summary

This session built Sprint 1 of the FormYaar health monitoring system. Three deliverables: real-time Telegram alerts when autofill breaks, a manual HTML selector health checker dashboard, and a context doc for the next session's `/health` endpoint + UptimeRobot setup.

---

## 1. Backend — `formyaar-backend`

### 1.1 New file: `src/utils/telegram.ts`
- Created a `sendTelegramAlert(message: string)` utility using Node's native `https` module (no extra dependency)
- Reads `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` from environment variables
- Sends messages with `parse_mode: "HTML"` for formatting
- Fire-and-forget pattern — errors are caught and logged, never thrown
- Never crashes the server if Telegram is unreachable

### 1.2 Updated: `src/routes/telemetry.ts`
**New critical events added to allowlist:**
- `field_fill_failed` — a CSS selector was not found in the DOM during autofill
- `step_match_failed` — no step in the config matched the current page URL
- `ao_code_failed` — AO code auto-fetch flow threw an error
- `autofill_error` — generic autofill crash

**Alert logic added:**
- `CRITICAL_EVENTS` set — only these four events trigger Telegram alerts
- Deduplication via in-memory `Map<string, timestamp>` — same event+form+selector combo won't alert more than once per 10 minutes
- `shouldAlert(key)` function checks dedup window
- `buildAlertMessage(event, form, metadata)` builds formatted HTML messages per event type with emojis, timestamps in IST, operator ID, selector, step info
- Alert is fire-and-forget in the request path — never `await`ed, never blocks response

**New environment variables required:**
- `TELEGRAM_BOT_TOKEN`
- `TELEGRAM_CHAT_ID`
Both added to Railway `intelligent-energy` service (the active backend service).

**Note on Railway services:** Two backend services were found running — `amiable-integrity` and `intelligent-energy`. The active one serving `formyaar-backend-production.up.railway.app` is `intelligent-energy`. Env vars must be set there, not on `amiable-integrity`.

---

## 2. Extension — `formyaar-extension`

### 2.1 Updated: `entrypoints/content/telemetry.ts`
- **Breaking change from previous pattern:** `trackEvent` no longer does a direct `fetch` to the backend
- Now sends a `browser.runtime.sendMessage({ type: "TELEMETRY_EVENT", payload: { event, form, metadata } })` to the background service worker
- **Reason:** Content scripts run in the government portal's page context. NSDL's CSP (Content Security Policy) blocks outbound fetch requests to `railway.app`. Routing through the background service worker bypasses this — the service worker runs in the extension context, not the page context, and is not subject to the page's CSP
- This is consistent with the existing pattern for `AI_CHAT`, `CREATE_PAYMENT`, and `OPEN_RAZORPAY` — all of which already route through the background for the same reason

### 2.2 Updated: `entrypoints/background.ts`
- Added `TELEMETRY_EVENT` message handler in the `browser.runtime.onMessage` listener
- Receives `message.payload` and forwards it via `fetch` to `${BACKEND}/telemetry/event`
- Uses `(message as any).payload` cast — type-safe version requires the union type update below
- Fire-and-forget `.catch(() => {})` — telemetry must never crash the background worker

### 2.3 Updated: `entrypoints/content/types.ts`
- Added `TELEMETRY_EVENT` to the `ExtensionMessage` union type:
  ```typescript
  | { type: "TELEMETRY_EVENT"; payload: { event: string; form: string; metadata: Record<string, unknown> } }
  ```
- Without this, TypeScript throws `ts(2367)` (type overlap) and `ts(2339)` (property doesn't exist) on the background handler

### 2.4 Updated: `entrypoints/content/autofill.ts`

**Three tracking additions:**

1. **`step_match_failed`** — in `runAutofill`, after `matchStep` returns null:
   ```typescript
   trackEvent("step_match_failed", form, {
     url: window.location.pathname + window.location.search,
   });
   ```

2. **`field_fill_failed`** — in `fillField`, when `document.querySelector` returns null:
   ```typescript
   trackEvent("field_fill_failed", "pan_card", {
     field_id: field.field_id,
     selector: field.selector,
     step: (field as any)._step ?? "unknown",
   });
   ```

3. **`ao_code_failed`** — in `autoFillAOCode` catch block:
   ```typescript
   trackEvent("ao_code_failed", "pan_card", {
     pincode: pinCode,
     reason: err instanceof Error ? err.message : "unknown",
   });
   ```

**Step propagation fix:**
- In `runAutofill`'s field loop, fields are now spread with `_step`:
  ```typescript
  const field = { ...step.fields[i], _step: step.step };
  ```
- This allows `fillField` to include the step number in telemetry metadata without needing to restructure the function signature

**Debug log added (temporary, for testing):**
- `console.log("FormYaar: fillField called for", field.selector)` added during debugging — should be removed before Chrome Web Store submission

---

## 3. Website — `formyaar-website`

### 3.1 New file: `internal.html`
**Path:** `formyaar.pages.dev/internal`  
**Purpose:** Internal health checker dashboard — manually verify that NSDL page selectors still match the `pan_card.json` config after NSDL makes changes to their portal HTML.

**How it works:**
1. You open an NSDL page in browser while logged in
2. DevTools → Elements → right-click `<body>` → Copy → Copy outerHTML
3. Open `formyaar.pages.dev/internal`, enter password
4. Select the matching page from the list
5. Paste raw HTML → click Run check
6. Dashboard shows: which selectors were found ✓, which are missing ✗, which new IDs appeared on the page that aren't in the config

**Technical implementation:**
- Pure client-side — no backend, no API calls
- `DOMParser` parses pasted HTML into a live DOM
- `document.querySelector(selector)` checks each selector from the hardcoded config
- `querySelectorAll("[id]")` extracts all IDs on the page for new-ID detection
- Password: `fy@internal2026` (hardcoded in script — change before deploying)

**Pages covered (NSDL/Protean PAN card flow):**

| # | Page name | URL pattern | Selectors tracked |
|---|---|---|---|
| 1 | Registration | `endUserRegisterContact.html` | 9 |
| 2 | Token generated | `registerEndUser.html` | 0 (no autofill) |
| 3 | Main form (all steps) | `endUserLogin.html` | 40 |
| 4 | Review & confirm | `fullFormSave.html` | 1 |
| 5 | Payment mode | `fullFormSubmit.html` | 2 |
| 6 | Pay confirm | `ddSave.html` | 1 |
| 7 | Payment receipt | `paytmResponseAfterStatus.html` | 1 |
| 8 | Aadhaar auth | `continuePayment.html` | 2 |
| 9 | eKYC proceed | `proceedAadhaarDetails.html` | 1 |
| 10 | OTP verification | `authenticateOTP.html` | 0 (no autofill) |

Pages with 0 selectors are shown as "no autofill" and are not clickable.

**Design:** Light theme, DM Sans + JetBrains Mono, sticky header with IST clock, service tabs (NSDL active, UTI/Passport/Sarathi locked for future).

---

## 4. Telegram Bot Setup

**Bot name:** FormYaar Doctor  
**Username:** `@formyaar_doctor_bot`  
**Created via:** BotFather on Telegram  
**Chat ID obtained via:** `@userinfobot`

The bot is a one-way notification channel — it only sends messages, you never reply to it. Two types of alerts currently implemented:
- `field_fill_failed` — with field ID, selector, step, operator ID
- `step_match_failed` — with URL and operator ID

Additional alert types wired but not yet tested in production:
- `ao_code_failed` — with pincode and error reason
- `autofill_error` — generic crash with error message and step

---

## 5. Debugging Notes

**Root cause of "no Telegram alerts" initially:**
1. Extension build was stale — `npm run build` + reload required after every change
2. Direct `fetch` from content script was blocked by NSDL's CSP — resolved by routing through background service worker
3. Env vars (`TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`) were set on the wrong Railway service (`amiable-integrity` instead of `intelligent-energy`) — resolved by adding them to the correct service

**How it was diagnosed:**
- Added `console.log("FormYaar: fillField called for", field.selector)` to confirm `fillField` was being called
- Checked Network tab on NSDL page — no telemetry requests visible (confirmed CSP blocking)
- Ran manual fetch from background DevTools console — got `{ ok: true }` confirming backend was reachable from background context
- Checked Railway logs — saw `"Telegram credentials not configured"` error confirming the backend code was correct but env vars were missing on that service

---

## 6. Open Items from This Session

- [ ] Remove debug `console.log` from `autofill.ts` before Chrome Web Store submission
- [ ] Set up UptimeRobot on `/ping` endpoint (10 mins, free — deferred to next session)
- [ ] Build `/health` endpoint that checks Supabase + Razorpay connectivity (next session)
- [ ] Change `internal.html` password from `fy@internal2026` before deploying (or move to env-based auth later)
- [ ] AO code city rules JSON (`bareilly.json`, `kota.json`) — design deferred, needs more thought on matching logic

---

## 7. Sprint Roadmap Status

### Sprint 1 — Health Monitoring
- [x] Telegram alert on `field_fill_failed`
- [x] Telegram alert on `step_match_failed`
- [x] Deduplication (10-min window)
- [x] CSP bypass via background service worker
- [x] Internal HTML health checker dashboard
- [ ] `/health` endpoint (Supabase + Razorpay check)
- [ ] UptimeRobot monitor on `/health`

### Sprint 2 — Subscription Management (upcoming)
- [ ] Razorpay webhook → auto-update `subscription_status` in Supabase
- [ ] Subscription expiry enforcement on operator queue access
- [ ] Operator subscription purchase flow on website

### Sprint 3 — Admin Dashboard (upcoming)
- [ ] All operators + subscription status
- [ ] Manual revoke/extend subscriptions
- [ ] Payment history
- [ ] Forms filled, success rates, per-operator stats

---

*End of CHANGELOG10*
