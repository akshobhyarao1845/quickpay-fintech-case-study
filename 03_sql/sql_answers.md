# SQL Answers

## Q1
### Query
Count transactions by status.
```sql
SELECT status, COUNT(*) AS transaction_count,
       ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS pct_of_total
FROM cleaned_transactions
GROUP BY status
ORDER BY transaction_count DESC;
```
### Result Summary
| status | transaction_count | pct_of_total |
|---|---|---|
| captured | 19 | 63.33% |
| failed | 7 | 23.33% |
| chargeback | 4 | 13.33% |

Out of 30 total transactions, 19 were successfully captured (63.3%), 7 failed (23.3%), and 4 resulted in chargebacks (13.3%). The chargeback rate of 13.3% is notably high and warrants merchant-level investigation.

---

## Q2
### Query
Calculate total captured GMV by merchant.
```sql
SELECT merchant_id, merchant_name, merchant_category, gateway_region,
       ROUND(SUM(amount_usd), 2) AS captured_gmv_usd
FROM cleaned_transactions
WHERE status = 'captured'
GROUP BY merchant_id, merchant_name, merchant_category, gateway_region
ORDER BY captured_gmv_usd DESC;
```
### Result Summary
| merchant_id | merchant_name | captured_gmv_usd |
|---|---|---|
| M002 | Beta Stores | $33,431.00 |
| M001 | Alpha Mart | $29,984.50 |
| M004 | Delta Travels | $10,300.00 |
| M003 | City Pharma | $8,640.00 |
| M005 | Eco Home | $0.00 |

Eco Home had $0 captured GMV — all its transactions were either failed or chargeback. Total captured GMV across all merchants: $82,355.50.

---

## Q3
### Query
Show top 10 merchants by captured GMV (limited to top 10).
```sql
SELECT merchant_id, merchant_name, merchant_category, gateway_region,
       COUNT(*) AS captured_txn_count,
       ROUND(SUM(amount_usd), 2) AS captured_gmv_usd
FROM cleaned_transactions
WHERE status = 'captured'
GROUP BY merchant_id, merchant_name, merchant_category, gateway_region
ORDER BY captured_gmv_usd DESC
LIMIT 10;
```
### Result Summary
Only 4 merchants had captured transactions. Beta Stores leads with $33,431 in captured GMV, followed by Alpha Mart ($29,984.50), Delta Travels ($10,300), and City Pharma ($8,640). APAC merchants (Beta Stores, Alpha Mart) dominate, contributing over 77% of total captured GMV.

---

## Q4
### Query
Show daily GMV and successful transaction count.
```sql
SELECT transaction_date,
       COUNT(*) AS total_transactions,
       SUM(CASE WHEN status = 'captured' THEN 1 ELSE 0 END) AS captured_count,
       ROUND(SUM(amount_usd), 2) AS total_gmv_usd,
       ROUND(SUM(CASE WHEN status = 'captured' THEN amount_usd ELSE 0 END), 2) AS captured_gmv_usd,
       ROUND(SUM(CASE WHEN status = 'captured' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS success_rate_pct
FROM cleaned_transactions
GROUP BY transaction_date
ORDER BY transaction_date;
```
### Result Summary
| date | total_txns | captured | total_gmv | captured_gmv | success_rate |
|---|---|---|---|---|---|
| 2026-03-01 | 5 | 5 | $26,382 | $26,382 | 100% |
| 2026-03-02 | 6 | 3 | $25,049 | $11,080 | 50% |
| 2026-03-03 | 5 | 4 | $18,391 | $16,032 | 80% |
| 2026-03-04 | 5 | 4 | $16,420 | $13,920 | 80% |
| 2026-03-05 | 6 | 1 | $19,232 | $6,136 | 16.7% |
| 2026-03-06 | 3 | 2 | $10,606 | $8,806 | 66.7% |

Mar 5 was the worst performing day with only 16.7% success rate. Mar 1 was perfect at 100%.

---

