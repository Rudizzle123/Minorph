# Minorph — Personal Finance Tracker

## What is this?
Minorph is a **personal CFO / monthly wealth-review tool** for Rudi Langford. It ingests monthly PDF bank statements, parses transactions client-side, and surfaces:

- **Net worth** (investments + savings, not current accounts)
- **Net pay** rolling 6-month average (to smooth bonus-month variance)
- **Savings rate** and **invested rate** per month
- **Discretionary spend** vs 6-month baseline
- **Pension / ANI tracker** — tax-year SIPP contribs grossed up + ANI vs £100k threshold
- **Where it went** — monthly spend breakdown by category + top merchants
- **Wealth snapshots** (manual monthly input of SIPP / ISA / CS2 valuations + contributions)
- **CS2 investment log** (manual purchase entries)
- **Goals tracking** (Amex Gold 40k point bonus, savings)
- **Subscriptions & Bills**

Cross-device sync via Supabase. Hosted on Cloudflare Pages. No open banking APIs — everything is manual monthly PDF uploads.

**The app's job is not budgeting.** It's once-a-month wealth review and tax/optimisation visibility (long-term focus: keep ANI under £100k via aggressive SIPP contributions).

---

## Infrastructure
| Thing | Detail |
|---|---|
| Frontend | Single `index.html`, vanilla JS, no framework |
| Backend | Supabase — project `minorph`, org `Rudizzle123` |
| Supabase URL | `https://soplpclugrtwahtvlkdi.supabase.co` |
| GitHub repo | `github.com/Rudizzle123/Minorph` |
| Hosting | Cloudflare Pages → `minorph.pages.dev` |
| PDF parsing | PDF.js 3.11 CDN, client-side only |
| Charts | Chart.js 4.x CDN |
| Fonts | Syne + DM Sans (Google Fonts) |

---

## Accounts
| Account | Bank | Acc Number | Type | Counts toward Net Worth? |
|---|---|---|---|---|
| Main Current | Lloyds | 39741868 | current | No |
| Rent & Bills | Lloyds | 48939060 | current | No |
| Platinum Credit | Lloyds | — | credit | No |
| Revolut Current | Revolut | 82501043 | travel | No |
| Revolut Savings | Revolut | 82501043 | savings | **Yes** |
| Amex Gold | Amex | — | credit | No |
| Fidelity SIPP | Fidelity | — | wealth (manual) | **Yes** |
| Fidelity S&S ISA | Fidelity | — | wealth (manual) | **Yes** |
| CS2 Portfolio | Steam | — | wealth (manual) | **Yes** |

Wealth accounts (SIPP / ISA / CS2) are not in the `accounts` table — they live as monthly rows in `wealth_snapshots`.

---

## Supabase Schema

Run in SQL Editor on a fresh project:

