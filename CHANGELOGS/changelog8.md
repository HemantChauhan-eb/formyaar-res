# FormYaar Changelog

## Session FY-TECH-8

### Architecture & Security

- **New operator authentication system** — replaced Supabase OAuth token flow with a backend-generated short-lived token system. Operators generate a 5-minute one-time token from the dashboard and paste it into the extension. Extension stores operator session in local storage permanently (like YouTube) — no re-auth needed unless manually signed out or reinstalled.
- **Removed Supabase client from extension entirely** — extension no longer has any direct Supabase dependency. All data flows through the FormYaar backend (Railway).
- **New `operator_tokens` Supabase table** — stores generated tokens with expiry, used_at timestamp, and operator_id reference. Tokens are single-use and expire in 5 minutes.
- **Service role key added to Railway** — backend now uses Supabase service role key for operator operations, correctly bypassing RLS for server-side writes.

### Backend (`formyaar-backend`)

- **New file: `src/routes/operator.ts`** — four new endpoints:
  - `POST /operator/generate-token` — generates a 12-char alphanumeric one-time token for an operator, stores it with 5-min expiry
  - `POST /operator/verify-token` — verifies token, marks it as used, returns operator session data
  - `GET /operator/queue/:operator_id` — fetches pending submissions for an operator (replaces direct Supabase realtime from extension)
  - `PATCH /operator/submission/:id/status` — updates submission status (filling/rejected/completed/pending)
- **CORS updated** — added `PATCH` to allowed methods in `src/index.ts`
- **Installed `@supabase/supabase-js`** in backend dependencies
- **Lazy Supabase client** in `operator.ts` — client created at request time via `getSupabase()` function instead of module load time, fixing Railway env var timing issues

### Dashboard (`formyaar-website`)

- **New Extension Token section** on operator dashboard — auto-generates token on page load, shows 5-minute countdown timer, Copy Token button, Regenerate Token button
- **Removed old "Copy Token to Connect Extension" card** — the OAuth-based token flow is fully replaced
- **QR color changed** from navy (`#000080`) to black (`#000000`)
- **Nav logo now links to homepage** — added `<a>` tag with `color: inherit; text-decoration: none` fix
- **Google account picker forced** — added `prompt: "select_account"` to OAuth options so operators always see account selection instead of auto-signing in with last account
- **Operator queue refresh button** — added refresh icon button in operator queue header with spin animation on click

### Extension (`formyaar-extension`)

- **`supabase.ts` rewritten** — removed Supabase client entirely, now uses backend REST endpoints for all operator operations. Operator session stored in `browser.storage.local` as `fy_operator_session`
- **Operator login screen redesigned** — removed OAuth Google button, now shows clean token input field (12-char, monospace, uppercase, centered) with "Connect Extension" button and specific error messages for expired/already-used tokens
- **`panel.ts` operator auth updated** — removed all `supabase` imports and direct Supabase calls, queue fetch and submission status updates now go through backend
- **`showReviewScreen` accept/reject** — both buttons now call `PATCH /operator/submission/:id/status` instead of direct Supabase writes
- **Removed Supabase realtime subscription** from `loadQueue` — replaced with manual refresh button
- **TypeScript errors fixed** — added explicit `: any` types to `sub` and `s` parameters in `loadQueue` map and find callbacks

### Website (`formyaar-website`)

- **`user-form.html` split into two files**:
  - `user-form.html` — now only contains the form type selector (PAN Card, Aadhaar, DL, Voter ID, Passport, Visa cards). Navigates to the relevant form file on selection, passing `?op=` param.
  - `pan-form.html` — new file containing the full PAN card details form. Reads `?op=` from URL, submits to `/api/submit`, shows success screen with reference ID. Back button returns to `user-form.html` preserving operator ID.
- **Guard added** in `pan-form.html` — if `?op=` param is missing, shows an error screen instead of submitting with null operator ID (fixes bug where all submissions were going to all operators)
- **Removed `flat` and `street` fields** from `pan-form.html` — address fields that were collected but never used in autofill or stored in Supabase. NSDL address is populated from Aadhaar via eKYC anyway.
- **Aadhaar field changed from 12-digit to 4-digit (last 4)** across:
  - `pan-form.html` — label, placeholder, maxlength, validation, input formatter, payload key renamed to `aadhaar_last_4`
  - `functions/api/submit.js` — added `aadhaar_last_4` to destructure and Supabase insert payload
  - `entrypoints/content/autofill.ts` — `runAutofillFromSubmission` now maps `sub.aadhaar_last_4` to `userData.aadhaar_last_4`
  - `configs/pan_card.json` — removed `aadhaar_first_8` field from step 6 (operator fills manually, full Aadhaar not stored server-side for legal compliance)
- **Supabase SQL** — added `aadhaar_last_4 TEXT DEFAULT ''` column to `submissions` table

### Legal / Compliance

- **Full Aadhaar number removed from operator flow** — storing 12-digit Aadhaar without UIDAI KUA registration violates Section 29 of the Aadhaar Act 2016. System now collects and stores only last 4 digits (explicitly permitted). Step 6 first-8 digits filled manually by operator. Privacy policy remains accurate.

### Bug Fixes

- **Bug: Operator QR not scoped** — fixed by adding `?op=operator_id` guard in `pan-form.html`. Previously null `operator_id` caused submissions to appear in all operator queues.
- **Bug: Token sign-in not working** — root cause was Supabase access token expiry (1 hour). Fixed by replacing the entire token mechanism with backend-issued tokens that don't expire on Supabase's schedule.
- **Bug: Google auto-signin** — fixed by adding `prompt: "select_account"` to OAuth config, forcing account picker every time.
- **Bug: Railway crash on startup** — `SUPABASE_SERVICE_ROLE_KEY` was not set in Railway environment variables. Was accidentally using the publishable/anon key. Added correct service role key.
