# FormYaar — Changelog & Build Journal

> A record of everything built, decided, and shipped.
> Started: April 27, 2026 · Founder: Hemant Chauhan (ExcuseMe Brother)

---

## The Idea

**April 27, 2026**

Hemant identified a problem he'd experienced personally — Indian government form portals (NSDL, Passport Seva, Sarathi) are confusing, poorly designed, and force people to pay local agents ₹500–₹2000 just to fill a form they could fill themselves with proper guidance.

The insight: this isn't a technology problem. It's an information asymmetry problem. The agent knows how to navigate the portal. The user doesn't. Remove that gap and you remove the agent's value.

**Original concept (rough):**
- Browser extension that overlays on government websites
- Highlights each field one by one
- Explains what to fill and why, in plain Hindi/English
- Circle-to-ask: draw around anything confusing, AI explains it
- Pre-fill user data to save time
- Step-by-step guidance across multiple pages

---

## Business Analysis

**April 27, 2026**

### Pricing research
- Local cafe charges ₹200 for PAN card
- Govt fee is ₹113 (₹113 to govt + ₹7 for prints)
- Cafe's actual cut: ₹80
- Competing online service charges ₹49 but fills form on user's behalf (trust issue)
- **FormYaar model:** User fills their own form, we just guide. Safer, cheaper, transparent.

### Final pricing decision
- ₹107 total = ₹93 (govt fee) + ₹14 (FormYaar service fee)
- Later revised to consider ₹19 flat service fee option
- Positioning: "10x cheaper than any agent, 100% from home, you stay in control"

### Market size
- 7M+ new PAN applications per year
- 3Cr+ driving license renewals/new per year
- ₹8000Cr+ informal agent market being disrupted
- 96Cr Indian internet users (2025)

### Distribution strategy
- B2C: Direct to users via Chrome Web Store, Reddit, YouTube
- B2B2C: White-label to local cafes/print shops — they charge ₹200, pay FormYaar ₹20, keep ₹180

---

## Architecture Decisions

**April 27, 2026**

### Why an extension (not a website)
A website cannot see or interact with another website due to browser sandboxing. The overlay, spotlight, and Circle-to-Ask features require a Chrome extension. Evaluated alternatives:
- Pure website → can't touch NSDL (rejected)
- Desktop companion app → too much friction (rejected)
- Extension → correct choice, risk of Chrome ban is low if built cleanly

### Self-healing config architecture
Key insight: the extension code is a one-time build. The real product is the **configs** — JSON files describing every field on every government form. These break when government sites update their HTML.

Solution:
- Configs live on backend server, not bundled in extension
- Extension fetches fresh config every session
- Fix config server-side = all users get fix instantly (no extension update needed)
- Three-layer selector fallback: CSS selector → label text → AI vision
- AI fallback discovers new selectors and saves them back to config ("self-healing")

### Cost architecture
- Layer 1: CSS selectors (free, 85% coverage)
- Layer 2: Label text matching (free, +10%)
- Layer 3: Claude Haiku vision fallback (cheap, +5%)
- User-facing chat: Claude Sonnet (quality matters here)
- Estimated cost per user session: ₹0.50–₹1
- At ₹14 service fee: ~93% gross margin

---

## Naming

**April 27–28, 2026**

Names considered:
- SarkariKaam.com (already bought)
- SarkariKagaz.com (already bought)
- Bharosa (trust) — strong emotional hook, scales beyond forms
- Kagaz Karo — "do the paperwork", very Indian, memorable
- **FormYaar** — chosen ✅

### Why FormYaar won
- Form + Yaar (friend) — name and tagline say the same thing two ways
- Matches ExcuseMe Brother brand energy
- One word, easy URL, easy to say
- "Form bharne ke liye? Bas FormYaar khol le"
- Scales to v2+ since "yaar" is about the relationship, not just forms

**Tagline:** "Your dost for every sarkari kaam"
- Chosen over "Sarkari kaam, sorted" and "Trusted Government Form Assistant"
- Works for PAN card today and municipal services in 2027
- Matches the brand warmth

---

## Logo

**April 28–29, 2026**

**Final logo:** `Form·Yaar`
- "Form" in weight 200 (light), color `#001845`
- "·" orange dot `#E8930A` (separator, slightly elevated)
- "Yaar" in weight 800 (bold), color `#001845`
- Font: Plus Jakarta Sans
- Works on light and dark backgrounds

