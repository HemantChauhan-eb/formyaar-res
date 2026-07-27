# CHANGELOG 9 — FormYaar

**Session date:** May 17, 2026
**Maintainer:** Hemant Chauhan (ExcuseME Brother)
**Scope:** Bug fixes (extension + website + Supabase schema), name-field pipeline refactor, strategic launch-blocker review.

This changelog covers an extended late-night debugging session that started with a single radio-button bug and ended with a full refactor of how customer name data flows from the QR mobile form through Supabase into the autofill engine. It also documents the strategic discussion of the 21 outstanding launch blockers and the decisions made about them.

---

## TL;DR

- **Fixed:** Mother radio button never being selected on PAN form step 2.
- **Fixed:** NSDL "Invalid Last Name" error appearing during QR-flow autofill (root cause was data, not a NSDL change).
- **Refactored:** Entire name-handling pipeline — mobile form, Cloudflare Pages function, Supabase schema, and extension all now use separate `first_name` / `middle_name` / `last_name` fields end-to-end. No more space-splitting hacks.
- **Fixed (the big one):** Operator-flow autofill failing on every step after the first. Root cause: customer data was stored on `window.__fy_operator_userdata`, which is destroyed on every page navigation. Moved to `browser.storage.session`.
- **Fixed:** Operator queue and review screens displaying "Unknown" / "null" for new submissions (they were still reading the deprecated `name` column).
- **Pricing correction:** B2B operator plan corrected from ₹999/month to ₹299/month across documentation.
- **Strategic decisions:** All 21 launch blockers reviewed and categorized. Critical path locked: bank account → Google OAuth → legal pages → Chrome Web Store.

---

## 1. Bugs fixed

### 1.1 Mother radio not selecting on PAN form step 2

**Symptom:** When a user picked "mother's name" for "whose name to print on PAN card", the autofill engine silently did nothing. The father radio stayed selected, the mother radio was never clicked.

**Root cause:** Two problems compounded.

1. `configs/pan_card.json` step 2 (`stepy_index: 1`) only defined `parent_on_card_father`. There was no corresponding `parent_on_card_mother` field, so the autofill engine had nothing to click.
2. The `fillRadio` function in `entrypoints/content/autofill.ts` short-circuited on `shouldSelect: false`, returning `true` without doing anything. So even if the config had been correct, the existing father radio would never have been deselected and the change event would never have fired.

**Fix:**

*`configs/pan_card.json`* — added inside `stepy_index: 1` field list, right after `parent_on_card_father`:

```json
{
  "field_id": "parent_on_card_mother",
  "type": "radio",
  "selector": "#parent_mother",
  "value_source": "user.parent_on_card_is_mother",
  "explanation": "Mother's name on PAN card."
}
```

*`entrypoints/content/autofill.ts`* — replaced `fillRadio` to support a `forceClick` flag so the change event always fires even when the radio is already in the correct state:

```typescript
function fillRadio(
  input: HTMLInputElement,
  shouldSelect: boolean,
  forceClick = false,
): boolean {
  if (!shouldSelect && !forceClick) return true;
  if (shouldSelect) {
    if (input.checked && !forceClick) return true;
    input.checked = true;
    input.dispatchEvent(new Event("change", { bubbles: true }));
    input.dispatchEvent(new Event("click", { bubbles: true }));
  }
  return true;
}
```

**Verified working.**

---

### 1.2 NSDL rejecting "Invalid Last Name" — first_name corrupted with base64 garbage

**Symptom:** During the operator-flow QR test, NSDL rejected step 1 with "Please provide valid Last Name/Surname of Applicant. Only spaces are not allowed." The DOM inspection showed `first_name` populated with `UALYZJK9E0XGMY4EK3F+PA==` — clearly base64-encoded — and `last_name` empty.

**Initial mis-diagnosis (recorded for posterity):** First hypothesis was that NSDL had added a blur-event-triggered encoder on their form fields, since the `fillText` function dispatches `blur` and the base64-looking value pointed at an encryption step. Suggested removing the blur dispatch.

**Actual root cause:** Not an NSDL change. The bug was entirely in `runAutofillFromSubmission` inside `entrypoints/content/autofill.ts`, which was doing this:

```typescript
first_name: (sub.name ?? "").split(" ")[0],
middle_name: (sub.name ?? "").split(" ")[1],
last_name: (sub.name ?? "").split(" ").slice(2).join(" "),
```