```sql
create table accounts (
  id             uuid primary key default gen_random_uuid(),
  user_id        uuid references auth.users not null,
  name           text not null,
  bank           text not null,
  account_number text,
  type           text
);
alter table accounts enable row level security;
create policy "Users see own accounts" on accounts
  for all using (auth.uid() = user_id);

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
  is_internal_transfer boolean default false,
  is_commission        boolean default false,
  category_override    boolean default false
);
alter table transactions enable row level security;
create policy "Users see own transactions" on transactions
  for all using (
    account_id in (select id from accounts where user_id = auth.uid())
  );

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

create table goals (
  id             uuid primary key default gen_random_uuid(),
  user_id        uuid references auth.users not null,
  name           text not null,
  target         numeric not null,
  current_amount numeric default 0,
  deadline       date,
  type           text,
  created_at     timestamptz default now()
);
alter table goals enable row level security;
create policy "Users see own goals" on goals
  for all using (auth.uid() = user_id);

create table bills (
  id       uuid primary key default gen_random_uuid(),
  user_id  uuid references auth.users not null,
  name     text not null,
  amount   numeric not null,
  cycle    text default 'monthly',
  due_day  int,
  icon     text
);
alter table bills enable row level security;
create policy "Users see own bills" on bills
  for all using (auth.uid() = user_id);

-- Added Session 9: monthly snapshots of investment / savings account values
create table wealth_snapshots (
  id                uuid primary key default gen_random_uuid(),
  user_id           uuid references auth.users not null,
  month             date not null,
  sipp_balance      numeric default 0,
  sipp_contrib_cash numeric default 0,
  isa_balance       numeric default 0,
  isa_contrib_cash  numeric default 0,
  cs2_balance       numeric default 0,
  unique (user_id, month)
);
alter table wealth_snapshots enable row level security;
create policy "Users see own snapshots" on wealth_snapshots
  for all using (auth.uid() = user_id);

-- Added Session 9: log of manual investment purchases (currently just CS2)
create table manual_investments (
  id      uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users not null,
  type    text not null,
  date    date not null,
  amount  numeric not null,
  notes   text
);
alter table manual_investments enable row level security;
create policy "Users see own investments" on manual_investments
  for all using (auth.uid() = user_id);

-- Added Session 10: per-user settings — currently just annual gross income for ANI estimate
create table user_settings (
  user_id              uuid primary key references auth.users,
  annual_gross_income  numeric default 0,
  tax_year_start_month int default 4,
  tax_year_start_day   int default 6,
  updated_at           timestamptz default now()
);
alter table user_settings enable row level security;
create policy "Users see own settings" on user_settings
  for all using (auth.uid() = user_id);
```

---

## Pulse Dashboard

The dashboard is built around six sections, top-to-bottom:

### 1. Net Worth hero
Big gradient number (white → lilac). Below: date chip + breakdown chips (`SIPP £X`, `ISA £X`, `CS2 £X`, `Savings £X`). Sparkline canvas plots net worth over all snapshot months.

**Computation:**
- SIPP / ISA / CS2 → latest `wealth_snapshots` row
- Revolut Savings → latest `statements.closing_balance` for the Revolut Savings account
- Net worth = sum of all four

### 2. Pulse grid (4 tiles)
- **Net Pay (Month)** = Infinity Renewables FPI total for the anchor month. Sub = 6-mo rolling avg.
- **Savings Rate** = (Revolut Savings deposits this month) / (net pay this month). Sub = total saved £ + 6-mo avg rate.
- **Invested (Month)** = (Fidelity outflows from Lloyds Main Current detected by description: `FSTL PRIMARY TRUST`, `FASL PRIM CLIENT B`, `FIDELITY`, `FIL SIPP`, `FIL ISA`, etc) + (manual CS2 purchases for the month). Sub = 6-mo avg.
- **Discretionary** = total `amount_out` for the month, minus internal transfers, commission, bills, rent, energy, subscriptions, interest, groceries. Sub = 6-mo avg + % delta (red if up, green if down).

### 3. Pension & ANI tracker (added Session 10)
Card between the Pulse grid and Wealth Snapshot section. ⚙ Settings cog opens modal to set `annual_gross_income`.

**Computation:**
- **Tax year** = UK tax year (6 Apr → 5 Apr). Determined client-side from current date.
- **SIPP cash YTD** = sum of `amount_out` on transactions where `description` matches `/FSTL\s*PRIMARY\s*TRUST/i` AND date falls within the current tax year.
- **Grossed (×1.25)** = cash YTD × 1.25 (basic-rate tax relief that Fidelity claims back from HMRC).
- **Estimated ANI** = `annual_gross_income − grossed YTD`.
- Threshold = £100,000 (personal allowance taper begins). Bar runs 0 → £150k so threshold sits at ~67%.
- ANI tint: green (>£5k below threshold) / amber (within £5k) / red (over).

Hidden behind a setup prompt if `annual_gross_income` is 0.

