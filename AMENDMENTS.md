# Minorph — Project Amendments (Session 7)

Update the project files before starting a new chat.

---

## Files to replace

### 1. `index.html` → replace entirely
Key changes this session:
- **Revolut internal transfer detection** — `isInternalTransfer()` now catches `"To GBP Savings"` / `"From GBP Savings"` descriptions before the type guard runs (Revolut transactions have `type: null`, so the previous `TRANSFER_TYPES` check always missed them). Regex: `/^(TO|FROM)\s+GBP\s+(SAVINGS|CURRENT)/i`
- **Dashboard 400 error fixed** — `loadDashboard` was using `.eq('accounts.user_id', ...)` on a join column which Supabase PostgREST rejects with a 400. Fixed to `.in('account_id', accountIds)` where `accountIds` is derived from the already-loaded `userAccounts` array. Limit bumped from 20 → 50.

### 2. `README.md` → replace entirely
Updated parser status, known issues, and to-do list.

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
- [ ] Add duplicate statement check (idempotent re-upload)
- [ ] Update Amex goal `current_amount` once Amex parser is verified
- [ ] Manual category override (tap transaction → change category)
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
