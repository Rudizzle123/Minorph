# Minorph — Project Amendments (Session 4)

Update the project files before starting a new chat.

---

## Files to replace

### 1. `index.html` → replace entirely
Download the latest from Claude's output. Key changes this session:
- `parseLloyds` rewritten with a **token-walking** approach (replaces the regex + line-fallback strategy that was producing 87 zero-amount rows)
  - Tokenises whitespace-cleaned text after stripping column labels (`Date`, `Description`, `Type`, `Money In (£)`, `Money Out (£)`, `Balance (£)`, `blank.`, `(£)`)
  - Walks tokens, treating each `day Mon yy` triplet as the start of a new row
  - Within each row, finds the TYPE token, takes everything before it as the description, everything after as numbers (last = balance, first = amount)
  - Type set deliberately narrowed to `DD, DEB, FPI, FPO, TFR, SO, CPT, COR, BGC, CHQ, ATM` to avoid description words like "PAY" being mis-detected as TYPE codes
  - Strips leading/trailing non-alphanumeric chars from descriptions so rows render as `TAYLOR GREEN` not `. TAYLOR GREEN .`
- Verbose `console.log` left in for next time debugging is needed: `tokens count`, `date positions found`, `parsed N transactions`, `sample`, plus `skip row (no type)` per skipped row
- Line-by-line fallback removed (was producing the zero-amount garbage)

### 2. `README.md` → replace entirely
Updated parser notes to reflect the new token-walking strategy.

---

## Current live state

- App live at `minorph.pages.dev`
- Supabase project: `minorph` (URL `https://soplpclugrtwahtvlkdi.supabase.co`)
- User: `cookihd101@gmail.com`, UUID `5cebd01a-5b81-4bbd-9d2e-ca7f79ce020b`
- 6 accounts seeded, 2 goals seeded
- Cloudflare Pages connected to `Rudizzle123/Minorph` on GitHub — auto-deploys on push to `main`

---

## Parser status after this session

| Parser | Status |
|---|---|
| Lloyds Current | ✅ Verified — April 2026 statement returned 70 real transactions with correct amounts, descriptions, and balances |
| Lloyds Credit | Untested — same Lloyds format, should work |
| Revolut Current | Written, untested |
| Revolut Savings | Written, untested |
| Amex Gold | Speculative — first statement arrives 28 May 2026 |

---

## Things to do next session

- [ ] Upload remaining Lloyds Main Current months (Nov 2025 – Mar 2026)
- [ ] Upload Lloyds Rent & Bills statements (verify grocery reimbursement logic works on real data)
- [ ] Upload Lloyds Credit statements (sanity-check Lloyds parser holds for credit format)
- [ ] Upload Revolut Current and Savings statements (first real test of those parsers)
- [ ] Test Amex Gold parser on 28 May 2026 statement
- [ ] Update Amex goal `current_amount` once the first Amex statement is processed
- [ ] Refresh README and AMENDMENTS after the session

---

## Rules agreed for next Claude (see project instructions for full list)

1. **One instruction at a time** — give the single highest-confidence fix, wait for the result, then move on. Never list options or multi-step plans.
2. **No code explanations in chat** — Rudi doesn't need the why. Output the file and tell him what to do.
3. **HTML changes go in the file, not the chat** — never paste big code blocks into the message. Always download a fresh `index.html`.
4. **Standard workflow** — Rudi downloads `index.html`, replaces local copy, commits and pushes via GitHub Desktop, Cloudflare auto-deploys. Don't suggest alternative workflows.
5. **Trust the project files** — read `AMENDMENTS.md` and `README.md` at the start of every session to pick up current state.
