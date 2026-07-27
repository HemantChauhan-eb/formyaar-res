# FormYaar — Changelog

> A running record of everything built, fixed, and decided across development sessions.
> Maintained for continuity between chats.

---

## Current State (as of May 10, 2026)

**Product:** Chrome extension that auto-fills Indian government forms (PAN card first).
**Stack:** WXT + React + TypeScript (extension) · Node/Express + TypeScript (backend on Railway) · Razorpay via Cloudflare Pages · Anthropic Claude Haiku (in-guide AI chat)
**Repos:** `HemantChauhan-eb/formyaar-extension` · `HemantChauhan-eb/formyaar-backend`
**Backend URL:** `https://formyaar-backend-production.up.railway.app`
**Compressor URL:** `https://formyaar.pages.dev/compress`
**Pricing:** ₹29 per form (B2C) · ₹999/month per device (B2B, planned)

---

## Session Log

---

### Session 1 — Initial Code Review & Architecture

**Code review of three repos (backend, extension, website)**

Bugs identified and addressed:
- Path traversal vulnerability in config file serving — fixed
- Razorpay order verification was in-memory Set — replaced with `/payment/status/:orderId` API call
- `setInterval` in service worker violated MV3 lifetime rules — replaced with `chrome.alarms`
- Rate limiting, input validation, CORS lockdown added to backend
- `crypto.timingSafeEqual` added for Razorpay webhook signature verification
- `helmet` and `pino` logging added to backend
- Graceful SIGTERM shutdown added

**Architecture decisions:**
- Aadhaar OCR auto-fill ruled out — UIDAI compliance (Section 40 Aadhaar Act), KUA registration required, criminal liability risk
- B2C: ₹29/form · B2B: ₹999/month flat per device

---

### Session 2 — Content Script Refactor

**`content/index.ts` god file (~1,500 lines) split into 8 focused modules:**
- `index.ts` — orchestration
- `constants.ts` — all magic numbers and config
- `fonts.ts` — Google Fonts loader
- `types.ts` — shared TypeScript interfaces
- `api.ts` — backend fetch helpers
- `overlay.ts` — spotlight overlay
- `panel.ts` — side panel UI and all screens
- `toasts.ts` — success/error toast messages

**Circular import resolution:** DOM custom events (`fy:open-panel`, `fy:show-resume`, `fy:resume-clicked`) used instead of direct imports between modules.

---

### Session 3 — Cross-Page Autofill Persistence Bug

**Bug:** Autofill state lost on page navigation (NSDL navigates between pages).

**Root cause:** Content scripts cannot access `chrome.storage.session` by default in MV3.

**Fix:** Added `chrome.storage.session.setAccessLevel({ accessLevel: "TRUSTED_AND_UNTRUSTED_CONTEXTS" })` to background script. This allows content scripts to read/write session storage, enabling autofill state to survive page navigations within the same browser session.

---

### Session 4 — PAN Card Config v3 + AO Code

**`pan_card.json` v3 built for NSDL's PAN portal (`onlineservices.proteantech.in`)**

Steps configured:
- Step 1: `registerEndUser.html` — application type, category, name, DOB, email, mobile, consent
- Step 2: `endUserLogin.html` stepy 1 — eKYC mode, ePAN option, Aadhaar last 4, personal details, parent name
- Step 3: `endUserLogin.html` stepy 2 — income source checkboxes, ISD code, residential status, passport/TIN
- Step 4: `endUserLogin.html` stepy 3 — AO code selection (Indian citizen / Defence)
- Step 5: `endUserLogin.html` stepy 4 — proof of identity/address/DOB, designation, declaration, place

**AO code auto-fetch logic:**
- Backend `/pincode/:pincode` endpoint returns state and city
- `autoFillAOCode()` selects state, waits for city dropdown to populate via AJAX, selects city, clicks Fetch
- `autoSelectAOCode()` uses MutationObserver to wait for table, then selects first valid non-exemption, non-company AO row

---

### Session 5 — User Data Collection Form

**Built inside the extension panel (`panel.ts`):**

Fields collected:
- Personal: first/middle/last name, DOB (DD/MM/YYYY), gender, email, mobile, income source
- Aadhaar: full 12-digit number (replaces the earlier last-4 only), PIN code as per Aadhaar
- Family: father's first/middle/last name, mother's first/middle/last name, whose name to print on card
- Verification: place (city), proof of DOB document type
- Additional: defence personnel status, passport number, TIN number