### 4. Wealth Snapshot section
Card showing latest snapshot row. **＋ Add / Edit** button opens modal; selecting a month auto-prefills if a snapshot exists for that month (becomes an edit). Upsert key: `(user_id, month)`.

### 5. CS2 Investments section
List of purchases (latest 6 visible). **＋ Log purchase** button adds a new row. × button on each row deletes (with confirm).

### 6. Net Pay month-on-month chart
Flame-palette bar chart (orange `#ff8a3d` / light `#ffb267`). Fixed 140px canvas height. Total + monthly avg legend below.

### Dashboard month anchoring
Priority chain for the "current month" Pulse uses:
1. Latest month with any Infinity FPI transaction
2. Latest statement period_end month
3. Wall-clock current month

This prevents the dashboard showing all-zero numbers in the gap between months when no new statement has been uploaded.

---

## Transactions Screen

The Transactions tab has a segmented toggle at the top:

### All transactions
Linear list of transactions (current behaviour). Account filter + category chips (All / Subscriptions / CS2 / Bills / Income / Transfers). Pulls up to 200 transactions ordered by date.

### Where it went (added Session 10)
Monthly spend breakdown. Month picker populated from months that have any Infinity FPI deposit (i.e. "paid months"), defaults to latest.

**Top section — two summary tiles:**
- **Total Spend** — sum of qualifying outflows. Sub = 6-mo avg + % delta.
- **Subscriptions** — sum of `subscription` category. Sub = % of total spend.

**By category** — categories ranked by £ this month. Each row:
- Icon (via `getCategoryIcon`)
- Category name + horizontal bar (% of leader)
- £ total + percentage of monthly spend + % delta vs 6-mo avg

**Top merchants** — top 10 descriptions for the month, after `cleanMerchantName()` normalises out card refs, date fragments, type tokens, and trailing punctuation.

**What counts as "spend":**
- Excluded categories: `transfer`, `commission`, `interest`, `income`
- Excluded by description (regardless of category): `FIDELITY | FIL\s*INV | FIL\s*LIFE | FIL\s*SIPP | FIL\s*ISA | FSTL PRIMARY TRUST | FASL PRIM CLIENT | REVOLUT | PLATINUM CREDIT | LLOYDS CREDIT CARD`
- These are wealth movements or already-counted spend (credit card payoffs), not consumption

**Data caching:** the first time the user opens the panel, all transactions are pulled into `spendCache` and reused across month picker changes.

---

## PDF Parsing

### Parser routing (in `startParse`)
```
Lloyds + account.type === 'credit'       → parseLloydsCredit
Lloyds (all others)                      → parseLloyds
Revolut + account.name includes 'saving' → parseRevolutSavings
Revolut (all others)                     → parseRevolutCurrent
Amex / American Express                  → parseAmex
```

### Lloyds Current (parseLloyds) — ✅ verified
PDF.js gives back text with mixed-up column labels and values. The parser:
1. **Strips label noise** — removes `Date`, `Description`, `Type`, `Money In (£)`, `Money Out (£)`, `Balance (£)`, `blank.`, `(£)`
2. **Tokenises** the cleaned text on whitespace
3. **Finds every date triplet** — `day(1–2 digits) Mon(3+ letters) yy(2 digits)` — these mark row boundaries
4. **For each row** (tokens between two date triplets):
   - Finds the first token that matches the TYPE set (`DD, DEB, FPI, FPO, TFR, SO, CPT, COR, BGC, CHQ, ATM`)
   - Description = tokens before TYPE, stripped of leading/trailing punctuation
   - Amounts = `\d+\.\d{2}` tokens after TYPE — last is balance, first is the in/out amount
   - In/out direction is decided by TYPE (FPI/BGC/COR = in; rest = out)
5. **TFR direction flip** — if description references own-account marker, swap in↔out
- Date format: `07 Apr 26`

