# Minorph — Project Amendments (Session 8)

Update the project files before starting a new chat.

---

## Files to replace

### 1. `index.html` → replace entirely (use the most recent output from this session)
Key changes this session:
- **Duplicate statement guard** — `startParse` now checks for an existing statement with the same `account_id` + `period_end` before inserting. Aborts with a toast showing which statement already exists. Check runs at 85% progress, before any DB write.
- **Manual category override** — tapping any transaction opens a bottom sheet with 12 category options. Saves `category` + `category_override: true` to Supabase. Overridden transactions show a small purple "edited" pill. Requires `category_override boolean default false` column on `transactions` table (SQL migration below).
- **Expanded `getCategoryIcon`** — far more description-based matches. Covers Deliveroo/Uber Eats/Just Eat (🛵), all major supermarkets (🛒), restaurants/cafes/fast food (🍽️), fuel/service stations (⛽), Amazon (📦), Smyths/toys (🧸), transport/rail/taxi (🚂/🚕), parking (🅿️), hospitals/pharmacies (🏥), gaming (🎮), utilities (💡/💧/🌐), insurance (🛡️), and more. Fallback is now 💳 instead of a dot.
- **Bills tab** — Subs screen renamed "Recurring" with a Subscriptions / Bills segment toggle. Bills are fully manual (new `bills` Supabase table): add, edit amount/cycle/due day, delete. Requires SQL migration below.
- **Subscription editing** — tapping a subscription card opens the bill modal pre-filled, allowing amount and cycle edits, or deletion.

### 2. `README.md` → replace entirely
Updated schema (two new tables), known issues, and to-do list.

---

## Pending SQL migrations (run in Supabase SQL editor before testing)

```sql
-- 1. Category override flag on transactions
ALTER TABLE transactions ADD COLUMN IF NOT EXISTS category_override boolean default false;

-- 2. Bills table
create table if not exists bills (
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

---

## Current live state

- App live at `minorph.pages.dev`
- Supabase project: `minorph` (URL `https://soplpclugrtwahtvlkdi.supabase.co`)
- User: `cookihd101@gmail.com`, UUID `5cebd01a-5b81-4bbd-9d2e-ca7f79ce020b`
- 6 accounts seeded, 2 goals seeded
- Lloyds Main Current: Nov 2025 – Apr 2026 ✅
- Rent & Bills: all months ✅
- Lloyds Credit: all statements ✅
- Revolut Current: May 2026 ✅
- Revolut Savings: May 2026 ✅
- Amex Gold: first statement arrives 28 May 2026

---

## Parser status

| Parser | Status |
|---|---|
| Lloyds Current | ✅ Verified — all months Nov 2025–Apr 2026 |
| Lloyds Credit | ✅ Verified — payments green, purchases red |
| Rent & Bills | ✅ Verified |
| Revolut Current | ✅ Verified — 20 transactions, internal transfers excluded |
| Revolut Savings | ✅ Verified — 46 transactions, interest tagged correctly |
| Amex Gold | Speculative — first statement arrives 28 May 2026 |

---

## Things to do next session

- [ ] Test Amex Gold parser on 28 May 2026 statement
- [ ] Update Amex goal `current_amount` once Amex parser verified
- [ ] Grocery spend tracker — £X of £300 budget used this month
- [ ] Search/filter transactions by description
- [ ] Export transactions as CSV

---

## Rules agreed (see project instructions for full list)

1. **One instruction at a time** — give the single highest-confidence fix, wait for result, then move on.
2. **No code explanations in chat** — output the file, tell him what to do.
3. **HTML changes go in the file** — never paste big code blocks into the message.
4. **Standard workflow** — download → replace local → GitHub Desktop → push → Cloudflare auto-deploys.
5. **Read `AMENDMENTS.md` and `README.md` at session start** before touching anything.
6. **Parser failures: read the console first** — never guess, always ask for output before changing code.