The mobile form (`pan-form.html`) was joining first/middle/last name into a single space-separated `name` field before saving to Supabase. The extension was then splitting it back on spaces. For a 2-word name like `"HEMANT CHAUHAN"`, this produced `first_name="HEMANT"`, `middle_name="CHAUHAN"`, `last_name=""` — which is exactly why NSDL rejected it.

The base64-looking garbage in `first_name` turned out to be unrelated noise from a separate test session — once the actual name-splitting bug was fixed, that artifact disappeared.

**Fix:** Full refactor of the name pipeline (see section 2 below). The split-on-space logic was deleted entirely. Names are now sent, stored, and read as discrete fields from end to end.

**Important note:** The B2C direct flow (panel autofill) was never affected by this bug because it always used `getUserData()` → `browser.storage.local`, which stores names as discrete fields. The bug was specific to the operator/QR path.

---

### 1.3 Operator autofill failing on every step after step 1 — the `window` bug

**Symptom:** After the name-pipeline refactor above, operator-flow autofill worked correctly on step 1 (personal details filled in), then failed on every subsequent step:

- Step 2: "Last 4 digits of Aadhaar Number should contain only numeric values and its length should be 4" (Aadhaar field appeared empty)
- Step 4 (AO code): PIN code field on NSDL was empty; state/city dropdowns never populated
- Step 5: "Invalid Place of verifier" — the Place field on the verification declaration was empty

The B2C direct flow continued to work flawlessly across all the same steps.

**Root cause:** The original implementation of `runAutofillFromSubmission` stored the customer submission as:

```typescript
(window as any).__fy_operator_userdata = userData;
```

…and `getUserData()` read it back:

```typescript
if ((window as any).__fy_operator_userdata) {
  return (window as any).__fy_operator_userdata;
}
return browser.storage.local.get(STORAGE_KEY); // fallback to operator's own data
```

`window` does not survive page navigation. NSDL navigates to a new HTML page between every step. The moment step 1 completed and NSDL loaded step 2, the `window` object was destroyed, `__fy_operator_userdata` was gone, and `getUserData()` silently fell back to the operator's own personal data in `browser.storage.local` — which had the operator's full 12-digit Aadhaar, the operator's PIN code, the operator's city. Hence all the validation errors specific to fields that differed between the operator and the customer.

This was the single most damaging bug in the session. Every visible step-2-onwards failure traced back to this one root cause.

**Fix:** Moved the operator submission override from `window` to `browser.storage.session`, which persists across navigations within the same tab.

*`entrypoints/content/userData.ts`* — added new storage layer:

```typescript
const OPERATOR_SUB_KEY = "fy_operator_submission";

export async function setOperatorSubmission(sub: Partial<UserData>): Promise<void> {
  await browser.storage.session.set({ [OPERATOR_SUB_KEY]: sub });
}

export async function clearOperatorSubmission(): Promise<void> {
  await browser.storage.session.remove(OPERATOR_SUB_KEY);
}
```

…and rewrote `getUserData()` to read from session storage first:

```typescript
export async function getUserData(): Promise<UserData> {
  try {
    const sessionResult = await browser.storage.session.get(OPERATOR_SUB_KEY);
    const sessionSub = sessionResult[OPERATOR_SUB_KEY] as Partial<UserData> | undefined;
    if (sessionSub && sessionSub.first_name) {
      return { ...EMPTY_USER_DATA, ...sessionSub };
    }
    const result = await browser.storage.local.get(STORAGE_KEY);
    const saved = result[STORAGE_KEY] as UserData | undefined;
    return { ...EMPTY_USER_DATA, ...(saved ?? {}) };
  } catch {
    return EMPTY_USER_DATA;
  }
}
```

*`entrypoints/content/autofill.ts`* — `runAutofillFromSubmission` rewritten to use the new storage helper, and the `(window as any).__fy_operator_userdata` line was deleted:

```typescript
export async function runAutofillFromSubmission(sub: any): Promise<void> {
  const incomeSources: string[] = (sub.income_source ?? "")
    .split(",").map((s: string) => s.trim()).filter(Boolean);

  const userData: Partial<UserData> = {
    first_name: sub.first_name ?? "",
    middle_name: sub.middle_name ?? "",
    last_name: sub.last_name ?? "",
    father_first_name: sub.father_first_name ?? "",
    father_middle_name: sub.father_middle_name ?? "",
    father_last_name: sub.father_last_name ?? "",
    mother_first_name: sub.mother_first_name ?? "",
    mother_middle_name: sub.mother_middle_name ?? "",
    mother_last_name: sub.mother_last_name ?? "",
    date_of_birth: sub.dob ?? "",
    email: sub.email ?? "",
    mobile: sub.mobile ?? "",
    aadhaar_number: "",
    aadhaar_last_4: sub.aadhaar_last_4 ?? "",
    gender: "",
    parent_on_card_is_father: true,
    parent_on_card_is_mother: false,
    aadhaar_pin_code: sub.pincode ?? "",
    place: sub.city ?? "",
    is_defence: sub.defence ?? false,
    passport_number: "",
    tin_number: "",
    proof_of_dob: sub.proof_of_dob ?? "",
    income_source: incomeSources[0] as UserData["income_source"] ?? "",
  };

  delete (window as any).__fy_operator_userdata;
  await setOperatorSubmission(userData);
  await runAutofill(sub.form_type);
}
```

