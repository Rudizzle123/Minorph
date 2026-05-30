# Minorph — Project Amendments (Session 11)

Update the project files before starting a new chat.

---

## Files to replace

### 1. `index.html` → replace entirely

One thing shipped this session:

**Amex Gold parser (`parseAmex`) — rewritten and verified**
- The Session-10 parser was speculative and assumed a `01 Apr 26` date format with description + amount on one line. The real 29 Apr–28 May 2026 statement is nothing like that.
- Real PDF.js extraction:
  - Rows are `MonName Day  MonName Day  DESCRIPTION` (two date columns, `May 6` style, **no year on the row**).
  - The **GBP amount sits on its own separate line** below the description.
  - Year is inferred from the period header (`From 29 April to 28 May 2026`) with a rollover guard.
  - GBP amount = last money token on the (possibly joined) line.
  - Noise lines (`GOODS`, `TICKET NUMBER: … PASSENGER NAME:`) skipped, but a real row sometimes shares a line after a noise fragment — parser reclaims the tail from the first date token.
- Rewrote as an indexed loop with amount look-ahead (up to 3 lines) + noise-prefix recovery.
- Console now logs parsed count, sample rows, and net spend total.
- Debug stash line (`window.__lastAmexText`) used during debugging has been removed.
- **Verified live: 16 transactions, net spend £1,168.78 — matches statement closing balance exactly.**

### 2. `README.md` → replace entirely
Amex parser section rewritten (now ✅ verified with the full real-format description). Removed from known-issues and to-do list. Parser status table updated.

### 3. Project instructions → no change this session.

---

## Pending SQL migration
None this session.

> ⚠️ Note on the Amex statement period: when only a partial parse succeeds, `period_end` gets set from the latest *parsed* transaction date, not the statement header. During this session two failed Amex statement rows (`period_end = 2026-05-27`) were left behind and had to be deleted manually by id before the duplicate guard would allow a clean re-upload. The fully-parsed statement spans through 27 May so `period_end` lands on 2026-05-27 regardless — fine, but worth knowing the header says 28 May.

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
- **Amex Gold: May 2026 ✅ (first real statement, 16 txns, £1,168.78)**
- Wealth snapshot for Apr/May 2026 seeded
- Net worth (Apr 2026): **£84,142.32**
- Annual gross income: **£140,000** (working estimate — base £35k + expected bonuses up to ~£105k)
- TY 2026/27 ANI estimate: **£137,875**
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
| Amex Gold | ✅ Verified — 29 Apr–28 May 2026, 16 txns, £1,168.78 |
| Fidelity (SIPP + ISA quarterly statements) | ⚠️ Not yet built — manual snapshot input for now |

---

## Things to do next session (priority order)

1. **Amex categorisation pass** — now that real Amex data is in, the new merchants need rules + icons: SAINSBURY'S, TJX UK, DOJO*BOSWELLS COFFEE, DOJO*THE CRICKETERS INN, EUG04107 SOUTHAMPTON (recurring — appears 3×, likely worth identifying), FMS Sutton Scotney, AVOLTA LGW (airport retail), W H SMITH, EASYJET, BACK MARKET, AMZNMKTPLACE. Check these don't all dump into `other`/`spending`.
2. **Categorisation cleanup** (carried over) — `spending` bucket too generic (DEB fallback dumps GI GI FIRECRACKER, LC INTERNATIONAL, DAMIRA QUAYSIDE, RORY PACK, CSFLOAT INC); add merchant rules + icons or rename to `uncategorised`. Hastings Insurance subscription-icon mis-routing in Top Merchants.
3. **Wealth long-game chart** — net worth line by account type (SIPP / ISA / CS2 / Savings)
4. **Fidelity PDF parser** — once Rudi has 2–3 historic quarterly statements
5. **Search/filter transactions by description**
6. **Export transactions as CSV**

---

## Open questions for next session

- Amex `EUG04107 SOUTHAMPTON` appears 3 times (£51.93, £4.20, £49.99) — what is it? Recurring enough that it may want a merchant rule. (Likely a parking/EV charger ref — confirm with Rudi.)
- ANI tracker: still no manual "add SIPP contribution" entry; waiting on Fidelity parser vs. adding a stopgap manual entry.

---

## Rules agreed (see project instructions for full list)

1. One instruction at a time
2. No code explanations in chat
3. HTML changes go in the file
4. Standard workflow: download → replace local → GitHub Desktop → push → Cloudflare auto-deploys
5. Read `AMENDMENTS.md` and `README.md` at session start
6. Parser failures: read the console first
7. Big architectural/UX decisions: break the brevity rule and have a proper design discussion
