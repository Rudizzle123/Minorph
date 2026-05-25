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

### Lloyds Credit (parseLloydsCredit) — ⚠️ CR BUG UNRESOLVED
Lloyds Platinum Mastercard PDF layout differs entirely from current accounts:
- No date column — rows begin with a **full month name** (OCTOBER, NOVEMBER…)
- Format per row: `MONTHNAME DESCRIPTION AMOUNT [CR] CARD_REF`
  - `CARD_REF` = 4-digit fixed number (last 4 of card, e.g. `1880`) — NOT a balance
  - `CR` suffix = payment received (amountIn); no suffix = purchase (amountOut)
- No running balance per row

**Current strategy:**
1. Tokenise full text
2. Extract statement year (first `20XX` token)
3. Find all full month-name token positions — these delimit rows
4. For each row: strip trailing card ref, detect `CR`, find last `\d+\.\d{2}` as amount
5. `CR` → type `PAY`, amountIn; no CR → type `PUR`, amountOut

**Known bug:** `isCR` never fires. Payment rows are stored as `PUR`/`amountOut`.
- Root cause not yet confirmed. Two candidates:
  - (a) `CR` is joined to the amount with no space (e.g. `251.96CR`) so it's not a standalone token
  - (b) The row is fragmented by a spurious month token inside the payment description prose, putting `CR` in a different row slice than the amount
- `month positions found: 111` for a single statement confirms heavy over-matching — month words in header/footer prose are being treated as row starters
- Debug log `[parseLloydsCredit] PAYMENT row toks:` has been added to the current `index.html` — **this log needs to be captured to diagnose the bug**

**To fix:** Push current `index.html`, re-upload one credit statement, grab the `PAYMENT row toks:` console output, then fix based on what the tokens actually look like.

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

- **Lloyds Credit CR bug** — payment rows stored as PUR/amountOut. Debug log added, needs one more upload to diagnose. See parser notes above.
- **Lloyds Credit month over-matching** — 111 month positions found in a single statement. Month words in header/footer prose (e.g. `May` in `35 | May | 2026`) are triggering false row starts. The real fix may need a stricter row-start heuristic (e.g. require the month token to be followed by a money-like pattern within N tokens).
- **Amex parser** — speculative, untested until 28 May 2026 statement
- **Lloyds TYPE collisions** — TYPE set is intentionally narrowed. If a future statement uses a code outside `DD/DEB/FPI/FPO/TFR/SO/CPT/COR/BGC/CHQ/ATM`, that row will be skipped
- **Grocery description matching** — Side A requires description to contain `39741868` or `R LANGFORD`; verify on real Rent & Bills data
- **Revolut Savings balance** — assumes last number on each line is the balance

---

## Things still to do

- [ ] Fix `parseLloydsCredit` CR detection (debug log added, needs one upload to diagnose)
- [ ] Delete and re-upload all Lloyds Credit statements once fix is confirmed
- [ ] Add duplicate statement guard (idempotent re-upload) — agreed, worth adding
- [ ] Upload Revolut Current and Savings statements
- [ ] Upload and verify Amex Gold statement (28 May 2026)
- [ ] Update Amex goal `current_amount` once Amex parser verified
- [ ] Manual category override (tap transaction → change category)
- [ ] Grocery spend tracker — £X of £300 budget used this month
- [ ] Search/filter transactions by description
- [ ] Export transactions as CSV
- [ ] Multi-month commission chart (needs 2+ months of data)
