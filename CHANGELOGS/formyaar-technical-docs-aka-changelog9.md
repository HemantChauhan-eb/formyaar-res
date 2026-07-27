# FormYaar — Technical Documentation

**Last updated:** May 17, 2026
**Maintainer:** Hemant Chauhan (ExcuseME Brother)
**Repos:** `HemantChauhan-eb/formyaar-extension`, `HemantChauhan-eb/formyaar-backend`, `HemantChauhan-eb/formyaar-website`

---

## 1. System Overview

FormYaar is a Chrome extension that auto-fills Indian government forms (PAN card initially, DL/Passport on the roadmap) on official government portals. The system has three deployment surfaces and one shared database:

| Surface | Tech | Hosting | Purpose |
|---|---|---|---|
| **Extension** | WXT + React + TypeScript (MV3) | Chrome Web Store (pending) | The autofill engine; runs as a content script on government portals |
| **Website** | Static HTML + Cloudflare Pages Functions | Cloudflare Pages (`formyaar.pages.dev`) | Marketing, operator dashboard, mobile customer form, payment redirect, file compressor |
| **Backend** | Node.js + Express + TypeScript | Railway (`formyaar-backend-production.up.railway.app`) | Form configs, payment orchestration, AI chat, operator tokens, telemetry |
| **Database** | Supabase (Postgres + Auth) | Supabase Cloud (`wkubrgktujihesjjxyrk`) | Operators, submissions, operator tokens |

There are two distinct user flows:

1. **B2C direct flow** — End user installs the extension, fills their data into the side panel, pays ₹29, and the extension autofills the government portal.
2. **B2B operator flow** — Cafe operator subscribes (₹299/month), prints a QR code, customer scans it and fills their data on a mobile form. The data lands in the operator's queue. The operator reviews, clicks "Accept & Fill", and the extension autofills the form on the operator's PC.

Both flows converge on the same autofill engine in the extension.

---

## 2. Database Schema (Supabase Postgres)

Project: `wkubrgktujihesjjxyrk.supabase.co`

### 2.1 `submissions`

