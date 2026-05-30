# Minorph — Project Amendments (Session 10)

Update the project files before starting a new chat.

---

## Files to replace

### 1. `index.html` → replace entirely

Two things shipped this session:

**A) Pension / ANI Tracker (Pulse dashboard)**
- New card between the Pulse grid and Wealth Snapshot section
- Auto-aggregates `FSTL PRIMARY TRUST` (Fidelity SIPP) outflows from Lloyds Main Current over the current UK tax year (6 Apr → 5 Apr)
- Grosses them up ×1.25 (basic-rate relief)
- Subtracts grossed total from `annual_gross_income` (stored in new `user_settings` table) to estimate ANI
- Visual progress bar with £100k threshold line; ANI tinted green / amber (<£5k headroom) / red (over)
- Settings cog opens modal to set/edit the annual gross income figure (single field — base + expected bonuses)
- Verified: TY 2026/27 shows £1,700 cash → £2,125 grossed → £140k − £2,125 = £137,875 ANI ✅

**B) Where It Went (Transactions tab)**
- New segmented toggle at the top of Transactions: **All transactions** vs **Where it went**
- "Where it went" panel includes:
  - Month picker populated from months with Infinity FPI deposits
  - Total Spend + Subscriptions summary tiles (with 6-mo avg + delta %)
  - "By category" list — ranked bars, % of total, % delta vs 6-mo avg
  - "Top merchants" list — top 10 descriptions, normalised by `cleanMerchantName()`
- Spend totals exclude: internal transfers, commission, interest, income categories
- Also excludes by description pattern: `FIDELITY | FIL\s*INV | FIL\s*LIFE | FIL\s*SIPP | FIL\s*ISA | FSTL PRIMARY TRUST | FASL PRIM CLIENT | REVOLUT | PLATINUM CREDIT | LLOYDS CREDIT CARD` — these are wealth movements / credit-card payoffs, not consumption
- All transactions are fetched once and cached client-side in `spendCache`

### 2. `README.md` → replace entirely
Schema updated (new `user_settings` table), Pulse dashboard now documents the pension card, Transactions docs now cover the Where It Went breakdown.

### 3. Project instructions → no change this session.

---

## Pending SQL migration (run in Supabase SQL editor before testing)

```sql
create table if not exists user_settings (
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
- Wealth snapshot for Apr/May 2026 seeded
- Net worth (Apr 2026): **£84,142.32**
- Annual gross income: **£140,000** (working estimate — base £35k + expected bonuses up to ~£105k)
- TY 2026/27 ANI estimate: **£137,875** (only £1,700 SIPP cash so far — needs aggressive contribs to get under £100k)
- TY 2026/27 SIPP cash YTD: **£1,700** → grossed **£2,125**

---

## Parser status

| Parser | Status |
|---|---|
| Lloyds Current | ✅ Verified — all months Nov 2025–Apr 2026 |
| Lloyds Credit | ✅ Verified |
| Rent & Bills | ✅ Verified |
| Revolut Current | ✅ Verified |
| Revolut Savings | ✅ Verified |
| Amex Gold | Speculative — first statement arrives 28 May 2026 |
| Fidelity (SIPP + ISA quarterly statements) | ⚠️ Not yet built — manual snapshot input for now |

---

## Things to do next session (priority order)

1. **Categorisation cleanup** — the Where It Went view exposed several uncategorised merchants and a few miscategorised ones. Specifics to look at:
   - **"Spending" bucket is too generic** — DEB fallback dumps a lot into it (e.g. `GI GI FIRECRACKER £1,045`, `LC INTERNATIONAL £1,019`, `DAMIRA QUAYSIDE`, `RORY PACK`). Need to either add merchant rules for the regulars or rename "spending" to something clearer (e.g. "uncategorised")
   - **"Other" bucket** — Bills/CS2 category rules look fine but a chunk of legitimate spend is still falling through to `other`. Likely Amex purchases (when they land) and one-offs
   - **Hastings Insurance** — currently tagged as subscription (correct intent), but appears in Top Merchants list with the bulb icon — icon logic for subscription category is mis-routing. Worth a look
   - **Top merchant cleanup** — `cleanMerchantName()` is fine but a few descriptions still come through as partial company names (`FASL PRIM CLIENT B`, `FSTL PRIMARY TRUST` — these now excluded so not visible, but the pattern applies to others)
   - Add merchant rules + icons for: GI GI FIRECRACKER, LC INTERNATIONAL, DAMIRA QUAYSIDE, RORY PACK, CSFLOAT INC
2. **Wealth long-game chart (Screen 3)** — net worth line chart broken down by SIPP / ISA / CS2 / Savings
3. **Fidelity PDF parser** — once Rudi has a few historic quarterly statements
4. **Amex Gold parser test** — 28 May 2026 statement arriving
5. **Search/filter transactions by description**
6. **Export transactions as CSV**

---

## Open questions for next session

- For ANI tracker: Rudi will update `annual_gross_income` post self-assessment once a year. Should there also be a "lock" toggle to mark TY as final once self-assessment is filed? (Probably no — keep it simple, just an editable number.)
- Whether to add a manual "add SIPP contribution" entry (for months where statement isn't uploaded yet), or wait for the Fidelity parser to land.

---

## Rules agreed (see project instructions for full list)

1. One instruction at a time
2. No code explanations in chat
3. HTML changes go in the file
4. Standard workflow: download → replace local → GitHub Desktop → push → Cloudflare auto-deploys
5. Read `AMENDMENTS.md` and `README.md` at session start
6. Parser failures: read the console first
7. Big architectural/UX decisions: break the brevity rule and have a proper design discussion
