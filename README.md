# edgar-form4-fetch

A small utility that parses **open-market insider purchases** (SEC Form 4, Transaction
Code "P") from EDGAR and caches them per quarter as parquet.

It exists to run the parse in the cloud: SEC's fair-access limit is ~10 requests/sec **per
IP**, so a full 2003-present parse (~2-3M filings) takes days on one machine. A GitHub
Actions matrix runs one quarter per runner — each a distinct IP under the limit — so the
whole history builds in a few hours on free public-repo runners.

## What it does

- Enumerates Form 4 filings from the EDGAR quarterly full-index.
- Fetches each full-submission `.txt` (which embeds the `ownershipDocument` XML) and keeps
  Code "P" non-derivative purchases, keyed on the **filing date** (point-in-time).
- Bounds the universe to listed issuers via SEC's `company_tickers.json` (resolves
  issuer → ticker). Excludes pure 10% holders who are not officers/directors.
- Respects SEC fair access: descriptive `User-Agent` + throttled to under 10 req/s.

## Usage

Local:

```bash
pip install -r requirements.txt
python build_data.py --list-quarters        # JSON list of quarter labels
python build_data.py --quarter 2015Q3       # build one quarter -> data/edgar/form4/2015Q3.parquet
python build_data.py                         # build everything (sequential; slow)
```

Cloud (this repo): set a repo **variable** `SEC_USER_AGENT` to a descriptive string with a
contact email (SEC requires it), then run the **build-form4** workflow from the Actions tab
(or `gh workflow run build-form4.yml`). When it finishes, download the `form4_cache`
artifact — a `form4_cache.tar.gz` of all per-quarter parquet files.

## Output schema

One row per Code-"P" purchase: `issuer_cik, issuer_ticker, issuer_name, owner_cik,
owner_name, txn_date, filing_date, shares, price, is_officer, is_director, is_ten_pct`.

Data is public SEC EDGAR; this tool only parses it.
