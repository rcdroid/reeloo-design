# Design request: Expense mobile app — transaction/invoice views + camera receipt-capture + self-host server URL

**Status:** open
**Filed:** 2026-05-28
**Author:** orchestrator (user-initiated)
**Surface in scope:** a NEW mobile app (Flutter — there's a scaffold at `~/work/tools/depot-app/`) for the personal-finance expense tracker. Companion to the existing web app at `~/work/finance/expense` (routes `/transactions`, `/breakdown`, `/spending`, `/classifier`). Same backend, same Postgres, same `/api/*` endpoints.

**Trigger (user, verbatim):**

> create a mobile app design for this app for the same feature, write product spec as prompt to claude design. plus the feature to use camera to take snapshot of invoice and can import to our backend db, allow user to override server url so that they can connect to their own backend server and db.

**Read first:** `requests/15-expense-purchase-view-not-transaction-view.md` — defines the Transaction-as-unit / Invoice-as-expander data model that this mobile app renders. Everything in §5–6 of #15 applies; this request is the mobile expression of it plus two net-new mobile-only features (camera capture, server-URL override).

---

## 1. Why a mobile app

The web app answers "where did my money go" at a desk. The phone is where two things actually happen:

1. **You're holding a paper receipt** right after a purchase (market stall, restaurant, a shop that isn't Lidl/Coop and has no digital receipt API). The phone camera is the only practical capture path.
2. **You want to glance at this-week spend** while standing in a checkout line deciding whether to buy the thing.

The mobile app is camera-first for input, glance-first for output. It is NOT a port of the dense web tables.

## 2. Platform + constraints

- **Flutter** (scaffold exists at `~/work/tools/depot-app/lib/`). iOS + Android from one codebase. The whole household is on iOS, so iOS is the primary target; Android is a bonus.
- Talks to the same backend over HTTPS. Auth via `X-App-Token` header (same token the web app + depot use).
- Offline-tolerant: captured receipts queue locally and sync when connectivity returns (you photograph receipts in shops with bad signal).
- Themes: same 3 as web (`vanilla` / `cobalt-dark` / `high-contrast`), but mobile-tuned — dark is the likely default on a phone.

## 3. The five screens

### 3.1 Home / "This period" (glance-first)

The landing screen. A period selector at top (This week / This month / Custom), then:

