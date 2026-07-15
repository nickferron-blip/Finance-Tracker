# Finance Tracker — Project Handoff

A personal finance tracking Progressive Web App (PWA), built as a single self-contained HTML file, hosted free on GitHub Pages, syncing data through Google Drive.

**Live app:** check GitHub Pages settings on the repo below for the URL.
**Repo:** https://github.com/nickferron-blip/Finance-Tracker
**Local path:** `/Users/nicolasferron/Library/CloudStorage/OneDrive-Mely.ai/Claude Main Folder/Apps/finance-tracker`

---

## 1. File structure

```
finance-tracker/
  index.html      ← the entire app (HTML + CSS + JS, no build step)
  manifest.json    ← PWA manifest (icons, theme color, install behavior)
  sw.js            ← service worker (offline caching)
  logo.png         ← app logo, background made transparent via a Python/PIL script
  icons/
    icon-192.png
    icon-512.png
```

There is no build process. Every change is a direct edit to `index.html`, followed by:

```bash
cd "/Users/nicolasferron/Library/CloudStorage/OneDrive-Mely.ai/Claude Main Folder/Apps/finance-tracker"
git add index.html
git commit -m "description of change"
git push
```

GitHub Pages redeploys automatically, usually within 1–2 minutes. The user then hard-refreshes (Cmd+Shift+R) to see changes.

---

## 2. Data model

### Categories (parent → sub-category hierarchy)
- Defined in `DEFAULT_CATEGORIES` — parents use IDs like `p1`–`p16`, sub-categories use `c1`–`c39`
- Each category has: `id`, `name`, `type`, `color`, `parentId` (`null` for parents)
- Sub-categories **inherit their parent's color** automatically (enforced in the category modal — color picker only shows for top-level categories)

### Four category/transaction types
Defined in `TYPE_META` / `TYPE_ORDER`:
- `expense` (red) and `income` (green) — these are the only two that count toward the Dashboard's Income / Expenses / Net Balance numbers
- `savings` (blue) and `investment` (purple) — envelope/set-aside money. Tracked in their own Budget page sections, but **excluded** from Net Balance on purpose (per explicit user request — these represent money moved to another account, not "spending")
- Editing a category's type via the Categories page **automatically retags every transaction** already assigned to it, so budget totals don't silently break

### Budgets
- Stored in `state.budgets`, keyed by category ID — budgets are normally set at the **sub-category** level
- A parent category's "budget" and "spent" are **always a rollup** of its sub-categories (parents can't have their own independent budget — this was an explicit design decision)
- `netCatAmount(catId, type, txns)` nets all transactions assigned to a category **regardless of the transaction's own type** — so a refund/reimbursement (e.g. an `income`-type transaction under an otherwise `expense` category like "Wimgym") correctly offsets spending instead of vanishing or double-counting. This function is shared between the Dashboard stat cards and the Budget page — don't duplicate it.

### Transactions
- Each has: `id`, `amount` (always positive), `date`, `description`, `categoryId`, `notes`, `type`
- Transactions page supports fully inline editing: click the category badge to get a dropdown, click the type badge to cycle through all 4 types, edit notes directly in a text field under the description — everything auto-saves

### Auto-categorization
- `autoCategorize(description, type)` checks, in order: exact learned rule → keyword learned rule → built-in `KEYWORD_MAP` (Quebec/Canadian merchant keywords) → falls back to **Miscellaneous** (not the first category — this was a deliberate fix; it used to default to Mortgage)
- `learnRule()` is called whenever a user manually changes a transaction's category, building up `state.rules` over time

---

## 3. Persistence — Google Drive + IndexedDB

```js
const CLIENT_ID = '168405386494-tl577olfbs8sjln89q5rm97jn3euimi7.apps.googleusercontent.com';
const SCOPES = 'https://www.googleapis.com/auth/drive';   // full drive (shareable data file)
const FILE_NAME = 'finance-tracker-data.json';
const IMPORT_FOLDER_NAME = 'Finance Tracker Imports';
```

- **Main data file** is now a **regular, visible** file `finance-tracker-data.json` in the user's My Drive (NOT the old hidden `appDataFolder`). This change was made so two people (a couple) can share one dataset with **separate** Google logins.
  - `_findVisibleFile()` searches all accessible files by name (includes files shared-with-me by a partner), newest first.
  - `_loadFromDrive()`: use the visible file if found; else migrate once from the legacy `appDataFolder` file (`_findAppDataFile`) into a new visible file (`_createVisibleFile`, multipart upload); else create fresh. The legacy appdata file is left intact as a backup.
  - `_saveToDrive()` PATCHes the visible file by `_driveFileId`.