Stores customer form data submitted via the mobile QR flow. Polled by the operator queue endpoint.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | uuid | NO | Primary key, auto-generated |
| `operator_id` | uuid | YES | Foreign key to `operators.id` (which operator's queue) |
| `form_type` | text | NO | `pan_card`, `driving_license`, `passport` |
| `status` | text | YES | `pending`, `filling`, `completed`, `rejected` |
| `name` | text | YES | **Legacy** — kept for old rows; no longer written |
| `first_name` | text | YES | Customer's first name |
| `middle_name` | text | YES | |
| `last_name` | text | YES | |
| `father_first_name` | text | YES | |
| `father_middle_name` | text | YES | |
| `father_last_name` | text | YES | |
| `mother_first_name` | text | YES | |
| `mother_middle_name` | text | YES | |
| `mother_last_name` | text | YES | |
| `father_name` | text | YES | **Legacy** — combined father name |
| `mother_name` | text | YES | **Legacy** — combined mother name |
| `mobile` | text | YES | 10-digit Indian mobile |
| `email` | text | YES | |
| `dob` | text | YES | DD/MM/YYYY format |
| `city` | text | YES | Verifier place |
| `state` | text | YES | |
| `pincode` | text | YES | 6-digit; used by autofill for AO code lookup |
| `aadhaar_last_4` | text | YES | Only last 4 digits ever stored (UIDAI compliance) |
| `income_source` | text | YES | Comma-separated: `salary,business,house_property,capital_gains,other_sources,no_income` |
| `defence` | boolean | YES | Defence personnel flag |
| `proof_of_dob` | text | YES | Currently always empty from QR flow |
| `created_at` | timestamptz | YES | |
| `completed_at` | timestamptz | YES | Set when autofill completes |

**Status lifecycle:**
- `pending` → customer submitted, awaiting operator review
- `filling` → operator clicked Accept & Fill (autofill in progress)
- `completed` → autofill reached the upload step
- `rejected` → operator rejected the submission

**RLS:** Currently disabled (`UNRESTRICTED`). Open todo to enable RLS with policies that scope rows to `auth.uid() = operator_id`.

### 2.2 `operators`

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key; matches Supabase auth user id |
| `email` | text | Operator's Google account email |
| `subscription_status` | text | `active` or `inactive` |
| `subscription_expires_at` | timestamptz | When the ₹299/month subscription expires |
| `created_at` | timestamptz | |

Operators authenticate via Supabase Google OAuth on `operator-dashboard.html`. The row is upserted on every dashboard load.

### 2.3 `operator_tokens`

Short-lived one-time tokens used to log the extension into an operator account without exposing the OAuth flow inside the extension.

| Column | Type | Notes |
|---|---|---|
| `token` | text | 12-char uppercase alphanumeric, primary key |
| `operator_id` | uuid | FK to `operators.id` |
| `expires_at` | timestamptz | 5 minutes from creation |
| `used_at` | timestamptz | Set when consumed by `verify-token` |
| `created_at` | timestamptz | |

A token is single-use: once `used_at` is populated, subsequent `verify-token` calls reject it.

---

## 3. Backend (Express on Railway)

**Base URL:** `https://formyaar-backend-production.up.railway.app`
**Repo:** `HemantChauhan-eb/formyaar-backend`
**Entrypoint:** `src/index.ts`

### 3.1 Stack & middleware

- **Framework:** Express 5
- **Language:** TypeScript, compiled to `dist/` via `tsc`, run with `node dist/index.js`
- **Logging:** Pino (structured JSON in prod, pino-pretty in dev)
- **Security middleware:**
  - `helmet()` — standard HTTP security headers
  - `express-rate-limit` on the AI chat endpoint (10 req/min/IP)
  - `cors` with a strict allowlist:
    - `https://formyaar.pages.dev`
    - `chrome-extension://*` (regex)
    - The five supported govt portal origins
  - Methods restricted to `GET`, `POST`, `PATCH`

**Known CORS gap:** government-portal origins are listed but the `fetchGuide` call in the content script runs in the page's context, so configs are sometimes blocked. Fix is pending deploy.

### 3.2 Routes

All routers are mounted in `src/index.ts`:

| Mount | Router file | Purpose |
|---|---|---|
| `/ai` | `routes/chat.ts` | Claude Haiku-powered help chat |
| `/configs` | `routes/configs.ts` | Serve form JSON configs |
| `/payment` | `routes/payment.ts` | Razorpay order creation, signature verification, status polling |
| `/pincode` | `routes/pincode.ts` | PIN code → state + city resolver |
| `/operator` | `routes/operator.ts` | Token gen/verify, queue listing, status updates |
| `/telemetry` | `routes/telemetry.ts` | Allowlist-gated event logger |
| `/ping` | inline | Health check |

#### 3.2.1 `POST /ai/chat`

Body: `{ fieldId, fieldExplanation, userMessage }`

Validates `userMessage` is 1–500 chars and `fieldExplanation` is ≤1000 chars. Calls Anthropic's `claude-haiku-4-5-20251001` with `max_tokens: 300`. There are two system prompts: a specialized one for `fieldId === "upload_proof_dob"` (the final upload step with FAQ-driven help), and a generic field-help prompt for everything else. Returns `{ response: string }`.

#### 3.2.2 `GET /configs/:form/latest`

Allowlist: `pan_card`, `passport`, `driving_license`, `aadhaar`. Reads `configs/<form>.json` from disk, parses it, returns the full JSON. The allowlist exists to prevent path traversal (`/configs/../../etc/passwd/latest` etc.).

Configs are version-stamped (`version` field). The extension always fetches `/latest` — there's no client-side caching, so a new deploy pushes config changes to every user immediately. This is the "self-healing" mechanism: when NSDL changes a field selector, only the JSON needs updating.

#### 3.2.3 Payment routes

- `POST /payment/create-order` — Creates a ₹29 (2900 paise) Razorpay order. Returns `{ success, order_id, amount }`.
- `GET /payment/order/:order_id` — Fetches order from Razorpay (used by `pay.html` to display the amount authoritatively, not from query params).
- `POST /payment/confirm` — Receives `razorpay_order_id`, `razorpay_payment_id`, `razorpay_signature`. Verifies HMAC-SHA256 signature with `crypto.timingSafeEqual` (constant-time compare to prevent timing attacks).
- `GET /payment/status/:order_id` — Polled by the extension's background script. Calls `razorpay.orders.fetchPayments(order_id)` and returns `{ paid: true }` if any payment is `captured`.

Critical design: there is **no in-memory order state**. The backend queries Razorpay directly every time. This was a deliberate rewrite from an earlier `paidOrders: Set<string>` cache that lost state on service worker restart.

#### 3.2.4 `GET /pincode/:pin`

Validates 6-digit PIN. Calls `api.postalpincode.in` (public Indian postal API). Maps the returned state name through a hardcoded `STATE_MAP` to NSDL's exact uppercase format (e.g., `Jammu & Kashmir` → `JAMMU AND KASHMIR`). Returns `{ state, city }`. Used by the autofill engine to pre-select the state and city dropdowns on the AO code page.

#### 3.2.5 Operator routes

- `POST /operator/generate-token` — Body `{ operator_id }`. Generates a 12-char token via `crypto.randomBytes(9).toString("base64url").slice(0,12).toUpperCase()`, inserts a row into `operator_tokens` with 5-minute expiry. Returns `{ token, expires_at }`.
- `POST /operator/verify-token` — Body `{ token }`. Looks up the row, checks not used and not expired, joins to `operators` table, marks `used_at`, returns operator session object.
- `GET /operator/queue/:operator_id` — Returns all `submissions` rows with `status='pending'` for the operator, oldest first.
- `PATCH /operator/submission/:id/status` — Body `{ status }`. Updates row status. Allowlist enforced: `pending`, `filling`, `rejected`, `completed`.

The Supabase client is created lazily inside each handler via `getSupabase()` rather than at module load, so missing env vars don't crash boot.

#### 3.2.6 Telemetry

`POST /telemetry/event` accepts only events on an explicit allowlist (`banner_shown`, `panel_opened`, `payment_started`, `guide_started`, `guide_completed`, `upload_screen_shown`, etc.). Unknown events return 400. Events are pino-logged with timestamp and metadata.

### 3.3 Environment variables

- `ANTHROPIC_API_KEY` — Claude API
- `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET` — Payment gateway
- `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` — Service role used for operator queue/token queries (RLS bypass)
- `PORT` — Railway-injected
- `LOG_LEVEL`, `NODE_ENV` — Logging config

### 3.4 Deployment

Railway auto-deploys from the `main` branch. Build command from `nixpacks.toml`: `npm run build` (runs `tsc`). Start: `npm run start` (runs `node dist/index.js`). Graceful shutdown on `SIGTERM` is implemented.

---

## 4. Extension (WXT + React + TypeScript)

**Repo:** `HemantChauhan-eb/formyaar-extension`
**Manifest version:** 3
**Framework:** WXT 0.20 (Chrome extension framework on Vite)

### 4.1 Manifest

Defined in `wxt.config.ts`:

```typescript
permissions: ["storage", "activeTab", "scripting", "alarms", "tabs"]
host_permissions: [
  "https://onlineservices.proteantech.in/*",
  "https://onlineservices.nsdl.com/*",
  "https://www.utiitsl.com/*",
  "https://passporthub.gov.in/*",
  "https://sarathi.parivahan.gov.in/*",
  "https://formyaar.pages.dev/*",
  "https://formyaar-backend-production.up.railway.app/*",
]
```

Scoped host permissions — no `<all_urls>`. This is a privacy posture deliberately maintained.

### 4.2 Three entrypoints

```
entrypoints/
├── background.ts          # Service worker
├── popup/App.tsx          # Toolbar popup (280px)
└── content/
    ├── index.ts           # Content script entry
    ├── panel.ts           # Side panel UI (vanilla DOM, ~1500 lines)
    ├── autofill.ts        # Autofill engine
    ├── userData.ts        # Storage layer for user data
    ├── uploadScreen.ts    # Final-step upload helper
    ├── constants.ts       # URLs, z-indices, timing constants
    ├── fonts.ts           # Google Fonts loader
    ├── toasts.ts          # Floating notification UI
    ├── supabase.ts        # Operator session storage
    ├── telemetry.ts       # Fire-and-forget event logging
    └── types.ts           # Shared types
```

The popup is React; the side panel is vanilla DOM injected into the page. The reason for vanilla DOM in the panel: it needs to coexist with the host government portal's CSS without React's runtime overhead, and the architecture predates a clean React-in-Shadow-DOM setup.

### 4.3 Content script lifecycle

`entrypoints/content/index.ts` matches:
```
*://*.proteantech.in/*
*://*.nsdl.com/*
*://*.utiitsl.com/*
*://*.passporthub.gov.in/*
*://*.sarathi.parivahan.gov.in/*
*://formyaar.pages.dev/*
```

On load:
1. Registers a message listener for `OPEN_PANEL` and `PAYMENT_VERIFIED` events from the background script.
2. On the NSDL `endUserLogin.html` page, attaches a `click` listener that re-runs autofill when the user clicks `.button-next` (stepy framework's next button). Uses a `MutationObserver` on `[style*="display: none"]` to detect when the next fieldset becomes visible.
3. If `SITE_CONFIGS[hostname]` matches, shows the contextual banner (`showContextualBanner`) after a 1.5s delay.
4. Checks `browser.storage.session.autofillActive` — if set, the user is mid-flow (e.g., page navigated during autofill). Resumes autofill for the stored form.
5. If no active autofill, checks for a paused session in `browser.storage.local` (`fy_active_session`). If found and not marked completed, shows the resume screen.

### 4.4 Side panel (`panel.ts`)

A 400px right-anchored panel injected as a `div#formyaar-panel`. Uses inline styles and z-index `2147483647` to sit above page content. Has multiple screens:

- `#fy-home` — Document picker (PAN card available, others locked)
- `#fy-payment` — ₹29 payment confirmation
- `#fy-filling` — Live progress during autofill
- `#fy-verify` — Post-fill success screen
- `#fy-upload` — Final upload helper with AI chat
- `#fy-resume` — Resume paused session
- `#fy-operator-login` — Token-based operator sign-in
- `#fy-operator-queue` — Pending submissions list
- `#fy-operator-review` — Submission detail + Accept/Reject

Screens are swapped by toggling `display: flex/none`. There's also a `#fy-userform-screen` dynamically appended for the user data collection form (rendered fresh from `renderUserFormScreen()` rather than baked into the panel HTML).

Panel state is opened/closed by translating `right: 0px` ↔ `right: -400px`. Click-outside-to-close and Escape-to-close are wired up.

### 4.5 Autofill engine (`autofill.ts`)

The engine is config-driven. Flow:

1. `runAutofill(form)` is called from either:
   - `panel.ts` after payment verification (B2C flow), or
   - `runAutofillFromSubmission(sub)` after operator accepts a submission (B2B flow)
2. Show the filling screen.
3. Fetch the form config from `GET /configs/:form/latest`.
4. Resolve user data from `getUserData()` (more on this in §4.6).
5. Call `matchStep(config)` to find which step the current page is.
6. For each field in that step's config:
   - Resolve the value via `resolveValue(field, userData)`
   - Call `fillField(field, value)` based on `field.type`
   - Update the progress UI between fields
7. On step 4 (AO code page), call `autoFillAOCode(pin)` to do the state→city→fetch→radio dance.
8. On the last step, show the upload screen instead of the generic verify screen.

#### Step matching

`matchStep` has three modes:

1. **Token page detection:** If `input.tokenButton` exists in the DOM, this is the "existing application found" page. Returns the step config with `is_token_page: true`.
2. **URL-based matching:** For pages other than `endUserLogin`, matches `page_pattern` against `location.pathname + location.search`.
3. **Stepy step detection:** On `endUserLogin.html`, NSDL uses the "stepy" multi-step wizard. Finds the visible `.stepy-step` (the one not `display: none`), gets its index, returns the step config with the matching `stepy_index`.

#### Field types

Defined in `pan_card.json` and handled by `fillField`:

| Type | Behavior |
|---|---|
| `text` / `date` | Uses native HTMLInputElement value setter (bypasses React/jQuery value tracking), dispatches `input`, `change`, `blur` events |
| `select` | Iterates `select.options`, matches by `value` or `text` (controlled by `match_by`), sets `selectedIndex`, fires `change`. If `window.$` and Select2 are present, also triggers jQuery `.val().trigger('change')` for Select2 widgets |
| `checkbox` | Sets `checked`, fires `change` and `click` |
| `radio` | Sets `checked` if `shouldSelect=true`. Has a `defence_selector` mechanism: on the AO code page, if `userData.is_defence` is true, clicks the defence radio instead of the Indian citizen radio. `forceClick` flag handles cases where the radio is already selected but the change event must still fire |
| `button_click` | Calls `.click()` and dispatches a synthetic `MouseEvent`. If the button is disabled, sets up a `MutationObserver` on the `disabled` attribute and waits up to 3s for it to enable (used for the token-page "Use Token" button) |

#### Value resolution

`resolveValue(field, userData)`:
- `value_source === "static"` → returns `field.static_value`
- `value_source === "user.<key>"` → returns `userData[key]`
- Special case: if both `value_source = "user.X"` AND `static_value` is set, this is a checkbox that should be checked when `userData[X] === static_value`. Used for the income_source checkboxes where one user field maps to multiple checkboxes.
- Derived key: `user.aadhaar_first_8` returns the first 8 digits of `userData.aadhaar_number`.

#### AO code auto-fill (`autoFillAOCode`)

The most complex piece. Sequence:
1. Call `GET /pincode/<pin>` to get state and city.
2. Find `#state_aoCode` select, match by text (uppercase), set value, fire `change` + Select2 trigger.
3. Wait for `#city_aoCode` to populate via NSDL's AJAX (MutationObserver, 5s timeout).
4. Match city by text. Falls back to partial-match if exact match fails (NSDL inconsistencies like "BAREILLY (UTTAR PRADESH)" vs "BAREILLY").
5. Wait 800ms, click `#fetchAOList` button.
6. Call `autoSelectAOCode()` — MutationObserver waits for `#table_id` rows to populate. Iterates rows, skips ones with "EXEMPTION" or "COMPANY CASES" in description, clicks the first valid radio.

### 4.6 User data layer (`userData.ts`)

Defines the `UserData` interface (24 fields including all name components, parents' names, Aadhaar, address, declaration fields).

**Storage strategy — critical to understand:**

Three storage tiers:

1. **`browser.storage.local`** under key `fy_user_data` — B2C user's saved details. Survives browser restart and reinstalls. Persisted by `saveUserData()`.
2. **`browser.storage.session`** under key `fy_operator_submission` — Operator-flow override. Set by `setOperatorSubmission()` when the operator accepts a queue item. Survives page navigation but cleared on browser close.
3. **`browser.storage.local`** under key `fy_active_session` — Paused B2C session tracking (`form`, `order_id`, `paid_at`, `completed`). Drives the resume screen.

`getUserData()` priority:
1. First check session storage for `fy_operator_submission`. If present and has `first_name`, return it merged with `EMPTY_USER_DATA`.
2. Otherwise read `fy_user_data` from local storage.
3. Fallback: return `EMPTY_USER_DATA`.

**Why session storage and not `window.__fy_operator_userdata`:** the original implementation stored the operator submission on the `window` object. This worked for step 1, but every time NSDL navigated to the next page, `window` was destroyed and the override was lost. Steps 2+ fell back to the operator's personal data (or empty), causing every field beyond step 1 to fail. The fix was moving to `browser.storage.session`, which persists across navigations within the same tab/window.

`browser.storage.session.setAccessLevel({ accessLevel: "TRUSTED_AND_UNTRUSTED_CONTEXTS" })` is set in the background script — without it, content scripts cannot read session storage.

### 4.7 Background service worker (`background.ts`)

Handles four message types from the content script:

1. `AI_CHAT` — Forwards to `POST /ai/chat`, returns the response.
2. `CREATE_PAYMENT` — Calls `POST /payment/create-order`, returns the order.
3. `OPEN_RAZORPAY` — Opens `https://formyaar.pages.dev/pay?order_id=<id>` in a new tab, then sets up payment polling.
4. `OPEN_URL` — Opens an arbitrary URL in a new tab (used for the operator dashboard link).

**Payment polling:** uses `chrome.alarms` (not `setInterval`). Reason: in MV3, the service worker can be killed at any time, and `setInterval` does not survive. `chrome.alarms` does. Polling interval is 0.1 minutes (~6 seconds), max 60 attempts (5 minutes total). Pending payment state is stored in `browser.storage.session.pendingPayment` as `{ orderId, originTabId, attempts }`.

The alarm handler:
1. Fetches `GET /payment/status/:order_id`
2. If paid: clears pending state, clears alarm, sends `PAYMENT_VERIFIED` to the origin tab
3. If not paid: increments attempts, re-saves state
4. If timed out: clears everything silently

### 4.8 Popup (`popup/App.tsx`)

React component, 280px wide. On mount, queries the active tab's URL, checks against `SUPPORTED_SITES`, and shows one of two states:

- **Supported site detected:** Shows "FormYaar is active — PAN Card form detected" and an "Open FormYaar" button that sends `OPEN_PANEL` to the content script.
- **Unsupported site:** Shows a friendly "Not active on this page" message and a list of supported sites.

### 4.9 Telemetry

`telemetry.ts` exposes `trackEvent(event, form, metadata)`. Fire-and-forget POST to `/telemetry/event`. Catches all errors silently — telemetry must never break the user flow. Events fired throughout the codebase: `banner_shown`, `panel_opened`, `payment_started`, `guide_started`, `guide_completed`, `upload_screen_shown`, `compressor_opened`, `faq_clicked`, etc.

---

## 5. Website (Cloudflare Pages)

**URL:** `https://formyaar.pages.dev`
**Repo:** `HemantChauhan-eb/formyaar-website`
**Hosting:** Cloudflare Pages with Pages Functions for the `/api/*` routes

### 5.1 Page inventory

| Path | Purpose |
|---|---|
| `/index.html` | Marketing landing page |
| `/privacy-policy.html` | Detailed DPDP-compliant privacy policy |
| `/terms.html` | Refund + ToS (stub — needs expansion) |
| `/contact.html` | Help page with phone + email |
| `/pay.html` | Razorpay checkout redirect target |
| `/compress.html` | Client-side PDF/image compressor |
| `/operator-login.html` | Pre-dashboard Google sign-in (legacy; dashboard handles auth too) |
| `/operator-auth-callback.html` | Supabase OAuth callback handler |
| `/operator-dashboard.html` | Operator's main dashboard (QR, stats, token generator) |
| `/user-form.html` | Customer document picker (scanned via QR) |
| `/pan-form.html` | Customer's PAN data entry form (mobile-optimized) |

### 5.2 Pages Functions

`functions/api/submit.js` — Single Pages Function that handles `POST /api/submit`. Receives the customer's mobile form data, validates `operator_id`, `form_type`, `first_name`, `mobile` are present, then inserts into Supabase via the REST API with `Prefer: return=representation` to get the row id back. Returns `{ success: true, id }` so `pan-form.html` can show a reference number to the customer.

Uses the anon publishable key (`sb_publishable_*`), not the service role. RLS would need a policy that allows anonymous inserts to `submissions` — currently RLS is disabled.

### 5.3 Operator dashboard (`operator-dashboard.html`)

Self-contained: imports `@supabase/supabase-js` from a CDN. On load:

1. Checks `window.location.hash` for an OAuth `access_token` (returning from Google). If found, exchanges via `supabase.auth.exchangeCodeForSession()`, then `window.history.replaceState` to clean the URL.
2. Calls `supabase.auth.getSession()`. If no session, shows the auth screen with a Google sign-in button.
3. With a session, upserts the operator row, fetches stats in parallel:
   - Total completed forms (count query on `submissions`)
   - This month's completed forms (count + `gte('created_at', monthStart)`)
   - Operator subscription status
4. Renders the QR code via `qrcodejs` library, pointing to `https://formyaar.pages.dev/user-form.html?op=<operator_uuid>`.
5. QR download: draws the QR onto a 320×380 canvas with a navy header, tagline, and tricolor strip, then triggers a PNG download.
6. **Auto-generates an extension token on dashboard load** — calls `POST /operator/generate-token`. Displays the 12-char token with a 5-minute countdown timer. The user can copy it to paste into the extension.

### 5.4 Mobile customer form (`pan-form.html`)

A pure-HTML form designed for mobile. Validates client-side (name required, DOB format `DD/MM/YYYY`, mobile is 10 digits starting with 6-9, email format, Aadhaar last 4 = exactly 4 digits, PIN = 6 digits). Income source supports multi-select; values are joined with commas.

On submit, posts to `/api/submit` with all 22 fields. Shows a reference ID like `FY-A1B2C3D4` derived from the returned row id.

**Naming refactor history:** This form used to send combined `name`, `father_name`, `mother_name` fields. The extension would split on spaces, which broke for 2-word names. The mobile form, the Pages Function, and the database schema were all refactored to use separate `first_name`/`middle_name`/`last_name` triples for self, father, and mother. Old rows still have the legacy combined fields populated.

### 5.5 PDF/Image compressor (`compress.html`)

A standalone 100%-client-side tool. Uses `pdf.js` and `pdf-lib`. Accepts PDFs or images, has two modes:

- **Auto-fit:** Tries 9 different (quality, scale, grayscale) attempts in order, returns the first one that fits under 280kb per page (300kb hard limit with 20kb headroom).
- **Manual:** User picks quality 0.15–0.95 and toggles grayscale.

Includes a "hard gate" — if all input files together are already under 300kb/page, refuses to compress (would only make it worse) and offers to bundle as-is. Multi-file uploads are merged into a single PDF.

The compressor is linked from the extension's upload helper screen for users whose proof of DOB document is too large.

### 5.6 Marketing landing (`index.html`)

Sections: hero with browser mockup, problem agitation, supported forms (PAN active, others locked), trust/privacy pillars, four-tier pricing (₹29 single, ₹99 pack of 5, ₹299/month operator), testimonials (currently placeholder), FAQ accordion, footer. Ashoka chakra and tricolor branding throughout. The B2B ROI calculator section is commented out pending a redesign.

---

## 6. End-to-end flows

### 6.1 B2C single-form flow

1. User visits `onlineservices.proteantech.in/paam/registerEndUser.html`.
2. Content script loads, banner appears after 1.5s.
3. User clicks PAN card → user form screen → fills 24 fields → clicks "Continue to Pay ₹29".
4. Data saved to `browser.storage.local.fy_user_data`. Payment screen shows.
5. User clicks Pay → `CREATE_PAYMENT` → background calls `/payment/create-order` → returns order id.
6. Background sends `OPEN_RAZORPAY` → opens `pay.html?order_id=<id>` in a new tab → starts `chrome.alarms` poll.
7. User completes payment in Razorpay checkout (loaded by `pay.html`).
8. Razorpay redirects back to `pay.html` with success → handler posts to `/payment/confirm` for signature verification.
9. Meanwhile the background's poll hits `/payment/status/<id>` → returns `paid: true` → sends `PAYMENT_VERIFIED` to the origin tab.
10. Content script saves `fy_active_session` and `autofillActive`, calls `runAutofill('pan_card')`.
11. Autofill executes step 1's fields. NSDL navigates to step 2.
12. Content script reloads, sees `autofillActive`, calls `runAutofill` again, fills step 2.
13. Repeat for steps 3, 4 (with AO code lookup), 5.
14. Last step → upload screen shows. User uploads docs, solves CAPTCHA, submits manually.

### 6.2 B2B operator flow

1. Operator opens dashboard, prints QR code.
2. Customer scans QR on phone → lands on `user-form.html?op=<uuid>` → picks PAN → fills `pan-form.html`.
3. `pan-form.html` POSTs to `/api/submit` → row inserted in `submissions` with `status='pending'`.
4. Customer sees ref ID and is told to talk to the operator.
5. Operator opens FormYaar extension panel on their PC → clicks "Cafe operator? Sign in here".
6. If not signed in: operator clicks "Open Dashboard", generates a token, pastes it into the extension. Extension calls `/operator/verify-token` → stores `OperatorSession` in `browser.storage.local`.
7. Operator queue screen loads → calls `/operator/queue/<id>` → shows pending submissions.
8. Operator clicks a tile → review screen shows full details.
9. Operator clicks "Accept & Fill" → PATCH status to `filling`, set `autofillActive`, call `setOperatorSubmission(userData)` with the customer's data, call `runAutofillFromSubmission(sub)`.
10. From here it's identical to the B2C flow, except `getUserData()` returns the operator-submission data from session storage instead of the operator's personal saved data.

### 6.3 Resume flow

If the user pays but doesn't complete autofill in one session (e.g., closes the tab):
1. `fy_active_session` is in local storage with `completed: false`.
2. Next time they visit any supported site, content script detects the session.
3. Resume screen shows with "Start Filling Now" and "Start over" buttons.
4. "Start Filling Now" sets `autofillActive` and navigates to NSDL.
5. NSDL may show "existing application found" → the token-page step config handles selecting the existing token automatically.

---

## 7. Form config schema (`configs/pan_card.json`)

The single most important piece of business logic. Schema:

```json
{
  "form": "pan_card",
  "version": 5,
  "site": "onlineservices.proteantech.in",
  "steps": [ { ... }, ... ]
}
```

Each step:

```json
{
  "step": 2,
  "page_pattern": "endUserLogin",
  "page": "endUserLogin",
  "stepy_index": 1,
  "stop_message": "Personal details filled. Please verify and click Next.",
  "fields": [ { ... }, ... ]
}
```

Special step flags:
- `is_token_page: true` — Step is shown when `input.tokenButton` exists in the DOM
- `stepy_index: 0` to `4` — Which stepy fieldset this corresponds to on the `endUserLogin` page
- `guidance_only: true` — Don't autofill; just show the verify screen
- `page_pattern` — Substring match against `location.pathname + location.search`

Each field:

```json
{
  "field_id": "first_name",
  "type": "text",
  "selector": "#f_name_end",
  "value_source": "user.first_name",
  "explanation": "Your first name as per Aadhaar."
}
```

For checkboxes mapping multiple options to one user field (income source):

```json
{
  "field_id": "income_salary",
  "type": "checkbox",
  "selector": "#sal_id",
  "value_source": "user.income_source",
  "static_value": "salary"
}
```

This checks the box when `userData.income_source === "salary"`.

Selects use `match_by: "value"` or `match_by: "text"` depending on whether NSDL's option value attribute is meaningful or only the display text matters.

The current `pan_card.json` has 12 steps covering: contact registration, token page, 5 stepy steps on `endUserLogin`, the `fullFormSave` confirmation, the `fullFormSubmit` terms agreement, the `ddSave` payment confirmation, the `paytmResponseAfterStatus` continue button, the Aadhaar consent, and the final e-sign step.

---

## 8. Security posture

### 8.1 Data minimization
- Aadhaar: only last 4 digits stored. Full 12-digit number is collected in the extension's local form (for the field display) but only the last 4 are ever sent anywhere outside the user's device for autofill purposes.
- B2C user's personal data is in `browser.storage.local` only. Never transmitted to FormYaar servers.
- B2B customer submissions go through `formyaar.pages.dev` → Supabase. Stored only as long as needed for the operator to fill the form.

### 8.2 Authentication
- Operators use Supabase Google OAuth on the dashboard. Extension auth uses one-time tokens (12-char, 5-min expiry, single-use) instead of embedding OAuth in the extension.
- Service role key never leaves the backend.

### 8.3 Payment integrity
- Razorpay webhook signatures verified with `crypto.timingSafeEqual` (constant-time compare).
- No order state is cached in memory — every status check queries Razorpay directly. Prevents stale-state attacks after service worker restart.
- The amount displayed on `pay.html` is fetched from Razorpay (`GET /payment/order/:order_id`), not trusted from URL params.

### 8.4 Input validation
- Pincode endpoint: regex `/^\d{6}$/`.
- AI chat: 1–500 chars on user message, ≤1000 on field explanation.
- Telemetry: allowlist of event names.
- Configs: allowlist of form names (no path traversal).
- Rate limit on AI chat: 10/min/IP via `express-rate-limit`.

### 8.5 CORS
- Extension origin pattern: `chrome-extension://*` (regex).
- Government portal origins explicitly listed (needed because the content script runs in the page's context for some calls).
- `formyaar.pages.dev` allowed.

### 8.6 Helmet
Applied globally on the backend. Default settings cover X-Content-Type-Options, X-Frame-Options, HSTS, CSP defaults, etc.

---

## 9. Known issues & open work

### Critical path (Chrome Web Store submission blockers)
1. **Razorpay-bank account linkage** — savings account decision made, switch to current+GST when revenue scales.
2. **Google OAuth verification** — currently in test mode; requires domain, legal pages, app submission.
3. **Legal pages** — Privacy Policy is comprehensive; Terms is a stub that needs proper refund policy + ToS expansion.
4. **Chrome Web Store submission** — blocked on all of the above.
5. **Subscription tracking system** — `operators.subscription_status` exists but no system updates it. No Razorpay-on-website for B2B subscription purchases yet.

### Functional gaps
- AO codes: PIN code lookup works via postalpincode.in, but ~30 cities have NSDL formatting issues and need explicit mapping.
- No back buttons on 2 panel screens.
- Panel opens on `formyaar.pages.dev/pay` (should be suppressed).
- "Add to Chrome" link on website is broken.
- Form auto-formatting in panel (date masking etc.) needs work.
- Defence personnel config path is stubbed but not fully tested.

### Polish
- Cafe operator entry button is a plain text link, needs proper styling.
- `contact.html` is minimal — needs to match other pages' design language.
- Payment + form history dashboard for B2C users doesn't exist.
- Supabase RLS is disabled; needs policies and re-enable.
- `formyaar.in` domain not purchased yet (~₹600–900/yr Hostinger).
- Some dummy phone numbers and font inconsistencies on the website.

### Architecture
- The income_source field in `UserData` is typed as a single value, but the mobile form supports multi-select and stores comma-separated. The autofill engine's checkbox-matching logic only matches the first one. Needs to be `income_source: string[]` and the matcher updated.
- Panel UI is vanilla DOM in 1500-line strings; long-term should migrate to React-in-Shadow-DOM for maintainability.
- No automated tests anywhere yet.

---

## 10. Operational reference

### Common commands

**Extension dev:**
```bash
cd formyaar-extension
npm run dev          # WXT dev server with HMR
npm run build        # Production build to .output/chrome-mv3
npm run zip          # Zip for Chrome Web Store submission
```

**Backend dev:**
```bash
cd formyaar-backend
npm run dev          # nodemon + ts-node, watches src/
npm run build        # tsc → dist/
npm run start        # node dist/index.js
```

**Website:**
Pure HTML/CSS/JS. Cloudflare Pages auto-deploys on git push to `main`.

### Useful URLs

- Backend health: `https://formyaar-backend-production.up.railway.app/ping`
- Operator dashboard: `https://formyaar.pages.dev/operator-dashboard.html`
- Customer form: `https://formyaar.pages.dev/user-form.html?op=<uuid>`
- Compressor: `https://formyaar.pages.dev/compress`
- Supabase studio: `https://supabase.com/dashboard/project/wkubrgktujihesjjxyrk`

### Debugging

- Extension logs: open the target govt portal, DevTools → Console. Background script logs: `chrome://extensions/` → FormYaar → "Service Worker" link.
- Backend logs: Railway dashboard → Deployments → Logs (pino JSON in prod, pretty in dev).
- Supabase queries: studio's SQL editor or table editor.
- Razorpay payments: Razorpay dashboard → Payments / Orders sections.

---

## Appendix A: Key file locations

```
formyaar-extension/
├── wxt.config.ts                      # Manifest config
├── entrypoints/
│   ├── background.ts                  # Service worker
│   ├── popup/App.tsx                  # Toolbar popup
│   └── content/
│       ├── index.ts                   # Entry, lifecycle, message routing
│       ├── panel.ts                   # ~1500-line side panel UI
│       ├── autofill.ts                # Autofill engine
│       ├── userData.ts                # Storage abstractions
│       ├── uploadScreen.ts            # Final-step upload UI
│       ├── constants.ts               # URLs, timing, z-indexes
│       └── supabase.ts                # Operator session
└── public/icon/                       # 16/32/48/96/128 PNGs

formyaar-backend/
├── src/
│   ├── index.ts                       # Express bootstrap
│   ├── logger.ts                      # Pino config
│   ├── types.ts                       # FormConfig types
│   └── routes/
│       ├── chat.ts                    # /ai/chat
│       ├── configs.ts                 # /configs/:form/latest
│       ├── payment.ts                 # /payment/*
│       ├── pincode.ts                 # /pincode/:pin
│       ├── operator.ts                # /operator/*
│       └── telemetry.ts               # /telemetry/event
├── configs/
│   ├── pan_card.json                  # 12-step PAN config
│   ├── passport.json                  # Stub
│   └── driving_license.json           # Stub
└── nixpacks.toml                      # Railway build config

formyaar-website/
├── index.html                         # Marketing
├── pan-form.html                      # Mobile customer form
├── user-form.html                     # Document picker
├── operator-dashboard.html            # Operator UI
├── operator-login.html                # Pre-dashboard login
├── operator-auth-callback.html        # OAuth callback
├── pay.html                           # Razorpay redirect
├── compress.html                      # Client-side compressor
├── privacy-policy.html                # Detailed policy
├── terms.html                         # Stub
├── contact.html                       # Stub
└── functions/api/
    └── submit.js                      # Pages Function: POST /api/submit
```

## Appendix B: Configuration constants

From `entrypoints/content/constants.ts`:

```typescript
BACKEND_URL = "https://formyaar-backend-production.up.railway.app"
PANEL_WIDTH = 400
PANEL_TRANSITION_MS = 300
BANNER_DELAY_MS = 1500
PULSE_INITIAL_DELAY_MS = 5000
PULSE_INTERVAL_MS = 25000
COMPLETION_AUTO_DISMISS_MS = 30000

Z_INDEX = {
  BARS: 999997,
  SPOTLIGHT: 999998,
  TOOLTIP: 999999,
  PANEL: 2147483647,
}
```

From the backend payment route: order amount `2900` paise (₹29). From the operator route: token length 12 chars, expiry 5 minutes.

---

*End of documentation. Treat this as living — update version numbers, schema columns, and known issues as the codebase evolves.*
