# Minorph — Project Amendments (Session 5)

Update the project files before starting a new chat.

---

## Files to replace

### 1. `index.html` → replace entirely
Download the latest from Claude's output. Key changes this session:
- All remaining **Lloyds Main Current** statements uploaded (Nov 2025 – Apr 2026) — all verified clean
- All **Rent & Bills** statements uploaded — all verified clean (12–16 transactions per month)
- **`parseLloydsCredit`** written and added — new dedicated parser for Lloyds Platinum Mastercard statements
  - Routed via `account.type === 'credit'` check in `startParse`
  - Token-walking strategy using full month names (OCTOBER, NOVEMBER…) as row delimiters
  - Correctly parses 50 transactions from a single statement
  - **KNOWN BUG: CR detection not working** — payment rows (`PAYMENT RECEIVED – THANK YOU`) are being stored as `PUR`/`amountOut` instead of `PAY`/`amountIn`. See below.
- `categorise()` updated: `PAY` type now returns `'income'` (previously missing from fallbacks)
- Debug logs added to `parseLloydsCredit`: `PAYMENT row toks:` log will fire on next upload to expose the raw token sequence

### 2. `README.md` → replace entirely
Updated parser notes to reflect new credit parser and known bug.

---

## Current live state

- App live at `minorph.pages.dev`
- Supabase project: `minorph` (URL `https://soplpclugrtwahtvlkdi.supabase.co`)
- User: `cookihd101@gmail.com`, UUID `5cebd01a-5b81-4bbd-9d2e-ca7f79ce020b`
- 6 accounts seeded, 2 goals seeded
- Lloyds Main Current: Nov 2025 – Apr 2026 ✅
- Rent & Bills: all months ✅
- Lloyds Credit: uploaded but **payment direction wrong** — needs fix + re-upload
- Revolut Current + Savings: not yet uploaded

---

## Parser status after this session

| Parser | Status |
|---|---|
| Lloyds Current | ✅ Verified — all months Nov 2025–Apr 2026, correct amounts |
| Lloyds Credit | ⚠️ Parsing 50 transactions correctly but CR (payment) detection broken — all rows stored as PUR/amountOut |
| Rent & Bills | ✅ Verified — all months clean |
| Revolut Current | Written, untested |
| Revolut Savings | Written, untested |
| Amex Gold | Speculative — first statement arrives 28 May 2026 |

---

## The CR bug — what's known

**Symptom:** `PAYMENT RECEIVED – THANK YOU` rows appear as `PUR` type with `amountOut` set, not `PAY`/`amountIn`. They display red and negative instead of green and positive.

**What the parser does:** After finding a row's token slice (between two month-name positions), it:
1. Strips trailing 4-digit card ref (e.g. `1880`)
2. Checks if last remaining token is `CR` → sets `isCR = true`
3. Strips a second card ref if present

**What the console shows:**
- `month positions found: 111` — far too many for one statement. The word `MAY` in the header (`35 | May | 2026`) and other incidental month words in prose are being picked up as row delimiters, fragmenting real rows.
- No `[parseLloydsCredit] CR row:` logs ever fire — meaning `isCR` never becomes true.
- The `PAYMENT row toks:` debug log (added last) hasn't been tested yet — **this is the next step**.

**Most likely cause:** The payment row is being split by a spurious month token mid-row (e.g. the word `NOVEMBER` appearing inside the payment description or in adjacent prose), so the `CR` token ends up in a different fragment than the amount, and neither fragment has both. Or `CR` is being tokenised differently (e.g. attached to the amount as `251.96CR` with no space).

**Next step for Opus:**
1. Push the current `index.html` (debug log already added), re-upload one credit statement, and grab the `[parseLloydsCredit] PAYMENT row toks:` console output.
2. That will show exactly what tokens the payment row contains and where `CR` sits relative to the amount.
3. Fix accordingly — likely either: (a) `CR` is attached to the amount token with no space, requiring the isMoney regex to handle `\d+\.\d{2}CR`, or (b) the row is being fragmented by a spurious month token inside the prose.

---

## Things to do next session

- [ ] Fix `parseLloydsCredit` CR detection (see bug notes above)
- [ ] Delete Lloyds Credit data from Supabase and re-upload all statements once fixed
- [ ] Upload Revolut Current and Savings statements
- [ ] Test Amex Gold parser on 28 May 2026 statement
- [ ] Add duplicate statement check (idempotent re-upload) — agreed this is worth adding
- [ ] Update Amex goal `current_amount` once Amex parser is verified

---

## Rules agreed (see project instructions for full list)

1. **One instruction at a time** — give the single highest-confidence fix, wait for result, then move on.
2. **No code explanations in chat** — output the file, tell him what to do.
3. **HTML changes go in the file** — never paste big code blocks into the message.
4. **Standard workflow** — download → replace local → GitHub Desktop → push → Cloudflare auto-deploys.
5. **Read `AMENDMENTS.md` and `README.md` at session start** before touching anything.
6. **Parser failures: read the console first** — never guess, always ask for output before changing code.