- **Shared-data setup the user performs once**: (1) add full `.../auth/drive` scope in Google Cloud Console → OAuth consent → Data Access; (2) add partner as a test user; (3) sign out/in to migrate; (4) in drive.google.com, share `finance-tracker-data.json` with the partner as **Editor**. Partner then signs in with their own account and the app finds the shared file.
- **Concurrency**: last-write-wins. Mitigation only — a `visibilitychange` listener reloads from Drive when the tab regains focus (skipped if a save is pending or a modal is open). No real merge; simultaneous edits can clobber. User accepted this tradeoff.
- **IndexedDB** is a local cache so the app works instantly offline and doesn't wait on network before rendering.
- Save is debounced (800ms) and writes to both IndexedDB and Drive.
- On load: try silent token refresh first (no popup) → fall back to IndexedDB cache → show sign-in overlay if nothing works.
- Google Cloud Console project is in **Testing** publishing status with the user (and now partner) added as test users — full `drive` is a "restricted" scope but Testing mode skips verification (users see an "unverified app" warning to click through).

---

## 4. Design system

- Font: Inter (Google Fonts CDN)
- Colors: purple (`--accent: #5b3fce`), copper (`--copper: #b5763a`), white/near-white surfaces
- Icons: inline SVG (Lucide-style stroke icons), not an external icon library — a CDN-loaded Lucide script was tried first but had timing/init issues, so icons were inlined directly as SVG markup instead
- Budget page: circular progress rings per sub-category (`budgetRingSVG()`), grouped under parent category headers, with category-specific icons chosen by keyword-matching the category name (`getCategoryIcon()`)

---

## 5. Auto-import from Drive folder (the "screenshot workflow")

This was built to support a manual-but-low-friction way to get bank transactions in without a backend or Plaid integration. The intended workflow:

1. User takes a screenshot of Apple Wallet's "Latest Transactions" list for their card (Apple Wallet shows this for RBC cards specifically, not just Apple Card)
2. User shares that screenshot with Claude in a chat
3. Claude reads the image and writes a correctly-formatted CSV into a Drive folder named exactly **"Finance Tracker Imports"** (must be a normal, visible folder — anywhere in the user's Drive, name match is exact)
4. The app scans that folder automatically on every sign-in (and via a "Check now" button on the Import CSV page), imports any new CSV, and dedupes against existing transactions

### CSV format Claude should write to that folder
```
Date,Description,Amount,Type
2026-07-04,Starbucks,5.75,expense
```
- `Date` — YYYY-MM-DD
- `Amount` — always a positive number
- `Type` — optional, defaults to `expense` if omitted or unrecognized

### Key functions
- `_findImportFolderId()` — searches Drive by folder name
- `parseAutoImportCSV(text)` — parses the fixed CSV format (header-name matching, not fixed column positions)
- `scanImportFolder(silent)` — the main entry point; tracks processed file IDs in `state.importedFileIds` so files aren't re-imported even without delete permission on the folder
- Called automatically after sign-in (`signIn()` and `loadState()`), plus a manual "Check now" button

### Outstanding setup steps (as of this handoff)
1. User needs to add the `drive.readonly` scope in Google Cloud Console (see section 3)
2. User needs to check/authorize a Google Drive connector in their Claude account settings (claude.ai → Settings → Connectors) so Claude can write files into their Drive directly
3. User needs to create a folder named exactly **"Finance Tracker Imports"** in their regular Google Drive
4. Next app sign-in after these changes will trigger a re-consent screen (new scope) — user needs to approve it

---

## 5b. Cashflow forecast (added after initial handoff)

- **Data**: `state.recurringItems` — expected charges, each `{id, description, categoryId, amount, type, schedule: {kind, startDate, customInterval?, customUnit?}, active}`. Schedule kinds: `once`, `weekly`, `biweekly`, `bimonthly_1_15` (1st & 15th), `bimonthly_15_last` (15th & last day), `monthly`, `yearly`, `custom` (every N days/weeks). `state.balanceAnchor = {amount, date}` — user's real bank balance, set/reset via "Set balance" on the Dashboard.
- **Forecast-only — a firm rule**: expected charges NEVER create transactions. Real transactions come from imports/manual entry. An earlier version auto-posted transactions; that was explicitly removed (`processRecurringItems` is now a no-op kept for compatibility).
- **Consumption model** (`expectedOccurrences()` in index.html): per category, this month's expected occurrences are consumed chronologically by the category's actual net spend this month. Exact charges (mortgage) vanish when the real transaction lands even if amounts differ slightly; estimates (weekly groceries) shrink as real spending eats the earliest occurrences. Unconsumed past-dated occurrences surface as due "today" (conservative).
- **Also firm**: expected charges are set *from* the Budget page (a "Plan" button on each budget circle) for convenience, but they don't touch budget math at all — budgets remain purely actual-vs-budget.
- **Plan modal** (`openRecurringModalForCat`): the only management UI (the standalone Cashflow nav page was removed at user request). It lists the category's existing plans (edit/pause/delete) and has two add modes: **"Scheduled charge"** (fixed amount + schedule; `mode: 'fixed'`) and **"Split remaining per week"** (`mode: 'spread'` — no amount stored; the forecast dynamically spreads the category's remaining budget (budget − spent) over the weeks left in the month, one spread max per category, requires a budget to be set).
- **Dashboard card** ("Upcoming Cashflow", first card): two columns — "Until next payday" (next payday = earliest unconsumed income occurrence) and "Until end of month" — each listing expected charges with a running projected balance per row (from the balance anchor) and the ending residual in the column header. `currentBalance()` = anchor + income − (expense/savings/investment) for transactions dated after the anchor.

