# Minorph — Project Amendments (Session 9)

Update the project files before starting a new chat.

---

## Files to replace

### 1. `index.html` → replace entirely (use the most recent output from this session)

This session was a strategy + UX session. The dashboard has been rebuilt around a clearer vision: Minorph is now a **monthly personal CFO dashboard**, not a budgeting app. Key changes:

- **Dashboard renamed conceptually to "Pulse"** — still the first tab, still labelled "Dashboard" in the nav. Old Total Balance hero, account cards, and commission section are all gone.
- **Net Worth hero** — gradient white→lilac headline number, soft glow, with date chip + breakdown chips (SIPP / ISA / CS2 / Savings) below.
- **Sparkline canvas** under the net worth number — wired up, plots all wealth snapshots over time. Container is fixed-height (56px) so Chart.js can't blow up the layout.
- **4-tile Pulse grid** — Net Pay (Month), Savings Rate, Invested (Month), Discretionary. Each shows current-month value + 6-month rolling avg. Discretionary shows % delta vs 6-mo avg with colour.
- **Wealth Snapshot section** — manual monthly input card for SIPP balance, SIPP YTD contrib (cash), ISA balance, ISA YTD contrib, CS2 portfolio value. Tap "＋ Add / Edit" to add or edit any month.
- **CS2 Investments section** — manual purchase log (date / amount / notes). Each entry counts toward "Invested this month".
- **Net Pay month-on-month chart** — replaces the old commission chart. **Flame palette** (orange `#ff8a3d` / light `#ffb267`) instead of green. Fixed 140px height. Shows total + monthly avg legend.
- **Dashboard period anchoring** — header date and all 4 Pulse tiles anchor on the latest month with Infinity FPI data (falls back to latest statement, then current month). This stops the dashboard showing empty £0 numbers for a current month that hasn't been uploaded yet.
- **Fidelity merchant detection** — invested tracker now detects `FSTL PRIMARY TRUST` (SIPP) and `FASL PRIM CLIENT B` (ISA) on Lloyds Main Current transactions, plus generic `FIDELITY` / `FIL SIPP` / `FIL ISA` patterns.

### 2. `README.md` → replace entirely
Updated schema (two new tables), parser status, Pulse dashboard documentation.

### 3. Project instructions → no change this session (rules and workflow are unchanged)

---

## Pending SQL migrations (run in Supabase SQL editor before testing)

```sql
-- Wealth snapshots — monthly manual entry of SIPP / ISA / CS2 balances + contribs
create table if not exists wealth_snapshots (
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

-- Manual investments — CS2 purchase log (and any future manual investment types)
create table if not exists manual_investments (
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
- **Wealth snapshot for Apr/May 2026** seeded: SIPP £3,827.70 / ISA £39,705.60 / CS2 £38,919.72
- Net worth (Apr 2026): **£84,142.32**

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
| Fidelity (SIPP + ISA quarterly statements) | ⚠️ Not yet built — manual snapshot input for now |

---

## The Vision (locked this session)

Minorph is a **monthly wealth-review tool**, not a daily budgeting app. The four questions it answers:

1. **Where am I leaking money?** — Discretionary tile, category breakdowns (future Spend screen)
2. **Am I getting ahead?** — Net Worth + sparkline, Net Pay rolling avg, Savings Rate, Invested Rate
3. **Am I optimising tax?** — Pension tracker (next session) showing tax-year SIPP contribs grossed up + ANI vs £100k threshold
4. **Am I on pace for Amex points?** — existing Goals screen

**Wealth model:** Net worth = SIPP + S&S ISA + CS2 + Revolut Savings only. Current accounts and credit cards are operational, not wealth.

**Income model:** All Infinity Renewables FPI = net pay (PAYE salary + bonus). "Commission" is just the legacy tag — display now says "Net Pay".

**Pension model:** Personal SIPP contributions via Fidelity. Cash contribution × 1.25 = grossed-up figure that reduces Adjusted Net Income. Target: keep ANI below £100k.

---

## Things to do next session (priority order)

1. **Pension tracker / ANI estimate** — tax-year SIPP contribution total, grossed up by ÷0.8, subtracted from estimated gross income, displayed vs £100k threshold. Auto-detect from Lloyds FSTL transactions (already partially wired for invested rate, just needs the pension-specific aggregation + tax year boundaries).
2. **Spend leak finder (Screen 2)** — category breakdown for the month, ranked by £, with % delta vs 6-mo avg. Top 10 merchants. Subscription audit total.
3. **Wealth long-game chart (Screen 3)** — net worth line chart broken down by SIPP / ISA / CS2 / Savings.
4. **Fidelity PDF parser** — once Rudi has a few historic quarterly statements, build a parser. Statement structure is known: ISA + SIPP + Cash Mgmt sections in one PDF, dates as `DD.MM.YY`, contribution lines start with "Money paid in".
5. **Amex Gold parser** — test on 28 May 2026 statement
6. **Search/filter transactions by description**
7. **Export transactions as CSV**

---

## Open questions for next session

- Rudi's pension contributions are personal Fidelity SIPP (not salary sacrifice). When building ANI tracker, need to know if he wants estimated gross income hard-coded, derived from net pay × tax-multiplier, or as another input.
- Whether savings rate denominator should stay as net pay only, or eventually include investment outflows too (currently: savings deposits / net pay).

---

## Rules agreed (see project instructions for full list)

1. **One instruction at a time** — give the single highest-confidence fix, wait for result, then move on.
2. **No code explanations in chat** — output the file, tell him what to do.
3. **HTML changes go in the file** — never paste big code blocks into the message.
4. **Standard workflow** — download → replace local → GitHub Desktop → push → Cloudflare auto-deploys.
5. **Read `AMENDMENTS.md` and `README.md` at session start** before touching anything.
6. **Parser failures: read the console first** — never guess, always ask for output before changing code.
7. **Big architectural/UX decisions are vision conversations** — exception to rule 5: when Rudi asks "what should we build next" or "give me your take on X", break the brevity rule and have a proper design discussion.