### Lloyds Credit (parseLloydsCredit) — ✅ verified
Lloyds Platinum Mastercard PDF layout differs from current accounts:
- Rows begin with a **full month name** (OCTOBER, NOVEMBER…)
- Format: `MONTHNAME DESCRIPTION AMOUNT [CR] CARD_REF [trailing_junk]`
  - `CR` suffix = payment received (amountIn); no suffix = purchase (amountOut)
  - Trailing junk tokens after card ref are stripped by finding the last money-shaped token

**Strategy:**
1. Tokenise full text; extract statement year from first `20XX` token
2. Find all full month-name token positions — these delimit rows
3. For each row: scan last 4 tokens for `CR`; strip everything after last `\d+\.\d{2}` token
4. `CR` → type `PAY`, amountIn; no CR → type `PUR`, amountOut

**Notes:** `month positions found:` will be ~110 per statement — normal, over-matches produce empty slices that skip cleanly.

### Revolut Current (parseRevolutCurrent) — ✅ verified
Columns: Date | Description | Money out | Money in | Balance
- Date format: `15 Apr 2026`
- Foreign-currency sub-rows (Fee/Rate/EUR) skipped via `SKIP_RE`
- Pending section detected and flagged
- 3 amounts → out/in/balance; 2 amounts → heuristic by description; 1 amount → out

### Revolut Savings (parseRevolutSavings) — ✅ verified
Same columns as Revolut Current. Mostly Gross Interest (daily) + Deposit/Withdrawal entries.
- Interest tagged `INT` type → categorised as `interest`
- Deposit tagged `DEP`, withdrawal tagged `WDL`

### Amex Gold (parseAmex) — ⚠️ speculative, untested
- Statement closes 28th of month — first due 28 May 2026
- Sections: "New Charges" / "Payments" / "Fees and Adjustments"
- Date format: `01 Apr 26` (same as Lloyds, reuses `parseLloydsDate`)
- If `[parseAmex] parsed 0 transactions`, raw text is dumped to console for debugging

### Fidelity (not built yet)
Quarterly statements covering ISA + SIPP + Cash Management in one PDF. Known structure:
- Per-account valuation: balance + holdings (fund name, quantity, price, value)
- Per-account statement: dated rows of credits (`Money paid in`, dividends, interest distributions) and debits (`Investment in X`, fees)
- Date format: `DD.MM.YY` (e.g. `2.1.26`)
- Account masks: ISA `******0990`, SIPP `******8515`, Cash `******9460`

Build when Rudi has at least 2-3 historical statements to validate against.

### Duplicate statement guard
Before inserting a new statement, `startParse` queries `statements` for a row matching `account_id` + `period_end`. If found, aborts with a toast and returns early — no DB writes occur.

---

## Categorisation

Priority order (first match wins):

1. Grocery reimbursement — TFR/FPO out of Rent & Bills ≤ £300 referencing Main Current → `groceries`
2. Grocery reimbursement — TFR/FPI into Main Current ≤ £300 referencing Rent & Bills → `groceries`
3. Internal transfer — catches Revolut `To/From GBP Savings` by description first, then TFR/FPO/FPI referencing own account numbers or R LANGFORD → `transfer`, excluded from spend
4. Commission — FPI from `INFINITY RENEWABLE` → `commission`, `is_commission = true` (kept as a flag, but displayed as "Net Pay" throughout the Pulse dashboard)
5. Revolut interest — type `INT` → `interest`
6. Pattern rules (subscriptions, CS2, bills, groceries by merchant name)
7. Type fallbacks — FPI/BGC/DEP/PAY → income, DD/SO → bills, DEB → spending

### Manual override
User can tap any transaction to change its category. Saves `category` + sets `category_override = true`. Override transactions show a purple "edited" pill. Override is permanent until changed again.