**Brand colors:**
- Navy: `#000080` (primary)
- Saffron: `#E8930A` (accent)
- India flag colors used throughout: `#FF9933`, `#138808`, `#000080`

---

## Tech Stack Decided

**April 27, 2026**

| Layer | Choice | Reason |
|---|---|---|
| Extension | WXT + React + TypeScript | Modern MV3, hot reload, familiar stack |
| Backend | Node.js + Express | Fast to ship, familiar |
| Hosting | Railway | Already using for other projects, new Google account = $5 free credits hack |
| AI | Anthropic Claude | Haiku for cheap tasks, Sonnet for quality |
| Payments | Razorpay | India-first, UPI native |
| Landing | Cloudflare Pages | Free, instant deploy |

---

## Build Log

### April 27, 2026 — Day 1

**Extension scaffold**
- Initialized WXT project with React template
- Renamed from `wxt-react-starter` to `FormYaar` in `wxt.config.ts`
- Loaded extension in Chrome via `chrome://extensions` → Load Unpacked

**Overlay engine (v1)**
- Built spotlight effect using CSS `box-shadow: 0 0 0 9999px` technique
- Issue: spotlight applied dim to highlighted element too
- Fixed with 4-bar technique: top/bottom/left/right dark divs surrounding the element, leaving spotlight area fully clear
- Tooltip card positioned next to highlighted element
- Scroll tracking: `getBoundingClientRect()` + scroll/resize event listeners to reposition spotlight

**Overlay engine (v2)**
- Added Next button to tooltip
- Required/Optional field badges
- Next button disabled until field is filled (listens to `input` + `keyup` events)
- Warning message + shake animation when user clicks disabled Next
- Auto-advance when field already filled (green checkmark flash)

**Guide state machine** (`guide.ts`)
- `startGuide(guide, onNext, onComplete)`
- `nextField()` — advances to next field, calls completion if done
- `getCurrentIndex()`, `getTotalFields()`
- Completion message on finish

**Backend scaffold**
- Node.js + Express + TypeScript
- Routes: `/ping`, `/ai/chat`, `/configs/:form/latest`
- Deployed to Railway
- Port issue: WXT dev server also runs on 3000, changed backend to 3001 locally

### April 28, 2026 — Day 2