**Prerequisite:** the background service worker must call `browser.storage.session.setAccessLevel({ accessLevel: "TRUSTED_AND_UNTRUSTED_CONTEXTS" })` on startup, otherwise content scripts cannot read session storage. This was already in place from previous work.

**Verified working.** All five NSDL steps now autofill correctly in the QR/operator flow.

---

### 1.4 Operator queue and review screens showing "Unknown" / "null"

**Symptom:** After the name-pipeline refactor (section 2), new submissions in the operator queue rendered as "Unknown" in the tile and "null" in the review header.

**Root cause:** Both `loadQueue()` and `showReviewScreen()` in `entrypoints/content/panel.ts` still read `sub.name`, which is now always null for new submissions because the mobile form stopped writing to it.

**Fix:** Both display points now build the display name from the discrete fields, falling back to the legacy `name` column for old test rows:

```typescript
${[sub.first_name, sub.middle_name, sub.last_name].filter(Boolean).join(" ") || sub.name || "Unknown"}
```

**Verified working.** Old test submissions (which had `name` populated) still render correctly; new submissions render correctly via the discrete fields.

---

## 2. Name-handling pipeline refactor

Replaced the fragile space-splitting design end-to-end.

### 2.1 `pan-form.html` (Cloudflare Pages — mobile customer form)

Old behavior: combined first/middle/last into a single `name` string before POSTing.
New behavior: sends each name component as its own JSON field. Same for father's name and mother's name.

Payload now includes: `first_name`, `middle_name`, `last_name`, `father_first_name`, `father_middle_name`, `father_last_name`, `mother_first_name`, `mother_middle_name`, `mother_last_name`. The combined `name`, `father_name`, `mother_name` fields are no longer sent.

### 2.2 `functions/api/submit.js` (Cloudflare Pages Function)

Old behavior: destructured the combined `name` field and inserted it into Supabase.
New behavior: destructures all nine discrete name fields, validates `first_name` (not `name`) as required, inserts them into Supabase with empty-string fallbacks for the optional middle/last fields.

### 2.3 Supabase `submissions` table

Added nine new columns (already verified present in the database at the start of the session):

```sql
ALTER TABLE submissions
  ADD COLUMN IF NOT EXISTS first_name text,
  ADD COLUMN IF NOT EXISTS middle_name text,
  ADD COLUMN IF NOT EXISTS last_name text,
  ADD COLUMN IF NOT EXISTS father_first_name text,
  ADD COLUMN IF NOT EXISTS father_middle_name text,
  ADD COLUMN IF NOT EXISTS father_last_name text,
  ADD COLUMN IF NOT EXISTS mother_first_name text,
  ADD COLUMN IF NOT EXISTS mother_middle_name text,
  ADD COLUMN IF NOT EXISTS mother_last_name text;
```

The legacy `name`, `father_name`, `mother_name` columns are kept so old test rows continue to display correctly. They will be dropped in a future cleanup once no legacy rows remain.

### 2.4 `entrypoints/content/autofill.ts`

`runAutofillFromSubmission` no longer splits a single `sub.name` field. It reads `sub.first_name`, `sub.middle_name`, `sub.last_name` directly (plus the father/mother triples). See section 1.3 for the full rewrite.

### 2.5 `entrypoints/content/panel.ts`

Queue tile and review screen heading both read the discrete fields with a legacy fallback. See section 1.4.

### Deployment order

The migration order that matters:

1. Supabase schema (already done before this session).
2. Cloudflare Pages — both `pan-form.html` and `functions/api/submit.js`.
3. Extension — autofill engine and panel display.

If extension is deployed before Pages, the extension reads NULL from the discrete columns and falls back to legacy `name` rendering, which still works for old rows. So the actual minimum-disruption order is: Supabase first, extension second, Pages last — but Pages-before-extension is also safe.

---

## 3. Strategic review of launch blockers

All 21 outstanding items were reviewed and categorized. Below is the consolidated state at end-of-session.

