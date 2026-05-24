# Minorph — Personal Finance Tracker

## What is this?
Minorph is a personal finance tracking web app for Rudi Langford. It ingests monthly PDF bank statements, parses transactions, detects subscriptions, tracks goals (including an Amex spending target), and displays everything in a clean per-account dashboard.

Cross-device sync via Supabase. Hosted on Cloudflare Pages. No open banking APIs — everything is manual monthly PDF uploads.

---

## Tech Stack
- **Frontend:** Single HTML file (`index.html`), vanilla JS, no framework
- **Backend:** Supabase (Postgres + Auth)
- **Hosting:** Cloudflare Pages (connected to GitHub repo)
- **PDF Parsing:** PDF.js 3.11 (client-side CDN — nothing uploaded to a server)
- **Charts:** Chart.js 4.x (CDN)
- **Fonts:** Syne + DM Sans (Google Fonts)

---

## Accounts Supported
| Account | Bank | Acc Number | Notes |
|---|---|---|---|
| Main Current | Lloyds | 39741868 | Primary spending — grocery reimbursements land here |
| Rent & Bills | Lloyds | 48939060 | Holding account — rent, energy, grocery float |
| Platinum Credit Card | Lloyds | — | Credit card — statement closes varies |
| GBP Current | Revolut | 82501043 | Travel spending |
| GBP Savings | Revolut | 82501043 | Instant access savings via ClearBank |
| Gold Rewards Card | Amex | — | Statement closes 28th of month |

---

## Screens
1. **Dashboard** — combined total balance, per-account cards, commission income bar + month-on-month Chart.js bar chart
2. **Transactions** — full list, filterable by account and category (All / Subscriptions / CS2 / Bills / Income / Transfers)
3. **Subscriptions** — auto-detected recurring charges, approval flow, monthly total
4. **Goals** — Amex £5k spend tracker (progress bar, months remaining, needed/month), Revolut savings goal
5. **Upload** — PDF upload per account, parse progress, upload history

---

## Design Tokens
| Token | Value |
|---|---|
| Background | `#080810` |
| Surface | `#0f0f1a` |
| Purple | `#9b5de5` / light `#c77dff` |
| Green | `#2dc653` / light `#57f288` |
| Red (negative) | `#ff6b6b` |
| Muted text | `#7777aa` |
| Card gradient | `linear-gradient(135deg, #1a0f2e, #0d1a14)` |
| Progress bar | `linear-gradient(90deg, #9b5de5, #57f288)` |
| Heading font | Syne 700/800 |
| Body font | DM Sans 400/500 |
| Logo | Abstract M in gradient circle ring |
| Nav | Fixed bottom tab bar, 5 tabs |

---

## PDF Parsing — Per Bank Format

### Lloyds (Current + Credit)
Columns: `Date | Description | Type | Money In (£) | Money Out (£) | Balance (£)`
- Date format: `07 Apr 26` (two-digit year)
- Transaction types: `DD` (Direct Debit), `DEB` (Debit Card), `FPI` (Faster Payment In), `FPO` (Faster Payment Out), `TFR` (Transfer), `SO` (Standing Order), `CPT` (Cashpoint), `COR` (Correction)
- Parse strategy: regex row extraction with line-by-line fallback for awkward PDF renderings
- TFRs referencing own account numbers or `R LANGFORD` → tagged `transfer` (internal, excluded from spend)

### Revolut Current
Columns: `Date | Description | Money out | Money in | Balance`
- Date format: `15 Apr 2026` (four-digit year)
- Foreign transactions have sub-rows (Fee, Rate, EUR amount) — skipped
- Two sections parsed: "Pending" (tagged) and "Account transactions"

### Revolut Savings
Columns: `Date | Description | Money out | Money in | Balance`
- Mostly `Gross Interest` daily entries + Deposit/Withdrawal
- Interest tagged as `INT` type → `interest` category (excluded from spend)

### Amex Gold
- Statement closes 28th of each month
- Date format: `01 Apr 26` (same two-digit year as Lloyds — reuses `parseLloydsDate`)
- Sections: "New Charges" (purchases), "Payments and Credits" (credits/payments), "Fees and Adjustments"
- Foreign currency sub-rows (USD, EUR etc) skipped
- No per-row balance column — `runningBalance` stored as `null`
- Synthesised type codes: `PUR` (purchase), `PAY` (payment/credit), `FEE` (fee)
- **Parser is speculative** — written against standard Amex UK format. Test against your first May 2026 statement and check console logs for any zero-result warnings.

---

## Categorisation Rules

Categories: `commission` · `groceries` · `transfer` · `interest` · `subscription` · `cs2` · `bills` · `income` · `spending` · `other`

### Special logic (evaluated in priority order before pattern rules)

