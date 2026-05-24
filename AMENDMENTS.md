# Minorph — Project Amendments (Session 3)

Update the project files before starting a new chat.

---

## Files to replace

### 1. `index.html` → replace entirely
Download from the latest Claude output. Key changes this session:
- Supabase URL and anon key are now populated (no longer placeholder)
- `extractPdfText` now uses `item.hasEOL` for proper line breaks
- `parseLloyds` completely rewritten — strips column label noise, flattens to one string, extracts amounts from the portion after the type code using a secondary regex scan rather than optional groups
- `parseAmex` literal newline bug fixed
- All `split('\n')` literal-newline-in-string bugs fixed

### 2. `README.md` → replace entirely
Updated to reflect live infrastructure, actual Supabase URL, deployment workflow, clear-data SQL, and parser notes.

---

## Current live state

- App is live at `minorph.pages.dev`
- Supabase project: `minorph` (separate from Infinity Renewables)
- User: `cookihd101@gmail.com`, UUID: `5cebd01a-5b81-4bbd-9d2e-ca7f79ce020b`
- 6 accounts seeded, 2 goals seeded
- Cloudflare Pages connected to `Rudizzle123/Minorph` on GitHub — auto-deploys on push to `main`

---

## Parser status after this session

| Parser | Status |
|---|---|
| Lloyds Current | Fixed and tested — 70 transactions parsed from April 2026 statement. Amounts now correct. |
| Lloyds Credit | Untested — same format, should work |
| Revolut Current | Written, untested |
| Revolut Savings | Written, untested |
| Amex Gold | Speculative — first statement arrives 28 May 2026 |

---

## Rules agreed this session (for next Claude to follow)

1. **One instruction at a time** — never give multiple options or steps at once. Give the single highest-confidence fix and wait for the result before the next step.
2. **No code explanations** — Rudi doesn't need to know why code works, only what to do. Keep technical detail out of responses unless directly asked.
3. **HTML updates** — when anything needs changing in index.html, just do it and output the file. Don't paste code into chat.
4. **Workflow** — Rudi downloads the file, replaces local copy, commits and pushes in GitHub Desktop, Cloudflare auto-deploys. Don't suggest alternative workflows.

---

## Things to do next session

- [ ] Verify April statement upload shows correct amounts (was £0.00 before final fix)
- [ ] Upload remaining Lloyds Main Current months (Nov 2025 – Mar 2026)
- [ ] Upload Lloyds Credit statements
- [ ] Upload Revolut Current and Savings statements
- [ ] Test Amex Gold parser on 28 May 2026 statement
- [ ] Update Amex goal `current_amount` once statement is processed
- [ ] Update README and AMENDMENTS in project files
