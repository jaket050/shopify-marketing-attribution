# Shopify Customer Segmentation & Marketing Attribution Analysis

**An end-to-end e-commerce analytics project connecting marketing channel performance, customer segmentation, and RFM analysis — built in Google BigQuery and Tableau to help growth teams optimize spend, prioritize high-value channels, and strengthen long-term customer retention.**

---

## Dashboard Preview

![Final Dashboard](visuals/Final%20Dashboard.png)

> **Title:** Understanding Which Marketing Channels Drive Revenue and Customer Value
>
> **KPI cards:** Orders YTD (+22.7% vs PY), Sales YTD $18,887 (+29.9% vs PY), Repeat Customer Revenue Share (+21.9%)
>
> **Visuals:** Channel Performance bar chart (Top N, rankable by Orders/Revenue/AOV), Order Volume vs Customer Intent quadrant (Scale / Nurture / Optimize / Deprioritize), Repeat Customer Revenue as Share of Total Revenue by channel

---

## Project Overview

E-commerce teams often struggle to connect where customers come from, how much they spend, and whether they come back. This project builds that unified view using Shopify transaction data, marketing attribution, and customer behavior signals — answering not just which channels drive volume, but which ones drive *value*.

> **Business Question:** How can we connect marketing attribution, channel performance, and customer behavior to create a unified view of growth — enabling teams to optimize marketing spend, prioritize high-value channels, and strengthen long-term customer retention?

---

## Stakeholders

| Role | Need |
|---|---|
| E-commerce Manager | Clear performance metrics by channel to guide budget allocation |
| Marketing Lead | First-touch and last-touch attribution to optimize campaigns |
| Merchandising Team | Customer segment profiles to tailor site content and promotions |

---

## Tools & Stack

