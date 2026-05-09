# QuickPay FinTech Operations Case Study

## Student Information

| Field | Details |
|---|---|
| Student Name | Akshobhya Rao A P |
| Public GitHub Repository | [https://github.com/yourusername/quickpay-fintech] |

---

## Project Overview

This project solves a 5-part data analytics assignment for QuickPay, a fintech company processing digital payments. It covers data cleaning, SQL analysis, Python reconciliation, JSON normalization, and a Looker Studio business monitoring dashboard.

---

## Tools Used

| Part | Tool |
|---|---|
| Part 1 — Spreadsheet Cleaning | Python (pandas, openpyxl) |
| Part 2 — SQL Analysis | SQL (SQLite-compatible syntax) |
| Part 3 & 4 — Python Pipeline | Python 3, pandas, numpy, json |
| Part 5 — Dashboard | Looker Studio |

---

## Repository Structure

```
.
├── README.md
├── 01_data/
│   ├── raw/                         ← Input files (do not modify)
│   │   ├── transactions_raw.csv
│   │   ├── merchant_master.csv
│   │   ├── users.csv
│   │   ├── ledger.csv
│   │   ├── gateway.csv
│   │   ├── exchange_rates.csv
│   │   └── api_response_sample.json
│   └── processed/                   ← All generated output files
│       ├── cleaned_transactions.csv
│       ├── merchant_risk_summary.csv
│       ├── missing_in_gateway.csv
│       ├── missing_in_ledger.csv
│       ├── amount_mismatches.csv
│       ├── status_mismatches.csv
│       ├── reconciliation_report.csv
│       ├── api_normalized.csv
│       ├── daily_summary.csv
│       ├── payment_method_breakdown.csv
│       ├── region_breakdown.csv
│       └── merchant_performance_summary.csv
├── 02_spreadsheet/
│   ├── spreadsheet_workbook.xlsx    ← Excel workbook (6 sheets)
│   └── spreadsheet_answers.md
├── 03_sql/
│   ├── analysis_queries.sql         ← All 8 SQL queries (Q1–Q8)
│   └── sql_answers.md
├── 04_python/
│   ├── fintech_pipeline.ipynb       ← Full reconciliation + JSON normalization notebook
│   └── summary_metrics.json
└── 05_visualization/
    └── dashboard_link.txt           ← Public Looker Studio dashboard link
```

---

## Run Instructions

### Prerequisites
```bash
pip install pandas numpy openpyxl nbformat
```

### Part 1 — Generate spreadsheet and cleaned CSVs
The spreadsheet workbook and processed CSVs are pre-generated and included in the repository. To regenerate:
```bash
python3 scripts/generate_cleaned.py   # if provided, otherwise open the xlsx directly
```

### Part 2 — SQL Queries
Load `01_data/processed/cleaned_transactions.csv` into any SQL environment (SQLite, DuckDB, PostgreSQL) as table `cleaned_transactions`, then run:
```bash
sqlite3 quickpay.db < 03_sql/analysis_queries.sql
```
Or with DuckDB:
```python
import duckdb
conn = duckdb.connect()
conn.execute("CREATE TABLE cleaned_transactions AS SELECT * FROM read_csv_auto('01_data/processed/cleaned_transactions.csv')")
conn.execute(open('03_sql/analysis_queries.sql').read())
```

### Part 3 & 4 — Python Notebook
```bash
cd 04_python
jupyter notebook fintech_pipeline.ipynb
```
Run all cells in order. All output files are written to `01_data/processed/`.

### Part 5 — Dashboard
Open the link in `05_visualization/dashboard_link.txt` in any browser.

---

## Key Findings

### Data Cleaning (Part 1)
- 30 raw transactions cleaned to standardized format
- 10 unique status variants normalized to 3 canonical values: `captured`, `failed`, `chargeback`
- 13 merchant name variants normalized to 5 canonical merchants
- 3 risk score formats unified to plain numeric
- **7 high-value transactions** and **9 high-risk transactions** flagged
- Top region by GMV: **APAC** ($82,594)
- Top merchant by captured GMV: **Beta Stores** ($33,431)

### SQL Analysis (Part 2)
- 63.3% success (captured), 23.3% failed, 13.3% chargeback
- All 4 merchants with chargebacks exceed the 1% threshold
- **APAC** is the only region with avg risk score > 50 AND > 20 transactions (avg risk: 65.48)
- **U008 (Ishaan Verma)** flagged for 4 failed/chargeback transactions in a single day — potential fraud

### Reconciliation (Part 3)
- 6 total issues found across ledger and gateway
- 2 records missing in gateway (R004: $2,100, R010: $2,500)
- 1 record missing in ledger (R011: $1,800)
- 2 amount mismatches (R002: $50 diff, R008: $40 diff)
- 1 status mismatch (R005: ledger=success, gateway=failed)
- **Amount at risk: $4,690.00**

### JSON Normalization (Part 4)
- Flattened 2 batches × 3 settlements = 6 rows
- Covers 2 merchants (Alpha Mart APAC, Delta Travels US)
- 4 settled, 1 pending, 1 failed; total $12,940.50
