# Data schema (v1)

Curated, source-backed data for IDX-listed companies. Plain JSON, one file per issuer.
Every commit is validated before it is published (see "Validation" below), and the
[seoultra.id](https://seoultra.id/stocks/idx/) pages never contain a typed number: every
figure on them comes from these files or is computed from them.

## Golden rules
1. **Never estimate, interpolate, fill or invent.** If a value cannot be read from a
   source document, store `null` (the page shows "n/a").
2. **Primary sources first.** `filing` = annual report / audited or interim financial
   statements filed with IDX/OJK. `company_release` = the issuer's own press release,
   analyst presentation, IR page or IDX disclosure (keterbukaan informasi). `news` is a
   cross-check only and must not be the sole source of a number unless nothing else exists.
3. **Consolidated vs bank-only are never mixed.** Every datum that could be either carries
   `basis`. Profit, EPS, equity, assets, loans and deposits are **consolidated**.
   Ratios are stored exactly as the company reports them, with whatever basis it used.
4. **Store per-share numbers exactly as printed** (raw) and record their `share_basis`.
   Split adjustment happens in code from `corporate_actions.json` — never by hand.

## Units
| Kind | Unit | Example |
|---|---|---|
| Money (profit, equity, assets, loans, deposits, dividend totals) | IDR **billion** (Rp miliar), number | `54836.4` |
| Per-share (EPS, DPS, prices) | Rupiah per share, as printed | `530` |
| Share counts | whole shares | `123275050000` |
| Ratios, growth, payout | percent as a number | `23.9` means 23.9% |
| Dates | ISO `YYYY-MM-DD` | `2025-12-31` |

Convert from Rp juta (millions) by dividing by 1,000. Keep the precision the source prints.

## Datum (every number is one of these)
```json
{
  "v": 54836.4,
  "source_url": "https://www.bca.co.id/-/media/.../AR-2025.pdf",
  "source_type": "filing",
  "as_of_date": "2025-12-31",
  "loc": "p. 24, Ikhtisar Keuangan",
  "basis": "consolidated",
  "share_basis": "2026-03-10",
  "note": "optional free text, e.g. definition used"
}
```
- `v` — number or `null`. When `null`, the other keys may be omitted except `note`
  (say why: "not disclosed", "not found in AR2021/AR2022").
- `source_url` — the exact document (PDF/page), not a homepage. https only.
- `source_type` — `filing` | `company_release` | `news` for issuer data; `official` (central bank /
  government publication, e.g. BI-Rate) and `market_data` (reserved for a licensed price feed; unused).
- `as_of_date` — the date the figure is AS OF (period end for balance sheet / FY figures,
  the price date for prices, the announcement date for dividends).
- `loc` — page number + table/section name inside the document, so a human can re-check.
- `basis` — `consolidated` | `bank_only` | `n/a`. Required on ratios and on money figures.
- `share_basis` — **required on per-share values and share counts** (eps, dps, prices,
  shares_out). A date on which the company's share count was on the same basis as the
  printed figure. Normally the source document's publication date. If a document prints a
  historical figure *as originally paid* (not restated for a split that happened before
  the document was published), use the figure's original announcement date instead.
  The adjuster applies every split whose `ex_date` is AFTER `share_basis`.

## Files
| File | What it holds |
|---|---|
| `index.json` | Entry point: coverage per issuer (periods, file path), units, file list. Generated from the data. |
| `sectors.json` | Sector registry: id, names (`id`/`en`), tickers, and optional display `groups` (id, names, tickers in display order; every ticker of the sector in exactly one group). Banking groups: `bumn`, `private_large`, `sharia`, `digital`, `bpd`, `private_mid`. |
| `issuers/<CODE>.json` | One issuer: annual data, dividends, latest interim period. |
| `corporate_actions.json` | Every issuer's splits, reverse splits, rights and bonus issues in one table. |
| `macro.json` | `bi_rate` and `sbn10y` datums. |
| `prices/<CODE>.json` | Daily open/high/low/close/volume, one row per trading session, as traded. Updated every trading day. |
| `validation_report.json` | Result of the validator for every issuer (errors, cross-check mismatches, coverage). |

## Issuer file: `issuers/<CODE>.json`
```json
{
  "schema": 1,
  "code": "BBCA",
  "name": "PT Bank Central Asia Tbk",
  "name_short": "Bank Central Asia",
  "sector": "banking",
  "exchange": "IDX",
  "sharia": false,
  "website": "https://www.bca.co.id",
  "ir_url": "https://www.bca.co.id/id/tentang-bca/hubungan-investor",
  "kbmi": { "v": 4, "source_url": "...", "source_type": "filing", "as_of_date": "2025-12-31", "loc": "..." },
  "label_overrides": {},
  "fy": {
    "2019": { ...balance-sheet levels only (see below)... },
    "2020": { ...all FY fields... },
    "...": {},
    "2025": {}
  },
  "dividends": [ ...dividend records... ],
  "interim": { ...latest interim period... },
  "validation_ack": [ { "check": "eps_shares:2021", "reason": "why the >3% gap is real, with the source" } ],
  "research_notes": [ "free text: anything a reviewer must know (restatements, definition changes, doubts)" ]
}
```
`label_overrides` — for sharia banks map generic keys to the label the bank uses, e.g.
`{"npl_gross": "NPF gross", "ldr": "FDR", "nim": "NIM/NOM"}`.

### `fy.<YEAR>` fields (all are datums)
FY2019 needs only: `total_assets`, `loans_gross`, `deposits`, `equity_attr`, `total_equity`
(for averages and growth). FY2020–FY2025 need everything below.

| Key | Meaning | Unit / basis |
|---|---|---|
| `net_profit_attr` | Net profit attributable to owners of the parent | IDR bn, consolidated |
| `eps` | Basic earnings per share | Rp, consolidated, + `share_basis` |
| `payout_ratio` | Dividend payout ratio for THIS fiscal year's profit, as reported | % |
| `total_assets` | Total assets | IDR bn, consolidated |
| `loans_gross` | Gross loans (incl. sharia financing if reported in the group total) | IDR bn, consolidated |
| `deposits` | Third-party funds (DPK) | IDR bn, consolidated |
| `equity_attr` | Equity attributable to owners of the parent | IDR bn, consolidated |
| `total_equity` | Total equity incl. non-controlling interest | IDR bn, consolidated |
| `shares_out` | Shares outstanding at FY end, net of treasury shares | count, + `share_basis` |
| `year_end_price` | Closing price on the last trading day of the FY | Rp, + `share_basis` |
| `roe` | Return on equity as reported (note the definition) | %, basis as reported |
| `roa` | Return on assets as reported | %, basis as reported |
| `nim` | Net interest margin | %, basis as reported |
| `coc` | Cost of credit | %, basis as reported |
| `npl_gross` | Gross NPL (NPF for sharia) | %, basis as reported |
| `npl_net` | Net NPL (optional) | % |
| `lar` | Loan at risk | % |
| `npl_coverage` | NPL coverage (loan-loss allowance / NPL) | % |
| `ldr` | Loan-to-deposit ratio (FDR for sharia) | % |
| `car` | Capital adequacy ratio | %, basis as reported |
| `cir` | Cost-to-income ratio (NOT BOPO — store BOPO in `bopo`) | % |
| `bopo` | Operating expenses / operating income (optional) | % |
| `casa` | CASA ratio (current + savings / DPK) | % |

Loan growth, deposit growth, BVPS, computed ROE, CAGR etc. are NOT stored — they are
calculated in code from the levels above.

### Dividend record (one per payment: interim, final, special)
Values in the examples below are illustrative only — never copy them as data.
```json
{
  "fy": 2020,
  "type": "final",
  "dps": 432,
  "total": 10650.6,
  "cum_date": "2021-04-01",
  "ex_date": "2021-04-05",
  "record_date": "2021-04-06",
  "pay_date": "2021-04-29",
  "share_basis": "2021-03-18",
  "source_url": "...",
  "source_type": "company_release",
  "as_of_date": "2021-03-18",
  "loc": "...",
  "note": ""
}
```
- `fy` = the fiscal year whose profit the dividend is paid from (NOT the payment year).
  An interim dividend paid in Dec 2025 out of 2025 profit is `fy: 2025`; the final paid in
  Apr 2026 out of 2025 profit is also `fy: 2025`.
- `dps` in Rp as announced (raw); `total` = total cash dividend in IDR bn (null if not stated).
- `ex_date` = the regular-market ex-date. Unknown dates → `null`.

### `interim` (latest reported interim period)
```json
{
  "period": "9M",
  "year": 2026,
  "period_end": "2026-09-30",
  "fields": {
    "profit_ytd": {datum},            "profit_ytd_prior": {datum},
    "equity_attr": {datum},           "shares_out": {datum},
    "total_assets": {datum},
    "loans_gross": {datum},           "loans_gross_prior": {datum},
    "deposits": {datum},              "deposits_prior": {datum},
    "roe": {datum}, "roa": {datum}, "nim": {datum}, "coc": {datum}, "npl_gross": {datum},
    "lar": {datum}, "npl_coverage": {datum}, "ldr": {datum}, "car": {datum}, "cir": {datum},
    "casa": {datum}
  }
}
```
`period` ∈ `Q1` | `H1` | `9M`. `*_prior` = same period one year earlier (for YoY).

### Corporate action record
```json
{
  "code": "BBCA",
  "type": "split",
  "ratio_old": 1,
  "ratio_new": 5,
  "ex_date": "2021-10-13",
  "source_url": "...",
  "source_type": "company_release",
  "as_of_date": "2021-09-23",
  "loc": "...",
  "note": "1 old share → 5 new shares; nominal Rp62.5 → Rp12.5"
}
```
`type` ∈ `split` | `reverse_split` | `rights` | `bonus`. Only `split`/`reverse_split` drive
the per-share adjustment factor in v1 (factor = ratio_new / ratio_old). `rights` issues are
recorded for the corporate-action calendar and flagged on the page.

Corporate actions live only in `corporate_actions.json` — an issuer file must not carry
its own copy (the validator rejects it), so there is one table and no drift.

## Macro
- `macro.json` — `sbn10y` and `bi_rate` datums (value, date, source). `sbn10y` is `null`
  until a source with redistribution permission is in place.

## Daily prices: `prices/<CODE>.json`
Not real-time. One row per IDX trading session since 2019-01-02. Two updates per trading
day (Monday to Friday, WIB): around 09:20 the session's opening price is published as
`today`; around 17:30 the full row is appended to `rows` and `today` is removed.

```json
{
  "schema": 1,
  "code": "BBCA",
  "exchange": "IDX",
  "currency": "IDR",
  "adjusted": false,
  "splits": [{ "ex_date": "2021-10-13", "factor": 5 }],
  "source": { "name": "Yahoo Finance", "symbol": "BBCA.JK", "url": "https://finance.yahoo.com/quote/BBCA.JK/history/", "source_type": "market_data", "note": "…" },
  "first_date": "2019-01-02",
  "last_date": "2026-11-02",
  "sessions": 1889,
  "today": { "date": "2026-11-03", "open": 5025, "status": "provisional", "note": "…" },
  "columns": ["date", "open", "high", "low", "close", "volume"],
  "rows": [
    ["2021-10-12", 36275, 36600, 36225, 36600, 18213400],
    ["2021-10-13", 7400, 8250, 7400, 7525, 210893300]
  ]
}
```

- **Prices are stored as traded**, like every other per-share value here. The rows above
  are the last session before BCA's 1:5 split and the first one after it. For a continuous
  series, divide open/high/low/close by `factor` (and multiply volume by it) for every
  split in `splits` whose `ex_date` is after the row's date: 36,600 ÷ 5 = 7,320.
  `splits` always equals the splits for that code in `corporate_actions.json`.
- Dividends are not adjusted for. Rights issues do not change the factor (rule 4 above).
- `volume` is in shares (not lots) for the regular-market session.
- `today` exists only while a session is in progress: the opening price, provisional, as
  traded. It is never part of `rows`; use `rows` for anything historical. If the evening
  update has not run yet, `today` can still be there after the close.
- In `index.json`, each issuer's `prices.today` mirrors it (`null` outside trading hours).
- **Source handling.** Yahoo Finance publishes its history already scaled for splits and,
  for some issuers, rights issues. The fetcher reverses that scaling. A rights reversal is
  listed in `source.reversed` with Yahoo's factor and the annual-report year-end price it
  was checked against (BBRI's 2021 rights issue: Yahoo factor 0.909075).
- A session Yahoo has no trades for is left out, never filled in. Rows already published
  do not change, except the last 10 sessions, which the source may still correct.

## Validation
A commit only reaches the website after it passes the validator:
- **Errors** (block publishing): a number without an https `source_url`, a bad
  `source_type`, date, unit or `basis`, a missing required field, a ratio outside its
  plausible range (usually a fraction-vs-percent slip), an unknown sector or ticker code.
- **Cross-checks** (must agree within 3%): net profit × payout ≈ total dividend;
  DPS × shares ≈ total dividend; EPS × shares ≈ net profit (catches a wrong
  `share_basis` around a split).
- **Daily prices.** Errors: dates not increasing or on a weekend, low > open/close or
  high < open/close, a price that is not a multiple of the IDX tick size (usually a missed
  split), a close that moves more than 35% in one session on the split-adjusted series,
  `splits` that differ from `corporate_actions.json`, a `today` that is not after the last
  row, off the tick grid or more than 35% from the last close. Cross-check: the last close of every
  year must equal the `year_end_price` printed in the annual report (within 0.5%).
- A gap with a real cause is never "fixed" by editing a number. It is recorded in the
  issuer's `validation_ack` with the reason, e.g. BBRI FY2021, where the rights issue
  makes the reported EPS use weighted-average shares. It then shows as `explained` in
  `validation_report.json`.