![BigQuery](https://img.shields.io/badge/GCP-BigQuery-4285F4?logo=google-cloud&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-Advanced-336791?logo=postgresql&logoColor=white)
![Tableau](https://img.shields.io/badge/Tableau-Dashboard-E97627?logo=tableau&logoColor=white)
![Shopify](https://img.shields.io/badge/Data-Shopify%20Export-96BF48?logo=shopify&logoColor=white)

| Tool | Role in Project |
|---|---|
| **BigQuery** | Cloud data warehouse; all tables, views, and SQL analysis |
| **SQL** | EDA, data cleaning views, channel attribution, RFM scoring, LTV quintiles, cohort retention |
| **Tableau** | Final interactive dashboard with dynamic Top N filter and channel quadrant |
| **Shopify Export** | Source transaction and customer data |

---

## Dataset

| Table | Source File | Description | Rows |
|---|---|---|---|
| `orders` | `shopifyData.csv` | Core transaction fact table | 417 |
| `dimAttribution` | `attributionDim.csv` | Marketing channel and device per order | 579 |
| `dimCustomer` | `customerDim.csv` | Customer ID per order | 579 |

**Date range:** March 13, 2024 — July 22, 2025
**Geographies:** 54 countries
**Order statuses:** Completed (402), Pending Payment (5), Refunded (2)

---

## Data Model

Star schema with `orders` as the fact table and two dimension tables joined on `orderNumber`.

### orders (Fact Table)

| Field | Type | Description |
|---|---|---|
| `orderNumber` | INTEGER | Primary key |
| `orderStatus` | STRING | Completed / Pending Payment / Refunded |
| `orderDate` | TIMESTAMP | Order placement timestamp |
| `countryCode` | STRING | ISO country code |
| `paymentMethod` | STRING | Card / PayPal / etc. |
| `orderRefundAmount` | FLOAT | Refund value |
| `orderTotalAmount` | FLOAT | Gross order value |
| `itemName` | STRING | Product or service purchased |
| `quantity` | INTEGER | Units ordered |

![Orders Table Schema](visuals/Marketing%20-%20orders%20table.png)

### dimAttribution

| Field | Type | Description |
|---|---|---|
| `orderNumber` | INTEGER | Foreign key to orders |
| `attributionDevice` | STRING | Desktop / Mobile |
| `attributionSource` | STRING | Marketing channel (direct, email, youtube.com, etc.) |

![dimAttribution Schema](visuals/Marketing%20-%20dimAttribution%20Table.png)

### dimCustomer

| Field | Type | Description |
|---|---|---|
| `orderNumber` | INTEGER | Foreign key to orders |
| `customerId` | FLOAT | Unique customer identifier |

![dimCustomer Schema](visuals/Marketing%20-%20dimCustomer%20Table.png)

---

## KPI Definitions

| KPI | Definition | Formula |
|---|---|---|
| **AOV** (Average Order Value) | Average net revenue per completed order | `SUM(orderTotalAmount - orderRefundAmount) / COUNT(DISTINCT orderNumber)` |
| **CRC** (Channel Revenue Contribution) | Share of total revenue by channel | `Revenue from channel / Total revenue` |
| **CRR** (Customer Refund Rate) | Percentage of net revenue refunded | `SUM(orderRefundAmount) / SUM(orderTotalAmount)` |

![KPI Definitions](visuals/Marketing%20KPI%20Definitions.png)
![Dimensions](visuals/Marketing%20Dimensions.png)
![Measures](visuals/Marketing%20Measures.png)

---

## SQL Analysis — Step by Step

### Step 1 — Initial Data Quality Checks

Validated all three tables before any cleaning:

```sql
-- Row counts across all tables
SELECT 'orders' AS table_name, COUNT(*) AS row_count FROM marketing_data.orders
UNION ALL
SELECT 'dimAttribution', COUNT(*) FROM marketing_data.dimAttribution
UNION ALL
SELECT 'dimCustomer', COUNT(*) FROM marketing_data.dimCustomer;
-- Result: orders=417, dimAttribution=579, dimCustomer=579
```

![Row Count Query](visuals/Marketing%20SQL%20Row%20count.png)

```sql
-- Duplicate order check
SELECT orderNumber, COUNT(*) AS occurrence_count
FROM marketing_data.orders
GROUP BY 1
HAVING COUNT(*) > 1;
-- 9 duplicate orderNumbers identified → handled in cleaning view
```

![Duplicate Check](visuals/Marketing%20-%20Duplicate%20orderNumber%20count%20query.png)

```sql
-- Null check for orphaned orders (no customer or attribution match)
SELECT
  SUM(CASE WHEN c.orderNumber IS NULL THEN 1 ELSE 0 END) AS no_customer,
  SUM(CASE WHEN a.orderNumber IS NULL THEN 1 ELSE 0 END) AS no_attribution
FROM marketing_data.prep o
LEFT JOIN marketing_data.dimCustomer c USING(orderNumber)
LEFT JOIN marketing_data.dimAttribution a USING(orderNumber);
-- Result: no_customer=0, no_attribution=0 ✓
```

![Null Check](visuals/Marketing%20-%20Count%20of%20null%20orders%20without%20customer%20or%20attribution.png)

```sql
-- Date range
SELECT MIN(orderDate) AS first_order, MAX(orderDate) AS last_order
FROM marketing_data.prep;
-- 2024-03-13 to 2025-07-22
```

![Date Range](visuals/Marketing%20-%20Date%20range.png)

```sql
-- Order status distribution
SELECT orderStatus, COUNT(DISTINCT orderNumber) AS count
FROM marketing_data.prep
GROUP BY orderStatus;
-- Completed: 402, Pending payment: 5, Refunded: 2
```

![Order Status](visuals/Marketing%20-%20Distrbution%20of%20order%20status.png)

```sql
-- Refund summary
SELECT
  COUNTIF(orderRefundAmount <> 0) AS refunded_orders,
  SUM(orderRefundAmount) AS total_refunded
FROM marketing_data.prep;
-- 2 refunded orders, $112.50 total refunded
```

![Refunds](visuals/Marketing%20-%20Refunded%20orders.png)

```sql
-- Unique orders vs total vs nulls
SELECT
  COUNT(*) AS total_num_orderNumber,
  COUNT(DISTINCT orderNumber) AS num_of_unique_ordernum,
  COUNTIF(orderNumber IS NULL) AS num_of_nulls
FROM marketing_data.orders;
-- 417 total, 408 unique, 0 nulls
```

![Unique Orders](visuals/Marketing%20-%20Count%20of%20unique%20orders.png)

---

### Step 2 — EDA: Distribution Analysis

```sql
-- Orders by year and month
SELECT EXTRACT(YEAR FROM orderDate) AS year,
       EXTRACT(MONTH FROM orderDate) AS month,
       COUNT(DISTINCT orderNumber) AS orders_count
FROM marketing_data.prep
GROUP BY year, month
ORDER BY year, month;
```

![Orders by Month](visuals/Marketing%20-%20Orders%20by%20Year%20and%20Month.png)

```sql
-- Distribution by country (top markets)
SELECT countryCode,
       COUNT(DISTINCT orderNumber) AS transaction_count,
       ROUND(100 * COUNT(DISTINCT orderNumber) / SUM(COUNT(DISTINCT orderNumber)) OVER(), 2) AS percentage_of_total
FROM marketing_data.prep
GROUP BY countryCode
ORDER BY transaction_count DESC;
-- US: 43.24%, GB: 14.0%, CA: 9.34%, AU: 5.65%
```

![Country Distribution](visuals/Marketing%20-%20Distribution%20of%20orders%20by%20country.png)

```sql
-- Attribution source distribution
SELECT attributionSource, COUNT(DISTINCT orderNumber) AS count
FROM marketing_data.prep
GROUP BY attributionSource
ORDER BY count DESC;
-- direct: 150, youtube.com: 141, google: 41, linkedin.com: 32
```

![Attribution Distribution](visuals/Marketing%20-%20Distribution%20by%20attribution%20channel.png)
![Attribution Devices](visuals/Marketing%20-%20Distribution%20by%20attribution%20devices.png)

```sql
-- Payment method distribution
SELECT paymentMethod, COUNT(DISTINCT orderNumber) AS count
FROM marketing_data.prep
GROUP BY paymentMethod;
-- Card: 263, PayPal: 113, PayPal Pay Later: 20, Credit Card (Stripe): 8
```

![Payment Methods](visuals/Marketing%20-%20Distribution%20of%20payment%20methods.png)

---

### Step 3 — Data Cleaning Views

```sql
-- Create base cleaned view (filter negative order amounts)
CREATE OR REPLACE VIEW marketing_data.v_cleaned_orders AS
SELECT * FROM marketing_data.prep
WHERE orderTotalAmount >= 0;
```

![Cleaned Orders View](visuals/Marketing%20-%20Created%20view%20v_cleaned_orders.png)
![Profile View](visuals/Marketing%20-%20Profile%20of%20v_cleaned_orders%20view.png)

```sql
-- Validate distinct attribution devices after cleaning
SELECT DISTINCT attributionDevice FROM marketing_data.v_cleaned_orders_2;
-- Desktop, Mobile ✓
```

![Cleaned Devices](visuals/Marketing%20-%20Cleaned%20attributionDevice.png)

```sql
-- Validate attribution sources
SELECT DISTINCT attributionSource FROM marketing_data.v_cleaned_orders_3;
-- (direct), lnkd.in, m.youtube.com, patreon.com, bing.com...
```

![Cleaned Sources](visuals/Marketing%20-%20Cleaned%20attributionSource.png)

```sql
-- Validate payment methods on completed orders
SELECT DISTINCT paymentMethod, COUNT(*) AS number
FROM marketing_data.v_cleaned_orders_4
GROUP BY 1;
-- Card: 271, PayPal: 115, Unknown: 23
```

![Cleaned Payment Methods](visuals/Marketing%20-%20Cleaned%20paymentMethod%20for%20orders%20that%20were%20%C3%A7ompleted.png)

---

### Step 4 — Core KPI Query

```sql
SELECT
  COUNT(DISTINCT orderNumber) AS total_orders,
  ROUND(AVG(orderTotalAmount), 2) AS AOV,
  ROUND(SUM(orderTotalAmount), 2) AS total_revenue
FROM marketing_data.v_cleaned_orders;
-- total_orders: 407, AOV: $134.38, total_revenue: $55,635.27
```

![Total Orders AOV Revenue](visuals/Marketing%20-%20Total%20orders%2C%20AOV%2C%20Total%20revenue.png)

---

### Step 5 — Channel Revenue Attribution

```sql
-- Revenue and orders by channel
CREATE OR REPLACE VIEW marketing_data.v_revenue_by_channel AS
SELECT
  attributionSource,
  COUNT(DISTINCT orderNumber) AS ordersCount,
  SUM(orderTotalAmount) AS totalRev,
  ROUND(AVG(orderTotalAmount), 2) AS AOV
FROM marketing_data.v_cleaned_orders_4
GROUP BY attributionSource
ORDER BY totalRev DESC;
```

```sql
-- Channel revenue share using window function
SELECT
  attributionSource,
  SUM(totalRev) AS totalRevenue,
  ROUND(100 * SUM(totalRev) / SUM(SUM(totalRev)) OVER(), 2) AS pct_of_total
FROM marketing_data.v_revenue_by_channel
GROUP BY attributionSource
ORDER BY totalRevenue DESC;
```

![Revenue by Channel](visuals/Marketing%20-%20Revenue%20by%20Channel.png)
![Total Revenue by Attribution](visuals/Marketing%20-%20Total%20Revenue%20by%20Attribution%20Source.png)

---

### Step 6 — Customer LTV View

```sql
CREATE OR REPLACE VIEW marketing_data.v_customer_ltv AS
SELECT
  customerId,
  COUNT(DISTINCT orderNumber) AS totalOrders,
  SUM(orderTotalAmount) AS totalSpend,
  MIN(orderDate) AS firstOrder,
  MAX(orderDate) AS lastOrder,
  DATE_DIFF(MAX(orderDate), MIN(orderDate), DAY) AS customerLifespanDays,
  SAFE_DIVIDE(SUM(orderTotalAmount), COUNT(*)) AS AOV
FROM marketing_data.v_cleaned_orders_4
GROUP BY customerId;
```

```sql
-- LTV quintile analysis
WITH clv_ranked AS (
  SELECT customerId, totalSpend,
    NTILE(5) OVER (ORDER BY totalSpend) AS spendQuintile
  FROM marketing_data.v_customer_ltv
)
SELECT spendQuintile,
       COUNT(*) AS customers,
       ROUND(AVG(totalSpend), 2) AS avgSpend
FROM clv_ranked
GROUP BY spendQuintile;
-- Q1: $7.63 avg | Q2: $40.16 | Q3: $128.21 | Q4: $208.30 | Q5: $356.16
```

![LTV Query](visuals/Marketing%20-%20LTV.png)
![LTV Distribution](visuals/Marketing%20-%20Distribution%20of%20Customer%20LTV.png)

---

### Step 7 — RFM Segmentation

```sql
SELECT * FROM marketing_data.v_customer_rfm;
-- Fields: customerId, recencyDays, frequency, monetary, rQuartile, fQuartile, mQuartile
```

![RFM Query](visuals/Marketing%20-%20RFM.png)

---

### Step 8 — Cohort Retention

```sql
CREATE OR REPLACE VIEW marketing_data.v_cohort_retention AS
WITH customerCohorts AS (
  SELECT customerId,
    MIN(DATE_TRUNC(DATE(orderDate), MONTH)) AS cohortMonth
  FROM marketing_data.v_cleaned_orders_4
  GROUP BY customerId
),
ordersFull AS (
  SELECT o.customerId, ...
  -- monthsSinceAcquisition calculated per cohort
)
SELECT cohortMonth, monthsSinceAcq, activeCustomers
FROM ...;
```

![Cohort Retention](visuals/Marketing%20-%20Cohort%20Retention.png)

---

## Key Findings

| Metric | Value |
|---|---|
| Total completed orders | 402 |
| Unique customers | 375 |
| Total revenue | $55,635.27 |
| AOV | $134.38 |
| Repeat purchase rate | 5.33% |
| Refunded orders | 2 ($112.50 total) |
| Champion customers (R1F1M1/2) | 13 |
| Top market | US (43.24% of transactions) |

### Channel Performance

| Channel | Orders | Total Revenue | % of Total | AOV |
|---|---|---|---|---|
| Direct | 154 | $23,543.27 | 42.39% | $147.15 |
| YouTube.com | 141 | $16,504.73 | 29.72% | $116.23 |
| Google | 43 | $8,303.93 | 14.95% | $193.11 |
| LinkedIn.com | 32 | $2,952.90 | 5.32% | $92.28 |
| Email | 15 | $1,121.46 | 2.02% | $74.76 |
| Bing.com | 4 | $900.00 | 1.62% | $225.00 |

### LTV Quintile Distribution

| Quintile | Customers | Avg Spend |
|---|---|---|
| Q1 (lowest) | 75 | $7.63 |
| Q2 | 75 | $40.16 |
| Q3 | 75 | $128.21 |
| Q4 | 75 | $208.30 |
| Q5 (highest) | 75 | $356.16 |

Top spenders (Q5) outspend bottom quintile by 47x — significant concentration at the top.

### Channel Quadrant Classification (Dashboard)

| Quadrant | Channels | Action |
|---|---|---|
| **Scale** | Direct, YouTube.com | High volume + high intent — invest more |
| **Nurture** | Google | Lower volume, higher AOV — improve conversion |
| **Optimize Conversion** | LinkedIn.com | High volume, lower intent — improve landing experience |
| **Deprioritize** | Patreon, low-signal channels | Low volume + low intent |

---

## Recommendations

1. **Build post-purchase retention flows** — Email/SMS sequences in the first 30–60 days (how-to, UGC, cross-sell) to lift repeat rate from 5.33% toward 8–10%. Track cohort M+1/M+3 lift.
2. **Invest in Champions** — 13 customers with high recency, frequency, and spend. Early access, limited drops, and surprise rewards increase AOV and order frequency.
3. **Personalize merchandising by RFM archetype** — Champions see premium bundles; Hibernating customers see reactivation offers.
4. **Reduce Direct channel dependency** — 42.39% single-channel concentration is a risk. Invest in trackable channels (email, paid search) to diversify attribution and improve measurement.
5. **Scale Google** — Highest AOV of major channels ($193.11). Low volume but high intent; improve funnel conversion to unlock growth.
6. **Deprioritize low-signal channels** — Channels in the bottom-left quadrant drain budget without delivering volume or value.

---

## Repository Structure

```
shopify-marketing-attribution/
│
├── data/
│   ├── shopifyData.csv                    # Orders fact table
│   ├── attributionDim.csv                 # Marketing channel dimension
│   └── customerDim.csv                    # Customer dimension
│
├── visuals/
│   ├── Final_Dashboard.png                # Tableau final dashboard
│   ├── Marketing Problem Statement.png
│   ├── Marketing Sub-questions.png
│   ├── Marketing KPI Definitions.png
│   ├── Marketing Dimensions.png
│   ├── Marketing Measures.png
│   ├── Marketing - orders table.png
│   ├── Marketing - dimAttribution Table.png
│   ├── Marketing - dimCustomer Table.png
│   ├── Marketing_SQL_Row_count.png
│   ├── Marketing - Count_of_unique_orders.png
│   ├── Marketing - Date_range.png
│   ├── Marketing - Distrbution_of_order_status.png
│   ├── Marketing - Number_of_orders_refunded.png
│   ├── Marketing - Refunded_orders.png
│   ├── Marketing - Duplicate_orderNumber_count_query.png
│   ├── Marketing - Count_of_null_orders_without_customer_or_attribution.png
│   ├── Marketing - Distribution_of_orders.png
│   ├── Marketing - Orders_by_Year_and_Month.png
│   ├── Marketing - Distribution_of_orders_by_country.png
│   ├── Marketing - Distribution_by_attribution_channel.png
│   ├── Marketing - Distribution_by_attribution_devices.png
│   ├── Marketing - Distribution_of_payment_methods.png
│   ├── Marketing - Cleaned_attributionDevice.png
│   ├── Marketing - Cleaned_attributionSource.png
│   ├── Marketing - Cleaned_paymentMethod_for_orders_that_were_çompleted.png
│   ├── Marketing - Created_view_v_cleaned_orders.png
│   ├── Marketing - Profile_of_v_cleaned_orders_view.png
│   ├── Marketing - Total_orders__AOV__Total_revenue.png
│   ├── Marketing - Revenue_by_Channel.png
│   ├── Marketing - Total_Revenue_by_Attribution_Source.png
│   ├── Marketing - LTV.png
│   ├── Marketing - Distribution_of_Customer_LTV.png
│   ├── Marketing - RFM.png
│   └── Marketing - Cohort_Retention.png
│
├── docs/
│   ├── Marketing_Project_Name_and_Scenario.docx
│   └── Marketing_-_Insights_and_Recommendations.docx
│
└── README.md
```

---

## How to Explore This Project

1. **Start with the docs** — `Marketing_Project_Name_and_Scenario.docx` frames the business problem
2. **Review the data model** — schema screenshots in `visuals/` show the star schema structure
3. **Follow the SQL steps** — EDA → cleaning → KPIs → channel attribution → LTV → RFM → cohort retention
4. **Explore the dashboard** — `visuals/Final%20Dashboard.png` shows the full Tableau output with channel quadrant analysis
5. **Read the findings** — `Marketing_-_Insights_and_Recommendations.docx` for the final summary

---

## About

Built as a portfolio project demonstrating e-commerce analytics: cloud data warehousing in BigQuery, multi-step SQL view architecture, marketing attribution analysis with window functions, RFM customer segmentation, LTV quintile analysis, cohort retention modeling, and Tableau dashboard design — using real Shopify export data.

**Data source:** Shopify e-commerce export — orders, attribution, and customer data (March 2024 — July 2025).