## Q5
### Query
Find merchants with chargeback ratio above 1%.
```sql
SELECT merchant_id, merchant_name,
       COUNT(*) AS total_transactions,
       SUM(CASE WHEN status = 'chargeback' THEN 1 ELSE 0 END) AS chargeback_count,
       ROUND(SUM(CASE WHEN status = 'chargeback' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS chargeback_ratio_pct
FROM cleaned_transactions
GROUP BY merchant_id, merchant_name
HAVING chargeback_ratio_pct > 1
ORDER BY chargeback_ratio_pct DESC;
```
### Result Summary
| merchant_name | total_txns | chargebacks | chargeback_ratio |
|---|---|---|---|
| Eco Home | 2 | 1 | 50.00% |
| Delta Travels | 4 | 1 | 25.00% |
| Beta Stores | 11 | 1 | 9.09% |
| Alpha Mart | 11 | 1 | 9.09% |

All 4 active merchants exceed the 1% threshold. Eco Home is critical at 50%. Even the APAC merchants (Beta Stores, Alpha Mart) are at 9.09%, well above acceptable levels. Immediate review recommended for Eco Home and Delta Travels.

---

## Q6
### Query
Find regions with average risk score above 50 and more than 20 transactions.
```sql
SELECT gateway_region,
       COUNT(*) AS transaction_count,
       ROUND(AVG(risk_score), 2) AS avg_risk_score
FROM cleaned_transactions
WHERE risk_score IS NOT NULL
GROUP BY gateway_region
HAVING AVG(risk_score) > 50 AND COUNT(*) > 20
ORDER BY avg_risk_score DESC;
```
### Result Summary
| gateway_region | transaction_count | avg_risk_score |
|---|---|---|
| APAC | 21 | 65.48 |

Only APAC meets both conditions (21 transactions, avg risk score 65.48). EU (4 txns) and US (4 txns) do not have more than 20 transactions. APAC's elevated average risk score warrants enhanced monitoring.

---

## Q7
### Query
Find users with 3 or more failed or chargeback transactions on the same day.
```sql
SELECT user_id, transaction_date, COUNT(*) AS failed_or_chargeback_count
FROM cleaned_transactions
WHERE status IN ('failed', 'chargeback')
GROUP BY user_id, transaction_date
HAVING COUNT(*) >= 3
ORDER BY failed_or_chargeback_count DESC, transaction_date;
```
### Result Summary
| user_id | transaction_date | failed_or_chargeback_count |
|---|---|---|
| U008 | 2026-03-05 | 4 |

User U008 (Ishaan Verma) triggered 4 failed/chargeback transactions in a single day (2026-03-05) — a strong indicator of potential fraud or payment method compromise. This user should be flagged for review. The user already has a `high` risk tier assigned in users.csv.

---

## Q8
### Query
Show chargeback count, unique affected users, and chargeback amount by merchant.
```sql
SELECT merchant_id, merchant_name, merchant_category,
       COUNT(*) AS chargeback_count,
       COUNT(DISTINCT user_id) AS unique_affected_users,
       ROUND(SUM(amount_usd), 2) AS chargeback_amount_usd,
       ROUND(SUM(amount_usd) / COUNT(*), 2) AS avg_chargeback_amount_usd
FROM cleaned_transactions
WHERE status = 'chargeback'
GROUP BY merchant_id, merchant_name, merchant_category
ORDER BY chargeback_count DESC;
```
### Result Summary
| merchant_name | chargebacks | unique_users | chargeback_amount | avg_amount |
|---|---|---|---|---|
| Alpha Mart | 1 | 1 | $5,400.00 | $5,400.00 |
| Beta Stores | 1 | 1 | $1,711.00 | $1,711.00 |
| Delta Travels | 1 | 1 | $2,500.00 | $2,500.00 |
| Eco Home | 1 | 1 | $6,649.00 | $6,649.00 |

All 4 merchants have exactly 1 chargeback each affecting 1 unique user. Total chargeback exposure is $16,260. Eco Home has the highest average chargeback value ($6,649), followed by Alpha Mart ($5,400). These amounts represent real financial risk requiring dispute resolution workflows.