**Key decisions:**
- `last_name` made optional — NSDL allows single-name applicants (full name in first_name field)
- Aadhaar stored as full 12 digits; `aadhaar_last_4` retained via `.slice(-4)` for backward compatibility
- Father/mother last names optional; only first names required
- Validation errors highlight the field, scroll to it, and show error message inline

---

### Session 6 — PDF Compressor (`compress.html`)

**Built standalone compressor at `formyaar.pages.dev/compress`**

Features:
- Auto-fit mode: iterates up to 9 attempts (grayscale-first, multiple quality/scale combinations) targeting 280kb/page
- Manual mode with quality slider
- Hard gate: if file already under 300kb/page, shows "Use as-is" option
- Grayscale toggle (default ON — matches NSDL's acceptable quality)
- Multi-file support: front + back of ID cards can be combined
- Reorderable file list with up/down/remove
- Per-page memory cleanup (`canvas.width = 0`, `page.cleanup()`)
- Async `canvas.toBlob` to avoid blocking main thread
- White JPEG background fill for transparent PNGs
- Image input supported (JPG/PNG/WebP) with multi-file combine
- Source-aware rendering capped at 1600px (no upscaling)
- Cancel button mid-compression run

**Limitation noted:** HEIC not supported (decoder too heavy for browser).

---

### Session 7 — Upload Screen

**Built `uploadScreen.ts` — shown after last autofill step**

Features:
- "Take me to upload section" button — scrolls to `#addFile` or `#docsUpload` on NSDL page with orange pulse highlight
- Compressor link opens `formyaar.pages.dev/compress` in new tab
- 5 FAQ chips for common upload questions
- Free-text AI chat backed by Claude Haiku via backend `/ai/chat` endpoint
- Chat thread with user/bot bubbles

**NSDL upload page structure confirmed:**
- `#addFile` — orange "+" add file button
- `#docsUpload` — final upload submit button
- Token radio has dynamic ID (token number) but static class `tokenButton`
- `#submitForm11` — "Use above Token" button (disabled until radio clicked)

---

### Session 8 — Token Page Autofill

**Problem:** NSDL shows the token selection page on `registerEndUser.html` URL (same as page 1), not `endUserLogin.html`. The DOM is completely different — no stepy fieldsets.

**Fix in `matchStep()`:** Check for `input.tokenButton` presence FIRST, before any URL-based matching. If found, return the step with `is_token_page: true`.

**Token step added to `pan_card.json`:**
- Selector: `input.tokenButton` (class-based, not ID — ID is the dynamic token number)
- After clicking radio, `#submitForm11` is still disabled
- `fillField()` made `async`, `button_click` case uses MutationObserver to wait for `disabled` attribute to clear (3s timeout)
- Works: radio clicked → button enabled → "Use above Token" clicked → navigates to page 3

---

### Session 9 — Resume System

**Problem:** If user pays but closes browser before completing the form, they had no way back without paying again.

**Solution:** Persist session to `browser.storage.local` (survives browser restart, unlike `storage.session`).

**Files changed:**

`userData.ts` — Added:
- `ActiveSession` interface (`form`, `order_id`, `paid_at`, `completed`)
- `getActiveSession()`, `setActiveSession()`, `markSessionCompleted()`, `clearActiveSession()`

`background.ts` — `PAYMENT_VERIFIED` message now includes `order_id` from polling alarm scope.

`index.ts` — On `PAYMENT_VERIFIED`, writes `fy_active_session` to local storage. On page load, if no `autofillActive` in session but an incomplete local session exists, shows resume screen after banner delay.

`panel.ts` — Added `renderResumeScreen()` and `showResumeScreen(session)`. Resume screen shows "Payment confirmed" with "Start Filling Now" button (sets `autofillActive`, navigates to NSDL page 1) and "Start over" link (clears session, shows home).

`uploadScreen.ts` — Calls `markSessionCompleted()` when upload screen is shown (user reached the final step).

**Logic:**
- Incomplete session → show resume screen on next NSDL visit
- Completed session → show normal home screen
- No session → show normal home screen

---

### Session 10 — Card Icons Redesign

**Replaced simple SVG placeholder icons in panel with detailed card designs:**

- PAN Card — blue-gradient card, Income Tax Department header, gold emblem, AAAAA1234A sample PAN, card holder photo placeholder
- Aadhaar — cream card, saffron/green bands, UIDAI fingerprint logo, QR code, masked number XXXX XXXX XXXX
- Driving License — red header "UNION OF INDIA / DRIVING LICENCE", Ashoka Chakra, smart-card chip, photo area, signature strip
- Voter ID — cream/green ECI card, Hindi + English header, barcode strips
- Passport — navy booklet, gold gradient "PASSPORT" + "REPUBLIC OF INDIA", Ashoka Chakra emblem
- Visa — blue-to-pink gradient sticker, big pink VISA wordmark, MRZ lines at bottom

**Technical note:** All gradient IDs prefixed with `fy-` to avoid collision with NSDL page gradients.

---

### Session 11 — Real PAN Card Flow Mapping

**First real PAN card application completed with a live user (Patit Pawan Bera).**

**Complete flow mapped (10 pages/steps):**

| # | URL | Description | FormYaar action |
|---|---|---|---|
| 1 | `registerEndUser.html` | Basic registration | ✅ Autofill (done) |
| 2 | `registerEndUser.html` (token DOM) | Token selection page | ✅ Autofill (done) |
| 3 | `endUserLogin.html` (5 stepy steps) | Main multi-step form | ✅ Autofill (done) |
| 4 | `fullFormSave.html` | Full form review + Aadhaar 8 digits | ⚠️ Selectors identified, config pending |
| 5 | `fullFormSubmit.html` | Terms agreement + payment mode | ⚠️ Selectors identified, config pending |
| 6 | `ddSave.html` | Payment confirmation | ⚠️ Show message only |
| 7 | `paytmResponseAfterStatus.html` | Payment receipt (SUCCESS) | ⚠️ Show "click Continue" |
| 8 | `continuePayment.html` | Aadhaar eKYC consent | ⚠️ Tick checkbox, show OTP message |
| 9 | `proceedAadhaarDetails.html` | "2 steps away" info + e-KYC button | ⚠️ Click button |
| 10 | `authenticateOTP.html` | OTP entry | ❌ User does manually |

**Key findings:**
- Address fields auto-populated from Aadhaar — we don't need to collect or fill address for eKYC flow
- `fullFormSave.html` requires Aadhaar first 8 digits input: `#aadhaarNo_1`
- `fullFormSave.html` Proceed button: `#confirmSubmit`
- `fullFormSubmit.html` agree radio: `#agree` (value=`Y`)
- `fullFormSubmit.html` Proceed to Payment button: `#pay`
- `ddSave.html` Pay Confirm button: `#payConfirm`
- `paytmResponseAfterStatus.html` Continue button: `#continuePayment`
- `continuePayment.html` consent checkbox: `#consent`
- `continuePayment.html` Authenticate button: `#submitAadhaarFullForm`
- `proceedAadhaarDetails.html` Continue with e-KYC button: `#submitEsign`

**Config steps for pages 4-9 to be written next session.**

---

### Session 12 — Cafe Flow UI (B2B New Idea)

**Built `formyaar-cafe.html` — mobile-first user data collection form for the B2B cafe operator flow**

**What it does:**
- User scans QR code at internet cafe → opens this page on their phone
- Selects which form they need (PAN Card, others locked/coming soon)
- Fills their own personal data into a simple mobile-optimized form
- Submits → cafe operator receives the structured data → reviews → triggers autofill on their PC

**Form fields collected:**
- Personal: first/middle/last name, DOB, mobile, email, Aadhaar (12-digit)
- Family: father's first/middle/last name · mother's first/middle/last name (all split)
- Address: flat/house, street/area, city, state, PIN code
- Income: multi-select checkboxes (salary, business, house property, capital gains, other, no income)
- Status: citizenship radio (Indian Resident / NRI) · defence personnel radio (Yes/No)

**Design decisions:**
- Pure HTML/CSS/JS — zero dependencies, no build step, loads fast on 2G
- 16px minimum font size everywhere (prevents iOS zoom on input focus)
- 44px+ tap targets for all interactive elements
- Checkbox/radio items styled as full-width tappable pills
- Works on 360px wide screens (2015-era Android devices)
- Card icons reused from the extension panel (same SVG designs: PAN, Aadhaar, DL, Voter ID, Passport, Visa)
- Hindi-English (Hinglish) labels and placeholder text for non-tech users
- DOB auto-formats to DD/MM/YYYY as user types
- Aadhaar auto-formats to groups of 4 with spaces
- Validation scrolls to first error, highlights red, shows inline error message in Hindi

---

## Pending / Next Session

1. **Write `pan_card.json` steps for pages 4–9** using selectors identified above
2. **Fix resume system bug** — `autofillActive` in session storage was not being cleared properly, causing home screen to show instead of resume screen after browser restart
3. **Deploy CORS allowlist fix** — `fetchGuide` runs in page context, needs govt site origins whitelisted
4. **AO code fetch bug** — requires State + City populated before Fetch button works; sequencing fix pending
5. **Connect cafe form to backend** — currently front-end only with `console.log`; needs `/api/submit` endpoint and operator dashboard
6. **`formyaar.in` domain** — buy after first 20 real users

---

## Key Principles (Standing Decisions)

- **Ship first, strategize later** — no more strategy work until 20 real paying users exist
- **Config maintenance is the moat** — willingness to keep govt form configs updated long-term is the defensible advantage
- **B2B is primary, not fallback** — CSC operators and internet cafes reduce CAC and increase volume meaningfully
- **No Aadhaar OCR** — UIDAI compliance risk is criminal liability, not negotiable
- **`chrome.storage.session` needs explicit access level** in MV3 background scripts for content scripts to read it
- **DOM custom events** are the clean solution for module decoupling in content scripts where circular imports are otherwise unavoidable

---

## New Business Angle — The Cafe Flow (Explained)

### The Problem with the Current Extension Model

The original extension requires the cafe operator to:
1. Type the customer's details into the FormYaar panel themselves
2. Sit at the computer the whole time
3. Handle one customer at a time

This is faster than manually filling the govt website, but the operator is still the bottleneck.

### The New Flow

**What changes:** Instead of the operator typing the customer's data, the **customer types their own data** on their own phone — by scanning a QR code the cafe displays.

**How it works:**
1. Customer walks in, says "bhaiya PAN card banana hai"
2. Operator shows QR code (printed or on screen)
3. Customer scans with phone → opens FormYaar's mobile data form
4. Customer fills their own name, DOB, Aadhaar, parents' names, address etc. — everything they'd have told the operator anyway
5. On submit, data appears in the operator's queue on their PC
6. Operator clicks Review → sees the pre-filled form → clicks Accept
7. FormYaar autofills the actual NSDL website instantly using that data
8. Done — operator just reviewed and clicked, never typed

### Why This Is Better

**Removes typing entirely from the operator's workflow.** The operator's job goes from "type + navigate" to "review + click." That's maybe 5% of the original effort.

**Enables parallel processing.** While one customer fills their data on their phone, the operator can be reviewing and submitting another customer's form on the PC. Multiple forms in flight simultaneously — something impossible before.

**Fixes the blame problem.** If a PAN card application has wrong details, today the customer can blame the cafe operator ("aapne galat likha"). With the new flow, the customer typed their own data and reviewed it before submitting. The cafe operator is just a conduit. This is a meaningful liability shift that cafe owners will actually care about.

**Lowers the barrier to running a form-filling service.** Today you need: a PC, knowledge of government websites, patience to navigate complex forms, ideally someone tech-savvy. With FormYaar's cafe flow you need: a phone (for the QR), a PC (for review + submit), a ₹299/month subscription. The technical complexity is hidden. Literally anyone can run this as a side income.

**Creates a scalable unit model.** ₹299/month gets you unlimited form submissions (or fair-use capped). A cafe doing 30 PAN cards/month at ₹100/form earns ₹3,000 revenue for ₹299 cost. That's a 10x return on the subscription. The math sells itself.

### What Gets Built for This

- `formyaar-cafe.html` — the customer-facing mobile form (built this session, front-end complete)
- Backend `/api/submit` endpoint — receives customer data, stores it, notifies operator
- Operator dashboard — shows pending submissions as a queue with Review/Accept/Reject
- QR code generation per operator account — unique URL so submissions route to the right operator

The customer-facing form is done. The operator dashboard and backend integration are next.
