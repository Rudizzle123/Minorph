# Minorph — Project Amendments (Session 6)

Update the project files before starting a new chat.

---

## Files to replace

### 1. `index.html` → replace entirely
Download the latest from Claude's output. Key changes this session:
- **`parseLloydsCredit` CR detection fixed** — payment rows (`PAYMENT RECEIVED – THANK YOU`) now correctly parse as `PAY` / `amountIn` and display green/positive.
- All Lloyds Platinum Credit statements deleted from Supabase and re-uploaded with the fix in place. Verified: November 2025 payment shows `+£84.25` (green), other payments equally correct.

### 2. `README.md` → replace entirely
Updated `parseLloydsCredit` section to reflect resolved CR bug and the actual token-tail strategy.

---

## What the bug was

The `PAYMENT row toks:` debug log revealed the actual issue: rows had trailing junk numbers **after** the card ref, e.g.
```
["PAYMENT","RECEIVED","-","THANK","YOU","251.96","CR","1880","17"]
```
The previous strip logic assumed the row ended with the card ref, so it never saw `CR`. The fix:
1. Scan the last 4 tokens for `CR` anywhere (not just last position)
2. Strip everything after the last money-shaped token (kills card ref + any trailing junk)

This is cause (b) from the original hypothesis — row fragmentation putting extra tokens at the tail — but the fragmentation was harmless once we stopped relying on positional strip.

The `month positions found: 111` is still high, but the over-matched month positions produce empty/noise rows that are correctly skipped by the existing `skip row (no amount)` guard. Not worth fixing unless it causes a real failure.

---

## Current live state

- App live at `minorph.pages.dev`
- Supabase project: `minorph` (URL `https://soplpclugrtwahtvlkdi.supabase.co`)
- User: `cookihd101@gmail.com`, UUID `5cebd01a-5b81-4bbd-9d2e-ca7f79ce020b`
- 6 accounts seeded, 2 goals seeded
- Lloyds Main Current: Nov 2025 – Apr 2026 ✅
- Rent & Bills: all months ✅
- Lloyds Credit: all statements re-uploaded with fix ✅
- Revolut Current + Savings: **not yet uploaded**
- Amex Gold: first statement arrives 28 May 2026

---

## Parser status after this session

| Parser | Status |
|---|---|
| Lloyds Current | ✅ Verified — all months Nov 2025–Apr 2026 |
| Lloyds Credit | ✅ Verified — payments now green/positive, purchases red/negative |
| Rent & Bills | ✅ Verified |
| Revolut Current | Written, untested |
| Revolut Savings | Written, untested |
| Amex Gold | Speculative — first statement arrives 28 May 2026 |

---

## Things to do next session

- [ ] Upload Revolut Current statements and verify parser
- [ ] Upload Revolut Savings statements and verify parser
- [ ] Test Amex Gold parser on 28 May 2026 statement
- [ ] Add duplicate statement check (idempotent re-upload)
- [ ] Update Amex goal `current_amount` once Amex parser is verified
- [ ] Manual category override (tap transaction → change category)
- [ ] Grocery spend tracker — £X of £300 budget used this month

---

## Rules agreed (see project instructions for full list)

1. **One instruction at a time** — give the single highest-confidence fix, wait for result, then move on.
2. **No code explanations in chat** — output the file, tell him what to do.
3. **HTML changes go in the file** — never paste big code blocks into the message.
4. **Standard workflow** — download → replace local → GitHub Desktop → push → Cloudflare auto-deploys.
5. **Read `AMENDMENTS.md` and `README.md` at session start** before touching anything.
6. **Parser failures: read the console first** — never guess, always ask for output before changing code.