**AI Help button**
- Floating chatbox inside tooltip
- Backend proxy endpoint `/ai/chat` (API key never in extension)
- Background script routes requests to avoid CSP issues (content script can't directly call localhost)
- Claude Haiku responds in 2-3 sentences max
- Rate limiting planned (not yet implemented)

**Side panel (BuyHatke-style)**
- Replaced popup-driven flow with full-height side panel (400px wide, slides from right)
- Reason: user shouldn't need to click puzzle icon → popup → select form
- Panel auto-appears when user lands on supported govt site
- 6 document cards (PAN Card active, rest "Coming Soon")
- Click-outside collapses panel to tab on right edge
- Revised: removed click-outside (causes accidental collapse while filling forms)
- Added Pause button directly in tooltip instead

**Resume button**
- When user pauses: overlay hides, resume button appears on right edge
- Resume button: navy, Form·Yaar vertical text, orange step number, play icon
- Pulse animation every 25s: grows 1.4x, shakes left-right 3 times, stays big 2s, shrinks back

**Payment screen**
- Three tabs: UPI / Card / Net Banking
- Purple gradient hero with ₹107 amount
- PCI Compliant + SSL Secured badges
- Fake payment → success animation → guide starts

**Indian government UI theme**
- Navy `#000080` header with India flag SVG (Ashoka Chakra with 24 spokes)
- Tricolor strip (saffron/white/green) below header
- Saffron/green watermark gradients in background
- Form·Yaar logo in header
- Disclaimer: "Not affiliated with any government entity. FormYaar is a private service."

**Razorpay integration**
- Razorpay account created
- Business category: IT and software > SaaS
- Website: formyaar.pages.dev (Cloudflare placeholder)
- Test keys obtained: `rzp_test_SjDubVNVTRrJHI`
- Backend: `/payment/create-order`, `/payment/webhook`, `/payment/status/:id`, `/payment/verify`
- Payment flow: extension → Railway creates order → formyaar.pages.dev/pay → Razorpay → webhook → polling → guide starts
- CSP issue with Razorpay: can't inject script into page from content script
- Solution: redirect to formyaar.pages.dev/pay which has no CSP restrictions
- Polling: background script polls `/payment/status/:order_id` every 5s until paid

**Cloudflare Pages**
- Deployed placeholder at `formyaar.pages.dev`
- `pay.html` planned but not yet uploaded

### April 29, 2026 — Day 3

**Railway deployment fixes**
- Variables added: `ANTHROPIC_API_KEY`, `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`
- PORT must NOT be set manually — Railway sets it automatically
- TypeScript fix: `req.params.order_id as string` (was `string | string[]`)
- Razorpay crash fix: `dotenv.config()` must run before `new Razorpay()` instantiation

**Config system wired up**
- `fetchGuide(form)` fetches from Railway `/configs/:form/latest`
- Falls back to `TEST_GUIDE` if fetch fails
- Popup "Open FormYaar" button sends `OPEN_PANEL` message to content script
- Message listener handles: `START_GUIDE`, `OPEN_PANEL`, `STOP_GUIDE`, `PAYMENT_VERIFIED`

**Page navigation tracker**
- `MutationObserver` watches for URL changes
- `popstate` event for back/forward navigation
- On URL change: tries to find current field on new page, continues guide

**Field skip if already filled**
- `showSkipFlash()`: green border + checkmark appears briefly, then auto-advances
- Prevents re-guiding fields user already filled before guide started

**Completion flow**
- Success message with green checkmark
- Dismiss button
- F·Y tab stays visible after dismissal so user can reopen panel

---

## Investor Brief

**April 29, 2026**

6-page PDF generated covering:
1. Cover (navy, tricolor, Form·Yaar logo)
2. The Problem (agent economy, market size stats)
3. The Solution (features, user flow table)
4. Business Model (pricing table, unit economics, revenue projections to ₹1.68Cr ARR Year 2)
5. Market & Moat (TAM, competitive landscape, 4 moat pillars)
6. Team & The Ask (₹50L seed ask, use of funds)

Key projection: 20,000 users/month → ₹2.8L/month → ₹33.6L ARR at Month 12

---

## Pending (as of April 29, 2026)

### Blocking
- [ ] Upload `pay.html` to Cloudflare Pages (`formyaar.pages.dev/pay`)
- [ ] Test end-to-end payment flow
- [ ] PAN card config research (go through NSDL form with DevTools, dump all field selectors)

### Important
- [ ] Remove test sites (`trimzy.in`, `www.amazon.in`) from SITE_CONFIGS before launch
- [ ] Fix `showResumeButton` and `handlePageChange` to use fetched guide instead of hardcoded `TEST_GUIDE`
- [ ] Build formyaar.in website (landing page)
- [ ] Chrome Web Store submission (requires privacy policy)
- [ ] Privacy policy page

### Later
- [ ] Add second form (Driving License)
- [ ] Pre-fill feature (collect user data once, reuse)
- [ ] Health monitoring dashboard (which selectors are breaking)
- [ ] Self-healing AI selector saves (when AI fallback finds new selector, save back to config)
- [ ] Rate limiting on AI chat endpoint
- [ ] Real Razorpay KYC completion (needs bank account)
- [ ] B2B white-label for cyber cafes

---

## Moat Summary

1. **Config repository** — months of field-by-field research across govt forms. Competitors can copy the code in weeks, not the data.
2. **Self-healing infrastructure** — AI fallback + config server = product gets smarter from breakage.
3. **Trust architecture** — never store user documents. Structural, not just a policy.
4. **B2B2C flywheel** — cafes as distribution channel. Each cafe = volume without user acquisition cost.

---

## Brand Voice

- Direct, warm, no corporate speak
- Treats users like a knowledgeable friend, not a customer
- Mix of English and Hindi is natural ("dost", "sarkari kaam") but not forced
- "FormYaar" should feel like something your college friend built to help you out
- Error messages: helpful, never blame the user
- Never claim to be affiliated with government

---

*Last updated: April 29, 2026*
*Next milestone: PAN card config research + pay.html deployment + end-to-end payment test*