**Grocery reimbursement** — two-sided:
- *Side A:* Outgoing TFR/FPO from Rent & Bills (48939060), ≤ £300, description references Main Current (39741868) or `R LANGFORD` → `groceries`
- *Side B:* Incoming TFR/FPI on Main Current (39741868), ≤ £300, description references `48939060` → `groceries`
- The £300 cap (`GROCERY_BUDGET_CAP`) prevents large own-account transfers being misclassified. Raise it if needed.
- Grocery purchases on the card (TESCO, LIDL etc) are tagged `groceries` separately via pattern rules.

**Internal transfers** — TFR/FPO/FPI where description references any own account number or `R LANGFORD` → `transfer`, `is_internal_transfer = true`, excluded from spend calculations.

**Commission** — any FPI from `INFINITY RENEWABLE` → `commission`, `is_commission = true`, shown separately on dashboard.

### Pattern rules (in order, first match wins)
| Pattern | Category |
|---|---|
| NETFLIX | subscription |
| EE LIMITED / EE MOBILE / EE DD | subscription |
| VIRGIN MEDIA | subscription |
| CLAUDE.AI / ANTHROPIC | subscription |
| HASTINGS | subscription |
| ASHBOURNE | subscription |
| DVLA | subscription |
| GOOGLE PLAY | subscription |
| SPOTIFY | subscription |
| AMAZON PRIME / AMZNPRIME | subscription |
| APPLE.COM/BILL | subscription |
| DISNEY PLUS / NOW TV | subscription |
| ADOBE / MICROSOFT 365 | subscription |
| CSFLOAT / CS FLOAT | cs2 |
| SKINLEDGER / SKIN LEDGER | cs2 |
| STEAMGAMES / STEAM PURCHASE | cs2 |
| LEADERS | bills |
| OVO ENERGY | bills |
| COUNCIL TAX | bills |
| Named water suppliers | bills |
| Major supermarkets (LIDL, TESCO etc) | groceries |

### Type-based fallbacks (if no pattern matches)
- `FPI` / `BGC` / `DEP` → `income`
- `DD` / `SO` → `bills`
- `DEB` → `spending`
- Everything else → `other`

---

## Subscription Detection
- After each statement upload, all transactions for that account are scanned
- Descriptions appearing in 2+ distinct calendar months with similar amounts → flagged as subscription candidates
- Inserted into `subscriptions` table with `approved = false`
- User reviews and approves/dismisses via the Subscriptions screen
- Existing subscriptions are not duplicated on re-upload

---

## Commission Income — Dashboard
- Any `is_commission = true` transaction displayed in income bar on Dashboard
- Chart.js bar chart below shows month-on-month commission grouped by `YYYY-MM`
- Chart renders with 1+ months of data and grows as more statements are uploaded
- Total and monthly average shown below chart

---

## Grocery Budget Logic (£300/mo)
1. Groceries purchased on any card (Amex, Lloyds credit, Main Current debit)
2. Exact amount transferred from Rent & Bills → Main Current to reimburse
3. Both sides tagged `groceries`, `is_internal_transfer = false`
4. End of month: credit cards paid off from Main Current float
5. Amex/Lloyds credit card statements will show the grocery purchases — tagged via supermarket patterns

---

## Supabase Schema

Run this in the Supabase SQL editor on a **new project** (not the Infinity Renewables one):

