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
1. Edit `index.html` locally (replace with file from Claude)
2. GitHub Desktop → commit → push to `main`
3. Cloudflare auto-deploys to `minorph.pages.dev`

### If Cloudflare Git connection breaks
Settings → Git repository → reconnect GitHub, then re-deploy.

### To clear bad parse data from Supabase
```sql
delete from transactions where account_id = (select id from accounts where name = 'Platinum Credit');
delete from statements  where account_id = (select id from accounts where name = 'Platinum Credit');
```
To nuke everything:
```sql
delete from transactions;
delete from statements;
```

---

## PDF Parsing

### Parser routing (in `startParse`)
```
Lloyds + account.type === 'credit'  → parseLloydsCredit
Lloyds (all others)                 → parseLloyds
Revolut + account.name includes 'saving' → parseRevolutSavings
Revolut (all others)                → parseRevolutCurrent
Amex / American Express             → parseAmex
```

### Lloyds Current (token-walking parser)
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
- **Debugging:** logs token count, first 30 tokens, date positions found, parsed count, sample of 3, `skip row (no type)` per skipped row

### Lloyds Credit (parseLloydsCredit) — ✅ working
Lloyds Platinum Mastercard PDF layout differs entirely from current accounts:
- No date column — rows begin with a **full month name** (OCTOBER, NOVEMBER…)
- Format per row: `MONTHNAME DESCRIPTION AMOUNT [CR] CARD_REF [trailing_junk]`
  - `CARD_REF` = 4-digit fixed number (last 4 of card, e.g. `1880`) — NOT a balance
  - `CR` suffix = payment received (amountIn); no suffix = purchase (amountOut)
  - Trailing junk tokens can appear after the card ref due to row fragmentation
- No running balance per row

**Strategy:**
1. Tokenise full text
2. Extract statement year (first `20XX` token)
3. Find all full month-name token positions — these delimit rows
4. For each row slice:
   - Scan the **last 4 tokens** for `CR` anywhere → sets `isCR`
   - Find the **last money-shaped token** (`\d+\.\d{2}`) and strip everything after it — this kills the card ref and any trailing junk in one move
   - Last money token is the amount; everything before it is the description
5. `CR` → type `PAY`, amountIn; no CR → type `PUR`, amountOut

**Notes:**
- `month positions found:` will be high (~110 for a single statement) because month words in header/footer prose are matched. Most produce empty or noise slices that get skipped by the "no amount" guard. Don't worry about the count.
- Skipped rows are logged as `[parseLloydsCredit] skip row (no amount):` for visibility.

### Revolut Current
Columns: Date | Description | Money out | Money in | Balance
- Date format: `15 Apr 2026`
- Foreign-currency sub-rows (Fee/Rate/EUR) are skipped
- Written, **untested on real data**

### Revolut Savings
Same columns as Revolut Current. Interest entries tagged as `interest` category.
- Written, **untested on real data**

### Amex Gold
- Statement closes 28th of each month — first one due 28 May 2026
- Parser written speculatively — test on first statement and check console
- If `[parseAmex] parsed 0 transactions` appears, raw text is dumped to console

---

## Categorisation

Priority order (first match wins):

1. Grocery reimbursement — TFR/FPO out of Rent & Bills ≤ £300 referencing Main Current → `groceries`
2. Grocery reimbursement — TFR/FPI into Main Current ≤ £300 referencing Rent & Bills → `groceries`
3. Internal transfer — TFR/FPO/FPI referencing own account numbers or R LANGFORD → `transfer`, excluded from spend
4. Commission — FPI from `INFINITY RENEWABLE` → `commission`, `is_commission = true`
5. Revolut interest — type `INT` → `interest`
6. Pattern rules (subscriptions, CS2, bills, groceries by merchant name)
7. Type fallbacks — FPI/BGC/DEP/PAY → income, DD/SO → bills, DEB → spending

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
- **Lloyds TYPE collisions** — TYPE set is intentionally narrowed. If a future statement uses a code outside `DD/DEB/FPI/FPO/TFR/SO/CPT/COR/BGC/CHQ/ATM`, that row will be skipped
- **Grocery description matching** — Side A requires description to contain `39741868` or `R LANGFORD`; verify on real Rent & Bills data
- **Revolut Savings balance** — assumes last number on each line is the balance
- **Lloyds Credit month over-matching** — ~110 month positions per statement is normal; over-matches produce empty slices that skip cleanly. Not a bug, just noisy logs.

---

## Things still to do

- [ ] Upload Revolut Current and Savings statements
- [ ] Upload and verify Amex Gold statement (28 May 2026)
- [ ] Update Amex goal `current_amount` once Amex parser verified
- [ ] Add duplicate statement guard (idempotent re-upload)
- [ ] Manual category override (tap transaction → change category)
- [ ] Grocery spend tracker — £X of £300 budget used this month
- [ ] Search/filter transactions by description
- [ ] Export transactions as CSV
- [ ] Multi-month commission chart (needs 2+ months of data)
