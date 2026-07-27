# FormYaar — Changelog 3

**Session dates: May 2, 2026 (previous session) + May 3–4, 2026 (this session)**

---

## Previous Session (May 2, 2026) — Carried Over from Transcript

### Backend Bug Fixes
- Fixed `express.json()` middleware order — moved before `telemetryRouter` mount (was breaking telemetry with 400 errors)
- Fixed CORS regex `/^chrome-extension:\/\/.*/` — double-escaped backslashes were causing TypeScript error
- Fixed payment amount: `amount: 10700` → `amount: 2900` (was charging ₹107, now correct ₹29)
- Created stub configs `driving_license.json` and `passport.json` to prevent 404s
- Replaced `window.close()` on pay.html success page with informational text ("You can close this tab now") — browser security blocks programmatic close of non-script-opened tabs

### Overlay Architecture Pivot
- Abandoned the spotlight/tooltip overlay system entirely — had 6 unfixable bugs (spotlight misalignment, broken resize, tooltip covering DOB calendar, etc.)
- **Decision:** Since auto-fill makes per-field guidance unnecessary, pivoted to in-panel status display only
- Deleted `entrypoints/content/overlay.ts` and `entrypoints/content/guide.ts`
- Rewrote `entrypoints/content/index.ts` — removed all `fy:*` event listeners, MutationObserver SPA navigation, `START_GUIDE`/`STOP_GUIDE` handlers. Kept only `OPEN_PANEL` and `PAYMENT_VERIFIED`

### Auto-Fill Engine — `autofill.ts` (created)
- Built `entrypoints/content/autofill.ts` from scratch
- `runAutofill(form)` — fetches config from backend, matches step by URL substring, fills each field with 150ms delay, updates progress UI
- `fillField()` switch handles text / date / select / checkbox / radio
- `fillText()` uses native setter via `Object.getOwnPropertyDescriptor` to bypass React value tracking, dispatches input/change/blur events
- `fillSelect()` matches by `value` or `text` attribute
- `fillCheckbox()` and `fillRadio()` added
- `resolveValue()` handles `static` and `user.*` value sources, preserves `false` booleans for radio logic
- Wired `runAutofill("pan_card")` into `PAYMENT_VERIFIED` handler in `index.ts`
- **First end-to-end test: PERFECT** — all 9 fields on page 1 filled correctly

### Panel Screens
- Replaced old `renderSuccessScreen()` with `renderFillingScreen()` (spinner + progress list) and `renderVerifyScreen()` (green check + two-step instructions)
- Added exported helpers: `showFillingScreen()`, `showVerifyScreen()`, `updateFillProgress(items)`

### `pan_card.json` v2
- Added full step 2 config covering Personal Details, Contact, AO Code, Document Details (all in one DOM on `endUserLogin.html`)
- Key fields: `eVerfication4` (eKYC radio), `consentEkyc` = "Y", `appli_epan_op1_y`, residential status "R", `aoSelection`, `capacityVerifier` = "01" (Self)
- Added `fillRadio()` — only acts when `shouldSelect === true`
- Added skip-disabled-fields check in `fillField()` (eKYC mode disables Residence Address fields)

---

## May 3, 2026 — Cross-Page Persistence + AO Code + User Data Form

### Cross-Page Autofill Persistence (Major Bug Fix)

**Bug:** After page 1 submit → token page → Continue → `endUserLogin.html` loads, fresh content script had no memory of active autofill flow. Panel showed default home screen, no autofill triggered.

**Root cause discovered:** `browser.storage.session` was inaccessible from content scripts by default — Chrome restriction. Error in console: `"Access to storage is not allowed from this context"`.

**Fix — `entrypoints/background.ts`:**
- Added `chrome.storage.session.setAccessLevel({ accessLevel: "TRUSTED_AND_UNTRUSTED_CONTEXTS" })` at top of `defineBackground()` — allows content scripts to read/write session storage

**Fix — `entrypoints/content/index.ts`:**
- On `PAYMENT_VERIFIED`: sets `browser.storage.session` flag `{ autofillActive: { form: "pan_card" } }` before running autofill
- On every page load: checks flag — if set and hostname is a supported site, resumes autofill after 1500ms delay
- Wrapped storage calls in try/catch to prevent content script crash if storage unavailable
- Flag is cleared in `autofill.ts` after the last config step runs

**Result:** Full cross-page autofill now works — page 1 → token page → `endUserLogin.html` all fill correctly in sequence.

---

### `pan_card.json` v3 — Complete Rewrite

**Decisions made:**
- Source of Income checkboxes: skipped — user picks manually
- AO Area Code / Type / Range / Number: skipped — NSDL auto-fetches from PIN code
- POI/POA dropdowns: skipped — pre-set to "AADHAAR Card" by NSDL JS
- Capacity verifier: hardcoded to "01" (Self) — correct for 99% of users
- eKYC photo consent: hardcoded "Y" — use Aadhaar photo on PAN card

**New fields added (step 2):**
- `aadhaar_last_4` → `#aadhaarNo_2`
- `gender` → `#gender` (select)
- `father_first_name/middle/last` → `#faf_name`, `#fam_name`, `#fal_name`
- `mother_first_name/middle/last` → `#mof_name`, `#mom_name`, `#mol_name`
- `parent_on_card_father` / `parent_on_card_mother` → `#parent_father`, `#parent_mother` (radio, boolean)
- `isd_code` → `#tel_num_isdcode` (static "91")
- `residential_status_resident` → `#residential_status_r`
- `ao_indian_citizen` → `#aoSelection`
- `aadhaar_pin_code` → `#res_pin_code_ekyc`
- `capacity_verifier` → `#capacityVerifier` (static "01")
- `place` → `#verifierPlace`

