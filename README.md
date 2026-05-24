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
  is_commission        boolean default false
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
1. Edit `index.html` locally
2. GitHub Desktop → commit → push to `main`
3. Cloudflare auto-deploys to `minorph.pages.dev`

### If Cloudflare Git connection breaks
Settings → Git repository → reconnect GitHub, then re-deploy.

### To clear bad parse data from Supabase
```sql
delete from transactions where account_id = (select id from accounts where account_number = '39741868');
delete from statements  where account_id = (select id from accounts where account_number = '39741868');
```
Change the account number to target a different account.

---

## PDF Parsing

### Lloyds (Current + Credit)
PDF.js extracts items joined with spaces per line. The parser:
1. Strips column label noise (`Date`, `Description`, `Type`, `Money In (£)`, `blank.` etc.)
2. Flattens to one string
3. Regex-matches: `DATE DESCRIPTION TYPE [amounts]`
4. Extracts amounts from the portion after the type code
- Date format: `07 Apr 26`
- Types: DD, DEB, FPI, FPO, TFR, SO, CPT, COR

### Revolut Current
Columns: Date | Description | Money out | Money in | Balance
- Date format: `15 Apr 2026`
- Foreign sub-rows (Fee/Rate/EUR) are skipped

### Revolut Savings
Same columns as Revolut Current. Interest entries tagged as `interest` category.

### Amex Gold
- Statement closes 28th of each month — first one due 28 May 2026
- Parser written speculatively — test on first statement and check console
- If `[parseAmex] parsed 0 transactions` appears, the raw text is dumped to console for debugging
- Sections: "New Charges" (purchases), "Payments and Credits", "Fees and Adjustments"

---

## Categorisation

Priority order (first match wins):

1. Commission — FPI from INFINITY RENEWABLE → `commission`, `is_commission = true`
2. Grocery reimbursement — TFR/FPO out of Rent & Bills ≤ £300 referencing Main Current → `groceries`
3. Grocery reimbursement — TFR/FPI into Main Current ≤ £300 referencing Rent & Bills → `groceries`
4. Internal transfer — TFR/FPO/FPI referencing own account numbers or R LANGFORD → `transfer`, excluded from spend
5. Pattern rules (subscriptions, CS2, bills, groceries by merchant name)
6. Type fallbacks — FPI → income, DD/SO → bills, DEB → spending

### Key constants in index.html
```js
const RENT_AND_BILLS_ACC_NUM = '48939060';
const MAIN_CURRENT_ACC_NUM   = '39741868';
const GROCERY_BUDGET_CAP     = 300;
const OWN_ACCOUNT_MARKERS    = ['R LANGFORD','RUDI LANGFORD','LANGFORD R','LANGFORD RUDI','48939060','39741868','82501043'];
```

---

## Known issues / watch points

- **Amex parser** — speculative, untested until 28 May 2026 statement
- **Lloyds PDF column order** — parser now handles the space-joined labelled format; if amounts ever appear wrong, check whether a new statement format is being used
- **Grocery description matching** — Side A requires description to contain `39741868` or `R LANGFORD`; verify on a real Rent & Bills statement
- **Revolut Savings balance** — assumes last number on each line is the balance

---

## Things still to do

- [ ] Upload and verify Amex Gold statement (28 May 2026)
- [ ] Upload remaining Lloyds months (Nov–Mar) and Lloyds Credit statements
- [ ] Upload Revolut Current and Savings statements
- [ ] Verify Lloyds Credit parser works (same format as current — should just work)
- [ ] Manual category override (tap transaction → change category)
- [ ] Grocery spend tracker — £X of £300 budget used this month
- [ ] Search/filter transactions by description
- [ ] Export transactions as CSV
- [ ] Multi-month commission chart (needs 2+ months of data)
