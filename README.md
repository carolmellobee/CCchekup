# Ledger

A credit card statement analyser that runs entirely in the browser. Drop in the PDF invoices your
bank sent you — password-protected ones included — and get spending analysis and a forecast. No
backend, no upload, no account.

Built by [Carol Mello](/about), data engineer.

---

## What it does

**Statements** — decrypts and parses PDFs with pdf.js. Detects the statement period from the due
date, reconciles what it parsed against the invoice total the statement declares, and includes a
parser view that highlights exactly which lines were read as transactions.

**Spending** — monthly totals, category breakdown, top merchants, category-over-time. Payments and
refunds are excluded from spend totals and listed separately.

**Forecast** — four models scored against your own history by walk-forward backtesting:

| Model | What it assumes |
| --- | --- |
| Linear trend (OLS) | Spending moves in a straight line |
| Holt exponential smoothing | Level plus trend, recent months weighted more (α, β grid-searched) |
| Robust recent median | No trend at all — median of the last three months |
| Trend + month-of-year | Seasonal multipliers per calendar month (needs 12+ months) |

Each model's MAE and MAPE are shown, and the ensemble weights them inversely to their error. Also on
that tab: remaining instalment commitments (contracted, not predicted), fixed vs. variable recurring
charges, and outlier detection by modified z-score.

**Transactions / Rules** — searchable table, editable categories that stick per merchant, editable
keyword rules, a custom regex escape hatch, and CSV/JSON export.

---

## Known issues

These are real and currently unfixed. Read this before trusting a number.

1. **Two-column statements parse incorrectly.** Itaú (and others) print two transaction columns per
   page. pdf.js reconstructs those as a single line — `21/11 BROOKSFIELD 06/10 320,01  31/03 CRIS
   CAFE 25,00` — and the parser reads it as one malformed transaction. Fix: cluster date-token
   x-coordinates per page to detect column boundaries, then build one line per (row, column).

2. **Future instalments are double-counted.** The `Compras parceladas - próximas faturas` section
   lists instalments due on *upcoming* invoices. The parser treats them as current transactions.
   Fix: track the section heading per column and exclude that section — then use it as the
   authoritative source for future commitments, which is better than the current inference.

3. **Issuer categories are ignored.** Itaú prints its own category under every line
   (`VESTUÁRIO .PORTO ALEGRE`). The parser ignores it and guesses from keywords, which misfires —
   `MERCADO*MERCADOLIVRE` is classified as Groceries because the description contains "mercado".
   Fix: read the continuation line, use the issuer category as the coarse bucket, and let keyword
   rules refine within it. Also make keyword matching word-boundary aware.

4. **Instalment lines are dated by purchase date, not billing month.** A `21/11 BROOKSFIELD 07/10`
   line on a June invoice is dated November. Across several statements the same purchase appears
   repeatedly, all dated November, distorting the monthly series. Fix: attribute transactions to the
   statement's billing month and keep the purchase date as metadata.

5. **Scanned PDFs cannot be read.** If the PDF has no text layer there is nothing to extract. The app
   says so rather than failing silently, but OCR is not implemented.

6. **No persistence.** Everything lives in memory. Export to JSON before closing the tab.

---

## Privacy

Statements are among the most revealing documents a person owns, so the design constraint is that
none of it leaves the browser.

- Files are read with `FileReader` and parsed in-page. There is no server component.
- PDF passwords are held in a DOM input and passed to pdf.js. They are never transmitted or stored.
- No analytics, no telemetry, no cookies, no logging.
- `default-src 'none'` CSP; the only permitted external origins are cdnjs (pdf.js, Chart.js) and
  Google Fonts.

The `.gitignore` blocks `*.pdf`, `*.xlsx`, `*.csv` and `ledger-export*.json` so real statements
can't be committed by accident. Keep it that way.

---

## Deploying

### 1. Push to GitHub

```bash
cd ledger
git init
git add .
git commit -m "Ledger: client-side credit card statement analyser"
git branch -M main
git remote add origin git@github.com:<your-username>/ledger.git
git push -u origin main
```

Before the first commit, confirm no statements are staged:

```bash
git status --porcelain   # nothing ending .pdf / .xlsx / .csv should appear
```

### 2. Connect Vercel

1. [vercel.com/new](https://vercel.com/new) → **Import Git Repository** → pick the repo.
2. Framework preset: **Other**. Build command: **none**. Output directory: leave blank (repo root).
3. **Deploy**.

It is a static site, so there is no build step and nothing to configure. `vercel.json` handles clean
URLs (`/about` rather than `/about.html`) and the security headers.

Or from the CLI:

```bash
npm i -g vercel
vercel        # preview
vercel --prod # production
```

### 3. Local development

No build tooling and no build step. All internal paths are relative, so opening `index.html` by
double-clicking it renders correctly.

Serving it is still more reliable, because over `file://` the pdf.js worker is cross-origin and gets
downgraded to a slower main-thread fallback:

```bash
python3 -m http.server 8000
# or
npx serve .
```

Either way the page needs internet on first load, since pdf.js and Chart.js come from cdnjs. Without
them the dashboard says so in a banner rather than failing silently. Vendoring both into `assets/`
removes that dependency — see *Hardening worth doing*.

---

## Structure

```
.
├── index.html          # dashboard — CSS, JS and favicon all inlined
├── about.html          # author page — same, self-contained
├── vercel.json         # clean URLs, CSP, security headers
├── LICENSE
└── README.md
```

**Why everything is inlined.** Each page is a single self-contained file with no local dependencies
at all. Split into `assets/styles.css` and `assets/app.js` it broke twice in practice — once from
root-absolute paths, once from the HTML being opened without its sibling folder. A statement
analyser that silently renders unstyled is worse than one with a duplicated stylesheet, so the trade
went to robustness. The cost is real: ~11KB of CSS is duplicated across the two pages, and there is
no shared cache between them. If you later add a third page, split the CSS back out and add a build
step.

Inside `index.html`, the JS is organised in numbered sections: utilities, PDF→text, period detection, transaction
extraction, file intake, aggregations, forecasting, rendering, rules/export, tabs.

No inline event handlers anywhere — all interaction is delegated from `#fileList` and `#txBody` — so
the CSP does not need `'unsafe-inline'` for scripts.

---

## Hardening worth doing

- **Pin the CDN scripts with SRI hashes**, or vendor pdf.js and Chart.js into the repo and drop
  cdnjs from the CSP entirely. That removes a third-party origin from a page handling financial
  documents and makes it work offline.
- **Tighten `script-src`.** Inlining the app forced `'unsafe-inline'` back into the script policy.
  Replacing it with a per-deploy nonce or a `sha256-` hash of the inline block restores the strict
  policy without giving up self-contained files.
- Add a `CHANGELOG.md` before the first shared link.

## Licence

MIT. See `LICENSE`.

## Credits

[pdf.js](https://mozilla.github.io/pdf.js/) (Apache-2.0) · [Chart.js](https://www.chartjs.org/)
(MIT) · [IBM Plex](https://github.com/IBM/plex) and [Archivo](https://fonts.google.com/specimen/Archivo)
(OFL).