- One big number: total spend in the period, in CHF (user's base currency)
- A compact category strip (horizontal scroll of chips: Groceries 4,666 · Rent 2,430 · Insurance 1,605 …) — tap a chip → filtered transaction list
- A "vs last period" delta (↓ 23%) with a tiny sparkline
- A prominent floating camera button (the primary action — see 3.3)

**Open design question:** what's the single most useful number to show first? Total period spend, or "remaining vs a budget", or "today so far"? Pick one as the hero, justify it. This is a glance screen — if it takes more than 2 seconds to parse, it failed.

### 3.2 Transactions list (the #15 model, mobile)

A scrollable list, one row per cash event (payment-method transaction). Per #15:

- Each row: date · merchant (alias-resolved) · amount · payment-instrument badge
- Rows with a linked invoice get an expand affordance → tap to reveal item lines inline (5× eggs, 3× hams, 2× milk)
- Filters live in a bottom-sheet (tap a filter icon): Merchant (multi-select chips), Paid-with (multi-select), Category, Date range, Currency
- Pull-to-refresh re-syncs from backend
- Tap a row (not the expand caret) → transaction detail screen (edit category, add memo, add tag, split, merge, mark-fraud)

**Open design question:** on a phone the web's 6-column table doesn't fit. What's the right row anatomy for a 375pt-wide screen? Two-line rows (merchant + amount on line 1, date + payment + category on line 2)? Where does the expand caret + item-count badge sit so it's tappable but not crowding the amount?

### 3.3 Camera receipt capture (NET-NEW — the headline feature)

Tap the camera FAB anywhere → camera opens in receipt mode:

1. **Capture**: live edge-detection overlay (like a document scanner) auto-crops the receipt. Multi-shot for long receipts (Costco) — stitch or multi-page.
2. **Preview + confirm**: show the captured image, let user retake.
3. **Auto-extract** (OCR happens backend-side; app just uploads the image): the app POSTs the photo to a new backend endpoint `POST /api/import/receipt-photo` (multipart image). Backend runs OCR (Gemini multimodal via the gateway, or Textract) → returns a parsed draft: merchant, date, total, line items, currency.
4. **Review draft**: the extracted fields come back and the user sees an editable form — merchant (with alias autocomplete), date, total, category (auto-suggested via the classifier), line items (each editable). This is the critical screen: OCR is never perfect, so the review-and-correct UX is the whole ballgame.
5. **Save**: confirmed receipt → becomes an Invoice row in the backend; the linker tries to match it to an existing card/bank Transaction (per #15). If it matches → linked. If no statement leg exists yet (cash purchase) → it stands alone as a cash Transaction.
6. **Offline**: if no connectivity, the photo + a local draft queue in a "pending uploads" tray; a badge shows the count; auto-flush on reconnect.

**Open design questions for Claude:**
- The capture screen: how much chrome? A pure camera viewfinder with a single shutter + an edge-detect rectangle, or guidance text ("align receipt within frame")?
- The review-draft screen: how to show OCR confidence? Highlight low-confidence fields in amber so the user knows what to double-check? How to make correcting a misread total feel like one tap, not a form-filling chore?
- Long receipts (Costco, 60 items): paginated capture? How does the review screen handle 60 line items on a phone — collapsed by default with a total, expand to verify?
- The pending-uploads tray: where does it live? A badge on the camera FAB? A banner on Home?

### 3.4 Capture-to-category flow

After OCR extracts the merchant, the app should auto-suggest a category via the existing `/api/classifier` logic (e.g. "Coop" → Groceries at 100% confidence). The user confirms or overrides with one tap. Overrides feed back to the classifier (same as web `/classifier` learning loop).

### 3.5 Settings — including server-URL override (NET-NEW)

A settings screen with:

- **Server URL** (the headline of this request): a text field where the user can override the backend base URL. Default points at the user's known host (e.g. `https://expense.reeloo.ai` or the tailnet `http://minis:3300`). The override lets anyone self-host the expense backend and point their app at their own server + DB. Must validate the URL (reachability ping to `/api/health`) and show a clear connected/disconnected state.
- **App token**: a secure field for the `X-App-Token` (stored in the OS keychain / secure storage, never plaintext). This is how a self-hoster authenticates to their own backend.
- **Base currency** display preference
- **Theme** (vanilla / cobalt-dark / high-contrast)
- **Default period** for Home (week / month)
- **Biometric lock** (Face ID / fingerprint to open the app — it's financial data)

**Open design questions:**
- The server-URL + token entry: this is a power-user / self-hoster feature. How to present it without scaring a normal user? A collapsed "Advanced / Self-hosting" section? An onboarding path that asks "Use Reeloo cloud or your own server?" on first launch?
- Connected-state feedback: a green dot + "Connected to minis:3300" line? What does the failure state look like (wrong URL, bad token, server down) — three distinct error messages, not one generic "failed"?

## 4. The new backend endpoint (out of design scope, noted for context)

`POST /api/import/receipt-photo` — multipart image → OCR → returns parsed draft JSON `{merchant, date, total, currency, items[], confidence{}}`. The app renders the draft for review (3.3 step 4), then on confirm POSTs the corrected data to `/api/import/retail` (which already exists) or a new `/api/invoices` endpoint. I'll build these; the design just needs to assume the draft-review contract exists.

## 5. What I want back from Claude design

1. The five screens at production density, iOS + Android, all three themes (at least Home + camera-review in all three; others in dark only is fine).
2. The camera capture flow as a connected sequence (viewfinder → preview → OCR-review-draft → saved), with the OCR-confidence treatment.
3. The transaction list row anatomy for narrow screens + the filter bottom-sheet.
4. The settings screen including the server-URL / token / self-hosting section, with connected/error states.
5. The offline pending-uploads pattern.
6. The empty states: no transactions this period, camera-permission-denied, server-unreachable, OCR-failed-to-parse.
7. A short note on navigation model: bottom tab bar (Home / Transactions / Camera / Settings) vs camera-as-FAB-over-everything. Recommend one.

Density bar: this is a tool the owner uses daily, not a consumer growth app. "Faster than opening the web app on my phone" is the win. No onboarding carousel, no gamification, no empty-state mascots. Calm, fast, glanceable.

## 6. Reference material

- Data model + filter semantics: `requests/15-expense-purchase-view-not-transaction-view.md` §5–6
- Web app screens to echo (not copy): `~/work/finance/expense/src/app/{transactions,breakdown,spending,classifier}/page.tsx`
- Flutter scaffold: `~/work/tools/depot-app/lib/`
- Backend endpoints already live: `/api/transactions` (supports `excludeRetailReceipts`, `source`, `keyword`, `startDate`/`endDate`), `/api/transactions/:id/invoice-items`, `/api/import/retail`, `/api/breakdown?start&end`, `/api/classifier/suggestions`
- Auth: `X-App-Token` header (same token used by depot at :18100 and the receipt-scrape daemon)
- Theme tokens: same `--text-primary` / `--card-bg` / 3-theme system as Reeloo v2 (§6.2 of v2/06)