```sql
-- ── Accounts ──────────────────────────────────────────────────────────────
create table accounts (
  id             uuid primary key default gen_random_uuid(),
  user_id        uuid references auth.users not null,
  name           text not null,
  bank           text not null,
  account_number text,
  type           text -- 'current' | 'savings' | 'credit' | 'travel'
);

alter table accounts enable row level security;
create policy "Users see own accounts" on accounts
  for all using (auth.uid() = user_id);

-- ── Statements ────────────────────────────────────────────────────────────
create table statements (
  id              uuid primary key default gen_random_uuid(),
  account_id      uuid references accounts not null,
  period_start    date,
  period_end      date,
  opening_balance numeric,
  closing_balance numeric,
  uploaded_at     timestamptz default now()
);

alter table statements enable row level security;
create policy "Users see own statements" on statements
  for all using (
    account_id in (select id from accounts where user_id = auth.uid())
  );

-- ── Transactions ──────────────────────────────────────────────────────────
create table transactions (
  id                   uuid primary key default gen_random_uuid(),
  statement_id         uuid references statements,
  account_id           uuid references accounts not null,
  date                 date,
  description          text,
  type                 text,
  amount_in            numeric default 0,
  amount_out           numeric default 0,
  balance              numeric,
  category             text,
  -- category values: 'commission' | 'groceries' | 'transfer' | 'interest'
  --                  'subscription' | 'cs2' | 'bills' | 'income'
  --                  'spending' | 'other'
  is_internal_transfer boolean default false,
  is_commission        boolean default false
);

alter table transactions enable row level security;
create policy "Users see own transactions" on transactions
  for all using (
    account_id in (select id from accounts where user_id = auth.uid())
  );

-- ── Subscriptions ─────────────────────────────────────────────────────────
create table subscriptions (
  id        uuid primary key default gen_random_uuid(),
  user_id   uuid references auth.users not null,
  name      text not null,
  amount    numeric,
  cycle     text default 'monthly',
  icon      text,
  approved  boolean default false,
  last_seen date
);

alter table subscriptions enable row level security;
create policy "Users see own subscriptions" on subscriptions
  for all using (auth.uid() = user_id);

-- ── Goals ─────────────────────────────────────────────────────────────────
create table goals (
  id             uuid primary key default gen_random_uuid(),
  user_id        uuid references auth.users not null,
  name           text not null,
  target         numeric not null,
  current_amount numeric default 0,
  deadline       date,
  type           text, -- 'spend' | 'save'
  created_at     timestamptz default now()
);

alter table goals enable row level security;
create policy "Users see own goals" on goals
  for all using (auth.uid() = user_id);

-- ── Seed accounts ─────────────────────────────────────────────────────────
-- Run this AFTER creating your user account in the app, then replace
-- the UUID below with your actual user ID from Supabase Auth → Users.
-- Your user ID: (find in Supabase dashboard → Authentication → Users)

insert into accounts (user_id, name, bank, account_number, type) values
  ('YOUR-USER-UUID', 'Main Current',      'Lloyds',  '39741868', 'current'),
  ('YOUR-USER-UUID', 'Rent & Bills',      'Lloyds',  '48939060', 'current'),
  ('YOUR-USER-UUID', 'Platinum Credit',   'Lloyds',  null,       'credit'),
  ('YOUR-USER-UUID', 'Revolut Current',   'Revolut', '82501043', 'travel'),
  ('YOUR-USER-UUID', 'Revolut Savings',   'Revolut', '82501043', 'savings'),
  ('YOUR-USER-UUID', 'Amex Gold',         'Amex',    null,       'credit');

-- ── Seed goals ────────────────────────────────────────────────────────────
-- Replace YOUR-USER-UUID with your actual user ID.
-- Adjust current_amount once you have Amex statement data.

insert into goals (user_id, name, target, current_amount, deadline, type) values
  ('YOUR-USER-UUID', 'Amex Gold — 40k Points', 5000, 0, '2026-10-28', 'spend'),
  ('YOUR-USER-UUID', 'Revolut Savings',        5000, 0, null,         'save');
```

---

## Setup Steps — Desktop

### 1. GitHub
```
git init
git add index.html README.md .gitignore
git commit -m "init"
gh repo create Rudizzle123/minorph --public --push
```

### 2. Supabase
1. Create new project at supabase.com (name: `minorph` — separate from Infinity Renewables)
2. SQL Editor → run the full schema SQL above
3. Authentication → Providers → enable Email
4. Settings → API → copy **Project URL** and **anon public** key
5. Paste both into `index.html` at the top:
   ```js
   const SUPABASE_URL  = 'https://xxxx.supabase.co';
   const SUPABASE_ANON = 'eyJ...';
   ```
6. Sign up in the app → get your User ID from Supabase Auth → Users
7. Replace `YOUR-USER-UUID` in the seed SQL and run it

### 3. Cloudflare Pages
1. Cloudflare dashboard → Pages → Create a project → Connect to Git
2. Select the `minorph` repo
3. Build settings: Framework preset = **None**, build command = **blank**, output directory = `/`
4. Deploy — live at `minorph.pages.dev`
5. Every `git push` auto-deploys

---

## File Structure
```
minorph/
├── index.html   ← entire app (HTML + CSS + JS, ~3,200 lines)
├── README.md
└── .gitignore
```

`.gitignore`:
```
.DS_Store
*.log
```

---

## Key Decisions
- PDF upload only — no open banking APIs
- Everything parsed client-side — PDFs never leave the device
- Each account shown independently — no cross-account spend aggregation
- Combined total balance at top uses closing balance from latest statement per account
- Subscriptions require manual approval before tracking
- Internal transfers excluded from spend by `is_internal_transfer` flag
- Grocery reimbursements are NOT internal transfers — they represent real spend
- Amex parser written speculatively — verify against first statement (closes 28 May 2026)
- Commission chart grows automatically as more months of data are uploaded
- Bare "STEAM" not pattern-matched as CS2 — too ambiguous (train companies)

---

## Things to do once first Amex statement arrives (28 May 2026)
1. Upload via the Upload screen (select "Amex Gold" account)
2. Check console logs — if `[parseAmex] parsed 0 transactions`, the raw text will be dumped to console
3. Common things to adjust in `parseAmex()`:
   - Section header patterns (`SECTION_CHARGES`, `SECTION_PAYMENTS`) if Amex uses different wording
   - The `SKIP_RE` foreign currency list if your statement uses other currencies
   - Amount sign convention if credits appear differently
4. Update the Amex Goal `current_amount` in Supabase once spend data is confirmed
