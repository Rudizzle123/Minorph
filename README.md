# Minorph — Personal Finance Tracker

## What is this?
Minorph is a personal finance tracking web app for Rudi Langford. It ingests monthly PDF bank statements, parses transactions client-side, detects subscriptions, tracks goals (Amex points target + savings), and displays everything in a clean per-account dashboard.

Cross-device sync via Supabase. Hosted on Cloudflare Pages. No open banking APIs — everything is manual monthly PDF uploads.

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
| Account | Bank | Acc Number | Type |
|---|---|---|---|
| Main Current | Lloyds | 39741868 | current |
| Rent & Bills | Lloyds | 48939060 | current |
| Platinum Credit | Lloyds | — | credit |
| Revolut Current | Revolut | 82501043 | travel |
| Revolut Savings | Revolut | 82501043 | savings |
| Amex Gold | Amex | — | credit |

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
```

### Seed data (replace UUID with your user ID from Auth → Users)
```sql
insert into accounts (user_id, name, bank, account_number, type) values
  ('YOUR-USER-UUID', 'Main Current',    'Lloyds',  '39741868', 'current'),
  ('YOUR-USER-UUID', 'Rent & Bills',    'Lloyds',  '48939060', 'current'),
  ('YOUR-USER-UUID', 'Platinum Credit', 'Lloyds',  null,       'credit'),
  ('YOUR-USER-UUID', 'Revolut Current', 'Revolut', '82501043', 'travel'),
  ('YOUR-USER-UUID', 'Revolut Savings', 'Revolut', '82501043', 'savings'),
  ('YOUR-USER-UUID', 'Amex Gold',       'Amex',    null,       'credit');

insert into goals (user_id, name, target, current_amount, deadline, type) values
  ('YOUR-USER-UUID', 'Amex Gold — 40k Points', 5000, 0, '2026-10-28', 'spend'),
  ('YOUR-USER-UUID', 'Revolut Savings',        5000, 0, null,         'save');
```

---

## Deployment

### Making changes
1. Edit `index.html` locally (replace with file from Claude)
2. GitHub Desktop → commit → push to `main`
3. Cloudflare auto-deploys to `minorph.pages.dev`

### If Cloudflare Git connection breaks
Settings → Git repository → reconnect GitHub, then re-deploy.

### To clear bad parse data from Supabase
```sql
-- Delete most recent statement for a specific account (safe, doesn't touch others):
DELETE FROM transactions
WHERE statement_id = (
  SELECT id FROM statements
  WHERE account_id = (SELECT id FROM accounts WHERE name ILIKE '%account name%')
  ORDER BY uploaded_at DESC LIMIT 1
);
DELETE FROM statements
WHERE id = (
  SELECT id FROM statements
  WHERE account_id = (SELECT id FROM accounts WHERE name ILIKE '%account name%')
  ORDER BY uploaded_at DESC LIMIT 1
);

-- Nuke everything:
DELETE FROM transactions;
DELETE FROM statements;
```

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

### Duplicate statement guard
Before inserting a new statement, `startParse` queries `statements` for a row matching `account_id` + `period_end`. If found, aborts with a toast and returns early — no DB writes occur.

---

## Categorisation

Priority order (first match wins):

1. Grocery reimbursement — TFR/FPO out of Rent & Bills ≤ £300 referencing Main Current → `groceries`
2. Grocery reimbursement — TFR/FPI into Main Current ≤ £300 referencing Rent & Bills → `groceries`
3. Internal transfer — catches Revolut `To/From GBP Savings` by description first, then TFR/FPO/FPI referencing own account numbers or R LANGFORD → `transfer`, excluded from spend
4. Commission — FPI from `INFINITY RENEWABLE` → `commission`, `is_commission = true`
5. Revolut interest — type `INT` → `interest`
6. Pattern rules (subscriptions, CS2, bills, groceries by merchant name)
7. Type fallbacks — FPI/BGC/DEP/PAY → income, DD/SO → bills, DEB → spending

### Manual override
User can tap any transaction to change its category. Saves `category` + sets `category_override = true`. Override transactions show a purple "edited" pill. Override is permanent until changed again.

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
- **Fallback**: 💳 (replaces old white dot)

---

## Known issues / watch points

- **Amex parser** — speculative, untested until 28 May 2026 statement
- **Lloyds TYPE collisions** — TYPE set is intentionally narrowed. If a future statement uses a code outside the known set, that row will be skipped
- **Grocery description matching** — Side A requires description to contain `39741868` or `R LANGFORD`; verify on real Rent & Bills data
- **Revolut Savings balance** — assumes last number on each line is the balance

---

## Things still to do

- [ ] Test Amex Gold parser on 28 May 2026 statement
- [ ] Update Amex goal `current_amount` once Amex parser verified
- [ ] Grocery spend tracker — £X of £300 budget used this month
- [ ] Search/filter transactions by description
- [ ] Export transactions as CSV