### Fidelity contribution detection (used in Invested tile)
Pattern matched against description, uppercased:
```
FIDELITY | FIL\s*INV | FIL\s*LIFE | FIL\s*SIPP | FIL\s*ISA | FSTL\s*PRIMARY\s*TRUST | FASL\s*PRIM\s*CLIENT
```
Verified on real Lloyds Main Current statements:
- `FSTL PRIMARY TRUST` (FPO type) → Fidelity SIPP
- `FASL PRIM CLIENT B` (FPO type) → Fidelity S&S ISA

### Pension contribution detection (used in ANI tile)
Stricter than the Invested detection — only `FSTL PRIMARY TRUST` (the SIPP-specific code) counts toward the tax-year grossed-up figure. ISA contribs don't reduce ANI.
- Date filter: within current UK tax year (6 Apr → 5 Apr).

### Key constants in index.html
```js
const RENT_AND_BILLS_ACC_NUM = '48939060';
const MAIN_CURRENT_ACC_NUM   = '39741868';
const GROCERY_BUDGET_CAP     = 300;
const OWN_ACCOUNT_MARKERS    = ['R LANGFORD','RUDI LANGFORD','LANGFORD R','LANGFORD RUDI','48939060','39741868','82501043'];
```

---

## Transaction icons (`getCategoryIcon`)
Category takes priority, then description patterns. Major coverage:
- **Food delivery**: Deliveroo, Uber Eats, Just Eat → 🛵
- **Supermarkets**: Aldi, Lidl, Tesco, Sainsbury's, Asda, Morrisons, Waitrose, Co-op, M&S Food, Ocado, Iceland → 🛒
- **Dining**: McDonald's, KFC, Nando's, Wagamama, Pizza, restaurant, cafe, coffee, Greggs, Pret, farm shops → 🍽️
- **Fuel**: BP, Shell, Esso, service station → ⛽
- **Shopping**: Amazon → 📦, Argos/Currys/John Lewis/Next/Primark/H&M/ASOS → 🛍️, Smyths/toys → 🧸
- **Transport**: Uber/Bolt/taxi → 🚕, Trainline/TfL/rail → 🚂, parking → 🅿️
- **Travel**: flights, hotels, Airbnb → ✈️
- **Health**: hospital, pharmacy, dentist, NHS → 🏥
- **Gaming**: Steam, PlayStation, Xbox → 🎮
- **Utilities**: energy suppliers → 💡, water → 💧, broadband → 🌐
- **Insurance** → 🛡️, **rent/council tax** → 🏠, **phone/mobile** → 📱
- **Fallback**: 💳

---

## Known issues / watch points

- **Amex parser** — speculative, untested until 28 May 2026 statement
- **Fidelity parser** — not built yet; manual snapshot entry is the stopgap
- **Lloyds TYPE collisions** — TYPE set is intentionally narrowed. If a future statement uses a code outside the known set, that row will be skipped
- **Grocery description matching** — Side A requires description to contain `39741868` or `R LANGFORD`; verify on real Rent & Bills data
- **Revolut Savings balance** — assumes last number on each line is the balance
- **Sparkline early-state** — with only 1 snapshot, sparkline renders a flat dot; will populate as more snapshots accumulate
- **"Other" / "Spending" buckets in Where It Went** — too many uncategorised merchants land here (e.g. GI GI FIRECRACKER, LC INTERNATIONAL, DAMIRA QUAYSIDE). Needs more merchant-specific rules in `CATEGORY_RULES`
- **Hastings Insurance icon** — currently tagged `subscription` but displays with a generic icon in Top Merchants; icon lookup precedence may need tweaking
- **ANI tracker assumes SIPP-only relief** — doesn't account for Gift Aid donations or other ANI-reducing items. Add if relevant later

---

## Things still to do (priority order)

1. **Categorisation cleanup** — add merchant rules + icons for repeat merchants surfaced by Where It Went; reconsider naming of `spending` bucket
2. **Wealth long-game chart** — net worth line by account type
3. **Fidelity PDF parser** (when historical statements available)
4. **Amex Gold parser** test (28 May 2026)
5. **Search/filter transactions by description**
6. **Export transactions as CSV**
