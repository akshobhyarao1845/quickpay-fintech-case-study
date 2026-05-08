# Spreadsheet Answers

## Cleaning Steps

1. **Loaded** `transactions_raw.csv` into the Raw Transactions sheet.
2. **Stripped whitespace** from all string fields (merchant_name, status, gateway_region, risk_score) — many values had leading/trailing spaces.
3. **Collapsed internal double spaces** in merchant names (e.g. `Alpha  Mart` → `Alpha Mart`, `Beta  Stores` → `Beta Stores`).
4. **Standardized merchant name casing** using title-case normalization before mapping to canonical names.
5. **Standardized date format** to `YYYY-MM-DD` (ISO 8601) across all 30 rows.
6. **Standardized status values** — removed noise codes like `E05 TIMEOUT` and excess whitespace.
7. **Extracted numeric risk scores** from three different formats: plain numeric (`55`), `score:62`, and `risk-83`.
8. **Standardized gateway_region** to uppercase (APAC, EU, US). One row had no value which was filled from merchant_master.
9. **Converted raw_amount to amount_usd** using a date-matched lookup from `exchange_rates.csv`.
10. **Enriched** with merchant_id, merchant_category, and default_region via VLOOKUP from `merchant_master.csv`.
11. **Applied flag columns** using rule-based formulas.

---

## Standardization Rules

### Merchant Names
Normalized to title case, collapsed multiple internal spaces, trimmed edges. Mapped to canonical merchant master names:
- All variations of `alpha mart`, `ALPHA MART`, `Alpha  Mart` → **Alpha Mart**
- All variations of `BETA STORES`, `beta stores`, `Beta  Stores` → **Beta Stores**
- `DELTA TRAVELS`, `delta travels` → **Delta Travels**
- `City Pharma`, `Eco Home` — already clean

### Date Format
All dates converted to `YYYY-MM-DD` (ISO 8601). Source dates were already in this format but cast to ensure consistency.

### Status Values
| Raw | Standardized |
|---|---|
| `captured`, `Captured`, ` CAPTURED`, `Captured ` | `captured` |
| `failed e05 timeout`, `Failed E05 Timeout`, `FAILED e05 TIMEOUT` | `failed` |
| `chargeback`, ` chargeback ` | `chargeback` |

### Risk Score
| Raw format | Extraction |
|---|---|
| `score:62` | Remove `score:` prefix → `62` |
| `risk-83` | Remove `risk-` prefix → `83` |
| `55 `, `75 ` | Strip whitespace → `55`, `75` |
| (blank) | Set to null/empty |

### Gateway Region
Stripped whitespace and converted to uppercase: `apac` → `APAC`, ` EU ` → `EU`, `us` → `US`. Missing values filled from `default_region` in merchant_master.

---

## Lookup and Enrichment Logic

### Currency Conversion (amount_usd)
- Joined `exchange_rates.csv` on `(transaction_date, currency)` to retrieve the correct daily rate.
- `amount_usd = raw_amount × usd_rate` (rounded to 2 decimal places).
- Example: T001 had 420,000 INR on 2026-03-01; rate = 0.0119 → amount_usd = **$4,998.00**

### Merchant Enrichment
- VLOOKUP from `merchant_master.csv` on `merchant_name` (cleaned) to retrieve `merchant_id`, `merchant_category`, and `default_region`.
- `gateway_region` was filled from `default_region` where the raw value was missing.

### Flag Logic
**high_value_flag = 1 when:**
- gateway_region = APAC AND amount_usd > 5,000
- gateway_region = EU AND amount_usd > 6,000
- gateway_region = US AND amount_usd > 7,000
- Otherwise 0

**high_risk_flag = 1 when:**
- risk_score ≥ 70 (any region)
- OR status = `chargeback`
- Otherwise 0

---

## Final Answers

| Metric | Value |
|---|---|
| Total raw rows | 30 |
| Total cleaned rows | 30 |
| Invalid or missing rows handled | 1 row had null risk_score; 10 rows had missing gateway_region (filled from merchant default_region) |
| Top region by GMV | **APAC** — $82,594.00 |
| Number of high value transactions | **7** |
| Number of high risk transactions | **9** |
| Top merchant by captured GMV | **Beta Stores** — $33,431.00 |

---

## Formula Samples

### amount_usd (currency conversion via INDEX/MATCH)
```excel
=ROUND(D2 * INDEX(exchange_rates!$C:$C,
       MATCH(1,(exchange_rates!$A:$A=B2)*(exchange_rates!$B:$B=E2),0)), 2)
```
*(Array formula — press Ctrl+Shift+Enter)*

### Merchant ID enrichment (VLOOKUP)
```excel
=VLOOKUP(C2, merchant_master!$B:$E, 1, FALSE)
```

### high_value_flag
```excel
=IF(OR(AND(K2="APAC", H2>5000),
       AND(K2="EU",   H2>6000),
       AND(K2="US",   H2>7000)), 1, 0)
```

### high_risk_flag
```excel
=IF(OR(G2>=70, ISNUMBER(SEARCH("chargeback", F2))), 1, 0)
```

### Merchant Risk Summary (chargeback ratio)
```excel
=COUNTIFS(cleaned!$C:$C, A2, cleaned!$F:$F, "chargeback") /
 COUNTIF(cleaned!$C:$C, A2)
```