### 3.1 Hard blockers (cannot launch B2C without)

| # | Item | Status | Notes |
|---|---|---|---|
| 1 | Razorpay-bank account linkage | Pending | Decided to use a personal savings account in Hemant's name. Hemant is 18+, well under the ₹2.5L taxable threshold, account is solely his. Plan: switch to current account + GST registration once revenue scales beyond ₹5L/year, likely post-college. |
| 2 | Google OAuth verification | Pending | Currently in test mode; only allowlisted emails can sign in. Verification is free but requires domain + privacy policy + terms. Only basic email/profile scopes are used (non-sensitive). Used exclusively for operators — B2C uses local storage. |
| 9 | Legal pages | Partial | Privacy Policy is comprehensive and DPDP-aware. Terms page is a stub with only the refund clause; needs full ToS expansion. |
| 8 | Chrome Web Store submission | Blocked | Blocked on all of the above. No work to do here until 1, 2, 9 are complete. |
| 5 | Subscription tracking | Pending | `operators.subscription_status` column exists but no system reads or writes it after dashboard upsert. No B2B subscription purchase flow on the website yet. |
| 6 | Razorpay on website for B2B subscriptions | Pending | Same root cause as #5 — no subscription purchase UI built. |

### 3.2 Functional gaps

| # | Item | Status | Notes |
|---|---|---|---|
| 11 | AO codes for 30+ cities | Open | Current implementation uses `api.postalpincode.in` for state/city lookup. ~30 cities have NSDL-specific formatting issues that need explicit hardcoded mapping. |
| 3 | Back buttons on 2 panel screens | Open | Small UI fix. |
| 17 | Suppress panel on `formyaar.pages.dev/pay` | Open | The content script currently runs on the payment redirect page and shows the panel, which is jarring during checkout. |
| 16 | "Add to Chrome" link broken on website | Open | The hero CTA on `index.html` doesn't point to a working Chrome Web Store URL (because the extension isn't published yet). |
| 13 | Form auto-formatting in panel | Open | Date masking, etc. The collection form accepts free-form input. |
| 12 | Defence personnel config path | Stubbed | The `defence_selector` mechanism exists in `autofill.ts` and `pan_card.json` references it, but the full flow has not been tested. |

### 3.3 Polish

| # | Item | Status | Notes |
|---|---|---|---|
| 4 | Cafe operator entry button styling | Open | Currently a plain underlined text link at the bottom of the home screen. Should be a proper card or button. |
| 10 | Improve `contact.html` | Open | Currently minimal (phone + email + raw HTML). Needs to match the design language of `privacy-policy.html`. |
| 18 | Payment / form history dashboard | Open | Currently there is no place for a B2C user to see past form-fill history. |
| 19 | Enable + optimize Supabase RLS | Open | RLS is currently disabled on `submissions` (marked `UNRESTRICTED`). Needs policy `auth.uid() = operator_id` for reads, plus an anon-insert policy for the mobile form to write submissions. |
| 14 | Buy `formyaar.in` domain | Open | ~₹600–900/year via Hostinger. Domain is a prerequisite for Google OAuth verification. |
| 7 | Website inconsistency (dummy numbers) | Open | Some pages still show placeholder phone numbers / stats. |
| 20 | Font inconsistency across pages | Open | Some pages use Plus Jakarta Sans, some use Noto Sans, etc. Need a single typography system. |
| 15 | Code structure improvements | Open | Panel UI is 1500 lines of vanilla DOM in template literals; long-term should migrate to React-in-Shadow-DOM. |

### 3.4 Critical-path lock-in

The agreed order of operations is:

