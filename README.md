# FinSight | Financial Performance and Transaction Intelligence Dashboard

**Financial transaction intelligence | KPI reporting | Customer segmentation | Year-over-Year analysis**

## Project Overview

Built an interactive **Power BI Financial Performance and Transaction Intelligence Dashboard** to explore financial transactions, transaction status, fees, taxes, customer behavior and regional performance. The project uses two linked CSV datasets to provide a high-level financial overview and a detailed transaction-level analysis, supporting evidence-based decisions through trends, segmentation, and drill-down reporting.

> **Data note:** Financial figures below describe the *dataset*, not realized revenue or business uplift. They are computed after deduplicating `transaction_id` and normalizing inconsistent `currency` labels. The dashboard's exact totals may differ if its Power Query transformations or status filters are different.

## Dataset at a Glance

| Metric | Verified result | Definition |
| --- | ---: | --- |
| Customer records | **5,000** | Unique `customer_id` values in `customers.csv` |
| Original transaction rows | **50,069** | Rows in `finance_transactions.csv` |
| Unique transactions | **50,000** | Unique `transaction_id` values |
| Customers with transactions | **3,983** | Distinct `customer_id` values in deduplicated transactions |
| Transaction amount | **₹45.54 crore** | ₹455,391,141.16 across **all** transaction statuses |
| Average transaction value | **₹9,107.82** | Total amount ÷ 50,000 unique transactions |
| Total fees | **₹7.26 lakh** | ₹725,914.91; blanks excluded from sum |
| Total taxes | **₹1.31 lakh** | ₹130,736.35 |
| Successful transactions | **42,869** | `transaction_status = Success` |
| Transaction success rate | **85.74%** | 42,869 / 50,000 |
| Date coverage | **01 Jan 2023 – 30 Apr 2026** | Dates recorded in the transactions file |
| Analysis dimensions | **5 customer segments, 10 transaction types, 13 states/UTs** | Based on source data |

**Important:** The **₹45.54 crore** figure includes transactions marked `Success`, `Failed`, and `Pending`; it is **not** the value of successfully settled transactions alone. The 2026 file covers only January–April, so full-year YoY comparisons against 2025 would be misleading without aligning the periods.

## Business Problem

Management needs a unified reporting view to answer questions such as:

- How are transaction amount and volume changing over time?
- What proportion of transactions succeed, fail, or remain pending?
- Which states/UTs and customer segments contribute most to transaction value?
- How do transaction types compare by amount, count, fees, and taxes?
- How does transaction activity vary by customer gender and occupation?
- Which transaction records drive changes in the KPIs?

## Data Sources and Model

### 1. `customers.csv` — Customer Dimension

**5,000 rows, 11 columns**

| Fields | Purpose |
| --- | --- |
| `customer_id` | Primary customer key |
| `fisrt_name`, `second_name` | Customer name fields (*`fisrt_name` is spelled this way in the source*) |
| `gender`, `date_of_birth` | Customer demographics |
| `city`, `state` | Geographic analysis |
| `occupation`, `customer_segment` | Segmentation and slicers |
| `annual_income`, `join_date` | Customer financial profile and onboarding date |

### 2. `finance_transactions.csv` — Transaction Fact

**50,069 original rows, 15 columns**

| Fields | Purpose |
| --- | --- |
| `transaction_id`, `reference_no` | Transaction/reference identifiers |
| `transaction_date` | Monthly trends and YoY analysis |
| `account_id`, `customer_id` | Account/customer association |
| `transaction_type` | Transaction category comparison |
| `channel`, `merchant_category` | Channel and merchant-category analysis |
| `amount`, `fee_amount`, `tax_amount` | Financial measures |
| `currency`, `transaction_status` | Currency and transaction outcomes |
| `is_fraud`, `risk_score` | Available fraud/risk fields for optional future analysis |

### Relationships

Use a **one-to-many** relationship:

```text
customers[customer_id]  (1) ──── (*)  finance_transactions[customer_id]
                                              |
                                              (*)
                                              |
                                        (1) Calendar[Date]
```

Create a proper **Calendar/Date** table to enable time-intelligence measures. Convert source dates (stored as `DD-MM-YYYY`) to Date type, using the appropriate locale before defining relationships and DAX.

## Data Preparation and Quality Checks

These are **recommended, reproducible preparation steps** for the supplied source data, not a claim that the dashboard already applies them:

1. **Deduplicate transactions by `transaction_id`:** 50,069 source rows contain 69 extra rows across 69 duplicate IDs. Of the 69 duplicated ID pairs, **68 pairs are exact duplicates**; **1 pair** differs in `fee_amount` (blank vs ₹21.19). For the metrics in this README, prefer the row with fewer missing values, then retain one row per ID.
2. **Handle missing fee values:** 24 source rows have blank `fee_amount`; **23 remain** after the above deduplication. Preserve as missing or use a documented business rule—do not silently imply blank fees were zero.
3. **Standardize currency:** `INR`, `inr`, and `inR` represent the same currency code in the supplied data. Normalize to `INR` before analysis; amounts are shown in ₹.
4. **Clean channel labels:** Trim stray whitespace and standardize `M@bile App` to `Mobile App` if confirmed as the intended label.
5. **Check negative amounts:** 8 negative `amount` values remain after deduplication. Preserve/flag them until their business treatment is established; don't assume they are data errors.
6. **Validate referential integrity:** All transaction `customer_id` values match entries in `customers.csv`; 3,983 of the 5,000 customers have at least one transaction.
7. **Protect time comparisons:** Compare equivalent periods when analyzing 2026 (Jan–Apr) against previous years.

