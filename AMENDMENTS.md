# Minorph — Project Amendments (Session 2)

Changes made this session. Update the project files listed below before
starting a new chat so the next session has full context.

---

## Files to replace in the project

### 1. `index.html` → replace entirely
The full app is built. Replace the old mockup (`minorph-v2.html` content)
with the new `index.html` from this session's output.

Key things now in `index.html`:
- Supabase auth (sign in / sign up, session persistence)
- All 5 screens wired to real Supabase data
- Lloyds PDF parser (regex + line fallback)
- Revolut Current PDF parser (skips foreign sub-rows)
- Revolut Savings PDF parser (interest tagged separately)
- Amex Gold PDF parser (speculative — test on 28 May statement)
- Full categorisation pipeline with grocery logic
- Subscription auto-detection on upload
- Commission month-on-month Chart.js bar chart
- Toast notifications, parse progress bar, upload history

**Before deploying:** paste your Supabase URL and anon key into the
two constants at the very top of the file:
```js
const SUPABASE_URL  = 'YOUR_SUPABASE_URL';
const SUPABASE_ANON = 'YOUR_SUPABASE_ANON_KEY';
```

### 2. `README.md` → replace entirely
Updated README reflects the full built state, correct Supabase schema
with RLS policies, seed SQL for accounts and goals, and setup steps.

---

## Logic decisions made this session (for context in next chat)

### Grocery reimbursement — two-sided tagging
The £300/mo grocery budget works like this:
1. Groceries bought on any card (Amex, Lloyds credit, Main Current debit)
2. Exact amount transferred from **Rent & Bills → Main Current** to reimburse
3. End of month: credit cards paid off from the Main Current float

Two transaction sides are tagged `groceries` (not `transfer`):

**Side A — Rent & Bills statement (outgoing):**
- Type: TFR or FPO
- Direction: amountOut > 0
- Amount: ≤ £300 (`GROCERY_BUDGET_CAP` constant — raise if needed)
- Description references: `39741868` OR `R LANGFORD` / `RUDI LANGFORD`

**Side B — Main Current statement (incoming credit):**
- Type: TFR or FPI
- Direction: amountIn > 0
- Amount: ≤ £300
- Description references: `48939060`

Both sides have `is_internal_transfer = false` so they appear in spend
analysis rather than being hidden.

The actual grocery purchase on the card (TESCO, LIDL etc) is tagged
`groceries` separately via the supermarket name pattern rules.

### Categorisation rule audit
- Bare `STEAM` removed — too ambiguous (also a UK train company)
- `STEAMGAMES` and `STEAM PURCHASE` added instead
- `SKINLEDGER` added as CS2
- `EE` pattern widened to catch `EE DD` variant
- `WATER` rule tightened to named suppliers only
- Several new subscription patterns added (Disney+, Now TV, Adobe, Microsoft 365)

### Commission chart
- Chart.js bar chart added to Dashboard below commission income bar
- Queries all `is_commission = true` transactions, groups by YYYY-MM
- Shows from 1 month of data upwards — grows automatically
- Total and monthly average legend rendered below chart
- Chart instance stored in `commissionChartInstance` — destroyed before
  re-render to prevent Canvas reuse warnings

### Amex parser
- Written speculatively against standard Amex UK PDF format
- Reuses `parseLloydsDate()` — same `DD Mon YY` format
- Skips foreign currency sub-rows (USD, EUR, etc)
- Synthesised type codes: `PUR` (purchase), `PAY` (payment), `FEE` (fee)
- No per-row balance (`runningBalance: null`)
- Console logs raw PDF text on zero results to aid debugging
- **Must be tested against actual May 2026 statement**

---

## Things still to do (carry into next chat)

### Immediate (28 May 2026 — when Amex statement arrives)
- [ ] Upload first Amex statement, check console for parse errors
- [ ] Adjust `parseAmex()` if section headers or amount format differ
- [ ] Update Amex Goal `current_amount` in Supabase

### Desktop setup (when back from Greece)
- [ ] Create GitHub repo: `github.com/Rudizzle123/minorph`
- [ ] Create Supabase project (new, not Infinity Renewables)
- [ ] Run schema SQL from README in Supabase SQL editor
- [ ] Paste Supabase URL + anon key into `index.html` config
- [ ] Sign up in the app → copy User ID → run seed SQL
- [ ] Connect repo to Cloudflare Pages → deploy

### Future features (no priority order)
- [ ] Lloyds Credit Card parser (same format as Lloyds current — should work already, verify)
- [ ] Multi-month commission chart once 2+ months of data available
- [ ] Category breakdown chart per account (pie/donut)
- [ ] Manual transaction category override (tap transaction → change category)
- [ ] Grocery spend tracking — show £X of £300 budget used this month
- [ ] Search/filter transactions by description text
- [ ] Export transactions as CSV

---

## Constants to know (in `index.html`)

```js
const RENT_AND_BILLS_ACC_NUM = '48939060';
const MAIN_CURRENT_ACC_NUM   = '39741868';
const GROCERY_BUDGET_CAP     = 300;  // raise if grocery reimbursements ever exceed £300

const OWN_ACCOUNT_MARKERS = [
  'R LANGFORD', 'RUDI LANGFORD', 'LANGFORD R', 'LANGFORD RUDI',
  '48939060', '39741868', '82501043',
];
```

---

## Known unknowns to watch when uploading real statements

1. **Lloyds PDF column order** — the parser handles both regex and
   line-by-line fallback. If transactions parse with wrong in/out
   direction, check whether Lloyds puts Money In before or after
   Money Out in the actual PDF (it varies by statement type).

2. **Amex section headers** — `parseAmex()` looks for "New Charges",
   "Payments and Credits", "Fees and Adjustments". If Amex uses
   different wording the section context will be wrong and direction
   may be misclassified. Fix: update `SECTION_CHARGES` / `SECTION_PAYMENTS`
   regex patterns in `parseAmex()`.

3. **Grocery description text** — Side A requires the description to
   contain `39741868` or `R LANGFORD`. Check a real Rent & Bills
   statement to confirm Lloyds includes the destination account number
   in TFR descriptions. If not, the fallback is to match on the
   reference text Lloyds uses for own-account transfers.

4. **Revolut Savings balance** — the parser assumes the last number on
   each line is the balance. If Revolut changes column order this breaks.