## 6. Live bank sync — research already done, no code built yet

The user asked about pulling transactions live from RBC. Researched and ruled out:

- **Plaid** — real option, but requires a backend (secret keys can't live in client-side JS on GitHub Pages) plus a Plaid developer account and Production approval. RBC has a formal OAuth partnership with Plaid (since 2022) specifically to avoid the old 2FA/screen-scraping breakage — so Plaid-RBC should work smoothly via OAuth today. This is the most "real" path if the user wants true live sync, but is a meaningfully bigger project (new backend, e.g. Vercel functions).
- **Mastercard/Finicity** — direct Plaid competitor (Mastercard acquired Finicity), also supports RBC, same backend requirement, no meaningful advantage for this use case over Plaid.
- **RBC scheduled/automated CSV export** — doesn't exist; RBC has no personal-banking export API. Automating via login-scraping was explicitly advised against (2FA will block it, and repeated automated logins risk RBC's fraud detection locking the account).
- **Apple Wallet / Google Wallet APIs** — Apple's `FinanceKit` API (launched 2024) only exposes Apple Card / Apple Cash / Savings with Apple transactions — **not** regular bank cards used via Apple Pay, and it's US-only. iOS Shortcuts' "notification from app" automation trigger does not list Wallet as a selectable source (confirmed by direct testing — Wallet's transaction notifications are generated by a system-level process, not a normal app notification), so there's no Shortcuts-based workaround either.
- **Conclusion for now:** the screenshot-to-Claude-to-Drive-folder workflow (section 5) is the agreed "stay simple" solution. Plaid remains on the table if the user later decides true live sync is worth building a backend for.

### Also discussed: native app packaging (App Store / Play Store)
Not built. If revisited: the recommended path is wrapping the existing app with **Capacitor**, pointed at the live GitHub Pages URL (so updates ship instantly without app store re-review). Requires Apple Developer Program ($99/yr) and Google Play Console ($25 one-time). PWA "Add to Home Screen" already gives an app-like experience without any of this.

### Also discussed: in-app screenshot OCR
Considered replacing the "share screenshot with Claude" step with in-browser OCR (Tesseract.js, no backend needed). Not built — flagged that accuracy would be materially worse than Claude's vision-based reading, since it would require heuristic parsing (regex, layout assumptions) of Apple Wallet's screenshot format. Left as a possible future experiment if the chat-based flow proves too much friction.

---

## 7. Known user preferences (in case future Claude sessions should know)

- Prefers direct, incremental changes with immediate `git push` after each edit — this project has been built through many small back-to-back requests rather than big upfront specs
- Cares about visual polish — "modern, pure, clean" with copper + purple + white accents was an explicit late-stage design direction
- Wants duplicate protection on all import paths (manual CSV wizard and auto-import both use the same `txnDupeKey` date+amount+description matching)
- Explicitly does not want savings/investment "envelope" money to affect the Net Balance calculation — that's a firm rule, not a suggestion
- Is cautious about security/risk tradeoffs — explicitly steered away from RBC login automation once the account-lockout risk was explained