## Dashboard Design

### Page 1 — Financial Performance Overview

**Primary KPI cards**

1. **Total Amount** — transaction value and YoY comparison.
2. **Total Transactions** — transaction count and change over time.
3. **Average Transaction Value** — average amount per unique transaction.
4. **Total Fees** — recorded transaction fees.
5. **Total Tax** — recorded transaction taxes.

**Six core analysis visuals**

| Visual | Business question |
| --- | --- |
| Monthly transaction amount (line/area chart) | When are transaction amounts rising or falling? |
| Amount by status (donut chart) | How does recorded value split between Success, Failed, and Pending? |
| Amount by customer segment (horizontal bar chart) | Which of the 5 segments contributes most? |
| Amount by state (horizontal bar chart) | Which states/UTs have the highest transaction value? |
| Transaction type matrix | How do 10 types compare by amount, fees, tax, and transaction count? |
| Amount by gender (donut chart) | How does transaction value vary by gender? |

**Interactive slicers (per project requirements):** Year, Dynamic Measure, Occupation, and Category. Map **Category** to the intended field (`merchant_category` or `transaction_type`) explicitly in the report. A Power BI field parameter can support the Dynamic Measure selector.

### Page 2 — Detailed Transaction View

A granular transaction table and drill-down/drill-through analysis to inspect the records behind KPI trends, status comparisons, customer segments, and regional summaries. Include transaction identifiers, dates, status, type, customer, amount, fees, and tax as appropriate.

## Example Power BI DAX Measures

> **Reference formulas:** The expressions below are suggested implementation examples. They are not verified against the `.pbix` file. Replace table names if your Power BI model uses different names. `finance_transactions` should refer to the **deduplicated** fact table; `Calendar` should have an active relationship to its transaction date.

```dax
Total Amount =
SUM ( finance_transactions[amount] )

Total Transactions =
DISTINCTCOUNT ( finance_transactions[transaction_id] )

Average Transaction Value =
DIVIDE ( [Total Amount], [Total Transactions] )

Total Fees =
SUM ( finance_transactions[fee_amount] )

Total Tax =
SUM ( finance_transactions[tax_amount] )

Successful Transactions =
CALCULATE (
    [Total Transactions],
    finance_transactions[transaction_status] = "Success"
)

Transaction Success Rate =
DIVIDE ( [Successful Transactions], [Total Transactions] )

Previous Year Amount =
CALCULATE (
    [Total Amount],
    DATEADD ( 'Calendar'[Date], -1, YEAR )
)

Amount YoY % =
DIVIDE ( [Total Amount] - [Previous Year Amount], [Previous Year Amount] )
```

Format the success rate and YoY measures as **percentages**. YoY comparisons depend on the active date filter context; use matched Jan–Apr date ranges for 2026 vs 2025 comparisons.

## Business Relevance

- **Financial visibility:** One reporting view for transaction value, counts, fees, and taxes.
- **Operational monitoring:** See the split of successful, failed, and pending transactions.
- **Customer intelligence:** Compare contribution across 5 segments, demographics, and occupations.
- **Regional insights:** Identify high-transaction-value states/UTs across 13 locations.
- **Decision support:** Explore 10 transaction types and trace aggregate results to individual records.

**Interpretation guardrail:** Transaction value is **not equivalent to company revenue**, and fees/taxes do **not** establish net profit. Profitability analysis requires actual revenue recognition and cost information, which the provided files do not contain.

## Tools and Techniques

**Microsoft Power BI** · **Power Query (recommended for cleaning)** · **DAX (reference measures above)** · **Data Modeling** · **Business Intelligence** · **Financial Analytics** · **KPI Reporting** · **Year-over-Year Analysis** · **Customer Segmentation** · **Interactive Data Visualization** · **Drill-Down Analysis**

## Run / Reproduce

1. Download or clone the repository and open the Power BI `.pbix` file **if included**.
2. Point its two source queries to `customers.csv` and `finance_transactions.csv`.
3. Apply/verify the cleaning rules described above, especially transaction-ID deduplication and data types.
4. Build or confirm the `customer_id` relationship and Calendar table.
5. Refresh in Power BI Desktop and review the KPIs, visuals, and transaction-level details.

### Suggested Repository Structure

```text
finance-analytics-powerbi/
├── README.md
├── data/
│   ├── customers.csv
│   └── finance_transactions.csv
├── Finance_Analytics_Dashboard.pbix     # add your actual report
└── assets/
    ├── overview.png                     # add your dashboard screenshot
    └── transaction-details.png          # add your dashboard screenshot
```

*The `.pbix` file and screenshots are suggested repository additions; they were not provided for verification.*

---

**Portfolio focus:** Power BI · Finance Analytics · Dashboard Development · Data Modeling · DAX · KPI Reporting