1. **Open Razorpay-linked savings account** (no dependency).
2. **Buy `formyaar.in` domain** (~₹600–900).
3. **Complete legal pages** — expand Terms; Privacy Policy is already done. Update all email/contact references to use `@formyaar.in`.
4. **Submit Google OAuth verification** with domain + legal pages.
5. **Submit Chrome Web Store listing.**
6. While verifications are pending, work through functional gaps (#11, #3, #17, etc.).

No work on subscription tracking (#5, #6) or polish items will start until the critical path is unblocked.

---

## 4. Pricing correction

Persistent memory previously recorded B2B pricing as ₹999/month. Corrected to ₹299/month, matching the website's actual pricing card. This price point is intentionally aggressive — the agreed strategy is to undercut typical cafe per-form pricing (₹200–500) so that a single cafe pays back the subscription with one customer per day.

The B2C price of ₹29 per form is unchanged.

---

## 5. Resume flow — feature audit

While debugging, the resume-from-paused-payment flow was audited. Confirmed working:

- `userData.ts` exports `ActiveSession`, `getActiveSession`, `setActiveSession`, `markSessionCompleted`, `clearActiveSession`.
- The session is keyed in `browser.storage.local` under `fy_active_session` and survives browser restart.
- `index.ts` checks for an incomplete session on every supported-site page load and calls `showResumeScreen()` from `panel.ts` if one exists.
- `panel.ts` renders the resume screen with two buttons: "Start Filling Now" (sets `autofillActive` and navigates to NSDL) and "Start over" (clears the active session and returns to home).

Known gaps (documented, not yet fixed):

- `markSessionCompleted()` only fires when the user reaches the upload-help screen, i.e. the final step. A user who fills steps 1–10 but never reaches the upload step gets stuck in a permanent resume-screen loop.
- The two resume-screen buttons attach event listeners every time the screen is shown, with no removal — minor memory leak over a long-running tab.
- No timestamp expiry: a paid session resumes forever, even months later. Should auto-clear after, say, 7 days.

---

## 6. Files touched in this session

### Extension (`HemantChauhan-eb/formyaar-extension`)

- `entrypoints/content/autofill.ts` — `fillRadio` rewritten; `runAutofillFromSubmission` rewritten; `window.__fy_operator_userdata` removed.
- `entrypoints/content/userData.ts` — added `setOperatorSubmission`, `clearOperatorSubmission`; rewrote `getUserData` to prefer session storage.
- `entrypoints/content/panel.ts` — queue tile and review screen now use discrete name fields with legacy fallback.

### Website (`HemantChauhan-eb/formyaar-website`)

- `pan-form.html` — payload refactored to send discrete name fields.
- `functions/api/submit.js` — destructures and inserts discrete name fields.

### Backend (`HemantChauhan-eb/formyaar-backend`)

- `configs/pan_card.json` — added `parent_on_card_mother` radio field.

### Supabase

- `submissions` table — discrete name columns confirmed present (added in prior session). No new schema changes in this session.

---

## 7. Verification matrix

| Test case | Status | Notes |
|---|---|---|
| B2C direct flow — PAN form steps 1–5 | ✅ Working | Unchanged; never broke. |
| B2C — mother on card radio selection | ✅ Working | Fixed in 1.1. |
| B2C — resume after payment + tab close | ✅ Working | Tested. |
| B2B — QR scan → mobile form submit | ✅ Working | Verified row inserts to Supabase with discrete name fields. |
| B2B — operator queue display | ✅ Working | New submissions render correct name; old submissions fall back to `sub.name`. |
| B2B — operator review screen | ✅ Working | Same fallback logic. |
| B2B — Accept & Fill → step 1 autofill | ✅ Working | All discrete fields populate correctly. |
| B2B — step 2 (Aadhaar last 4 + names) | ✅ Working | Was broken; fixed by 1.3. |
| B2B — step 3 (contact + income) | ✅ Working | Was broken; fixed by 1.3. |
| B2B — step 4 (AO code + PIN) | ✅ Working | Was broken; fixed by 1.3. |
| B2B — step 5 (Place + declaration) | ✅ Working | Was broken; fixed by 1.3. |
| Income source multi-select | ⚠️ Partial | Mobile form supports multi-select; `UserData` type still single-value. Currently picks the first one only. Logged as architecture debt. |
| Defence personnel flow | ❓ Untested | Config path exists but not exercised in this session. |

---

## 8. Lessons learned

A few principles worth keeping for future debugging sessions:

1. **When two flows share a code path but only one breaks, the bug is in the part of the code where they diverge.** The QR flow and the B2C flow both end at `runAutofill()`, but the data path before `runAutofill` differs. Once that was clear, the bug was forced into a narrow region.

2. **`window` is not durable storage for MV3 content scripts.** It is destroyed on every navigation. Anything that needs to persist across NSDL's step transitions must live in `chrome.storage` (session or local), period.

3. **Mis-diagnoses are cheap if you don't act on them.** The blur-event hypothesis for the "base64 garbage in first_name" symptom was wrong, but recognizing it was wrong took five minutes of testing instead of an hour of code changes. Always test the hypothesis against the simplest possible reproduction before editing code.

4. **Schema-and-code refactors need migration order discipline.** The name-pipeline change touched four artifacts (mobile form, Pages function, Supabase, extension). Doing them in the wrong order causes either NULLs or dropped writes. Always: database first, write path second, read path last.

---

*End of CHANGELOG 9. Next session focus: critical-path execution — bank account, domain purchase, terms page expansion, then OAuth verification.*