---

### AO Code Auto-Fill

**Problem:** AO Code page (Area Code, AO Type, Range Code, AO No.) was empty — NSDL requires clicking a "Fetch" button after entering PIN code, not an onchange event.

**Fix — `autofill.ts`:**
- Added `"button_click"` to `FieldConfig` type union
- Added `case "button_click": return clickButton(el)` in `fillField()`
- Added `clickButton()` function — calls `el.click()` + dispatches `MouseEvent`
- Added dynamic delay: `field.type === "button_click" ? 2500 : 150` — waits 2.5s after Fetch for NSDL's AJAX to return

**Fix — `pan_card.json`:**
- Added `ao_fetch_btn` field with selector `#fetchAOList`
- Fixed field order: `ao_indian_citizen` (radio click) must come **before** `aadhaar_pin_code` and `ao_fetch_btn` — NSDL's `fetchAOCodesFields()` requires "Indian Citizens" to be selected first
- Final AO section order: `ao_indian_citizen` → `aadhaar_pin_code` → `ao_fetch_btn`

**Status:** PIN code fills and Fetch button clicks. AO auto-fetch still failing with "Please select State and City" — NSDL requires State/City dropdowns to be populated before Fetch works in the `#aoHide` section. **Pending fix.**

---

### User Data Collection Form

**Architecture decision:** Replaced hardcoded `TEST_USER_DATA` in `autofill.ts` with real user-provided data saved to `browser.storage.local`.

**New file — `entrypoints/content/userData.ts`:**
- `UserData` interface — 18 fields covering all PAN form requirements
- `EMPTY_USER_DATA` — default blank state
- `getUserData()` — reads from `browser.storage.local`, merges with defaults
- `saveUserData(data)` — persists to `browser.storage.local` (survives browser restarts)
- `validateUserData(data)` — returns `ValidationError[]` with field-level errors
- `STORAGE_KEY = "fy_user_data"`

**New flow:**
```
Click PAN Card → User data form → "Continue to Pay ₹29" → Razorpay
                     ↑ pre-filled if returning user
```

**Form fields (17 total, 10 required):**
- About you: first name, middle name, last name, DOB, gender, email, mobile
- Aadhaar: last 4 digits, PIN code
- Family: father's name (first/middle/last), mother's name (optional), parent on card (radio)
- Verification: place (city)

**`panel.ts` additions:**
- `renderUserFormScreen(form, data)` — full HTML form with 4 sections, inline validation, error highlighting
- `escapeHtml()` utility for XSS-safe value injection
- `showUserForm(form)` — hides all existing screens, dynamically injects form as new DOM node, attaches handlers
- `attachUserFormHandlers()` — onSubmit: validates → saves → proceeds to payment; onBack: returns to home screen
- `collectFormData()` — reads all inputs, auto-uppercases name fields, handles radio groups
- Full CSS added inside panel `<style>` block: `.fy-userform`, `.fy-userform-section`, `.fy-userform-field`, `.fy-userform-radio`, `.fy-userform-errors`, `.fy-userform-footer`, etc.

**`attachPanelEventHandlers()` updated:**
- PAN Card click now calls `showUserForm("pan_card")` instead of going directly to payment screen

**`autofill.ts` updated:**
- Removed `TEST_USER_DATA` constant
- Added `import { getUserData, type UserData } from "./userData"`
- `runAutofill()` now calls `await getUserData()` to get real user data
- `resolveValue(field, userData)` — takes `userData` as parameter, uses `keyof UserData` typing

**`FIELD_LABELS` expanded** — added labels for all 22 step-2 fields including new `ao_fetch_btn: "Fetch AO code"`

---

### TypeScript Fixes
- Added `"radio"` to `FieldConfig.type` union (was causing ts(2678) error in switch)
- Added `"button_click"` to `FieldConfig.type` union
- Fixed `resolveValue` call — `resolveValue(field)` → `resolveValue(field, userData)` (was ts(2554) — expected 2 args, got 1)
- Fixed block-scoped variable order — `const field = step.fields[i]` moved above `const delay = field.type === ...` (was ts(2448) — used before declaration)
- Removed unused `EMPTY_USER_DATA` import from `autofill.ts`

---

## Summary of Files Changed

| File | Change type |
|---|---|
| `entrypoints/background.ts` | Added `setAccessLevel` for session storage |
| `entrypoints/content/index.ts` | Cross-page persistence logic, storage flag |
| `entrypoints/content/autofill.ts` | button_click type, clickButton(), dynamic delay, getUserData(), resolveValue refactor |
| `entrypoints/content/panel.ts` | User form screen, showUserForm(), CSS, pan card click handler |
| `entrypoints/content/userData.ts` | **New file** — UserData interface, storage, validation |
| `formyaar-backend/configs/pan_card.json` | v3 — full step 2 fields, AO fetch button, correct field order |

---

## Pending

- [ ] AO Code Fetch: "Please select State and City" error — `aoSelection` radio click not registering before Fetch fires. Need delay between radio click and Fetch, or populate State/City dropdowns manually
- [ ] Test full user data form end-to-end with real data (not test data)
- [ ] Fix any remaining step 2 field mismatches found during night testing
- [ ] Panel header on user form screen is white (doesn't match navy blue of other screens) — cosmetic fix pending
