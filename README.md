# Customer Shopping Trends Analysis

An end-to-end data analytics project analyzing customer shopping behavior using **Python** (data cleaning & feature engineering), **MySQL** (data storage & business analysis via SQL), and **Power BI** (interactive dashboard). The goal is to turn raw retail transaction data into actionable business insights on revenue drivers, customer segmentation, discount strategy, and churn risk.

---

## Table of Contents
- [Project Overview](#project-overview)
- [Business Questions Answered](#business-questions-answered)
- [Tech Stack](#tech-stack)
- [Dataset](#dataset)
- [Project Architecture](#project-architecture)
- [Repository Structure](#repository-structure)
- [Setup & Installation](#setup--installation)
- [Data Cleaning & Feature Engineering](#data-cleaning--feature-engineering)
- [SQL Analysis](#sql-analysis)
- [Power BI Dashboard](#power-bi-dashboard)
- [Key Insights](#key-insights)
- [Future Improvements](#future-improvements)
- [Author](#author)

---

## Project Overview

This project simulates a real-world analytics workflow: raw transactional data is extracted, cleaned, and enriched in Python, loaded into a relational database (MySQL), analyzed with advanced SQL (window functions, CTEs, segmentation logic), and finally visualized in an interactive Power BI dashboard for business stakeholders.

The dataset contains **3,900+ customer shopping transactions**, including demographics, purchase details, payment behavior, and subscription status.

## Business Questions Answered

- Which product categories and seasons drive the most revenue?
- Who are the highest-value customers, and how concentrated is revenue among them?
- Are customers dependent on discounts to make a purchase?
- Do subscribers generate meaningfully more value than non-subscribers?
- Which customer segments are at risk of churn?
- What does each age group prefer to buy, and how does that inform marketing?

## Tech Stack

| Layer | Tool |
|---|---|
| Data Cleaning & Feature Engineering | Python (Pandas, NumPy) |
| Database | MySQL (via XAMPP) |
| Data Loading | SQLAlchemy, PyMySQL |
| Business Analysis | SQL (window functions, CTEs) |
| Visualization | Power BI Desktop |
| Connectivity | MySQL ODBC 8.0 Driver |

## Dataset

Source: [Customer Shopping Trends Dataset (Kaggle)](https://www.kaggle.com/datasets/mohammedmokhtar77/customer-shopping-trends-dataset)

Key columns: `customer_id`, `age`, `gender`, `category`, `purchase_amount_usd`, `location`, `season`, `review_rating`, `subscription_status`, `discount_applied`, `payment_method`, `previous_purchases`, `frequency_of_purchases`.

## Project Architecture

```
Raw CSV (Kaggle)
      │
      ▼
Python (Pandas) ── clean, handle missing values/outliers, engineer features
      │
      ▼
MySQL (customerssales DB) ── structured storage + SQL business analysis
      │
      ▼
Power BI ── interactive dashboard (KPIs, segmentation, trends)
```


## Setup & Installation

### 1. Python — Clean & Load Data

```bash
pip install pandas numpy sqlalchemy pymysql
```

Run the cleaning script (see [`clean_and_feature_select.py`](./clean_and_feature_select.py)) to clean the raw CSV and engineer features such as `age_group`, `spend_tier`, and `customer_value_score`.

Push the cleaned data into MySQL:

```python
from sqlalchemy import create_engine

# XAMPP default: user=root, password="" (empty), port=3306
engine = create_engine("mysql+pymysql://root:@localhost:3306/customerssales")

df.to_sql(
    name="customer_shopping",
    con=engine,
    if_exists="replace",
    index=False,
    chunksize=1000
)
```

### 2. MySQL — Database

- Install [XAMPP](https://www.apachefriends.org/) and start the **MySQL** module
- Create a database named `customerssales` via phpMyAdmin (`http://localhost/phpmyadmin`)
- Run [`sql_queries.sql`](./sql_queries.sql) to explore the business insight queries

### 3. Power BI — Dashboard

Power BI requires a MySQL driver to connect. Two options:

**Option A — Native MySQL connector**
- Install [MySQL Connector/NET](https://dev.mysql.com/downloads/connector/net/)
- Power BI → Get Data → MySQL database → Server: `localhost:3306`, Database: `customerssales`

**Option B — ODBC connector (recommended, more reliable with local XAMPP setups)**
- Install [MySQL Connector/ODBC 8.0 (64-bit)](https://dev.mysql.com/downloads/connector/odbc/)
- Set up a System DSN: Windows → *ODBC Data Sources (64-bit)* → System DSN → Add → *MySQL ODBC 8.0 Unicode Driver* → enter Server `127.0.0.1`, Port `3306`, User `root`, Database `customerssales`
- In Power BI: Get Data → ODBC → select the DSN → connect with username `root`, password blank

Load the `customer_shopping` table, then open [`dashboard.pbix`](./dashboard.pbix) to view/edit the full report.

## Data Cleaning & Feature Engineering

Performed in Python using Pandas:

- Standardized column names (lowercase, underscores)
- Handled missing values (median for numeric, mode for categorical)
- Removed duplicate records
- Fixed data types and standardized text casing
- Capped outliers in `purchase_amount_usd` using the IQR method
- **Engineered features:**
  - `age_group` — demographic bucket (18-25, 26-35, 36-45, 46-55, 56+)
  - `spend_tier` — Low / Medium / High / Premium spend segment
  - `customer_value_score` — composite score combining spend, purchase history, and review rating
  - Binary flags: `is_subscribed`, `discount_used`, `promo_used`
- Dropped low-value columns (e.g., `item_purchased`, superseded by `category` for aggregate analysis)

Full script: [`clean_and_feature_select.py`](./clean_and_feature_select.py)

## SQL Analysis

All queries run against the `customer_shopping` table in the `customerssales` database. Full file: [`sql_queries.sql`](./sql_queries.sql)

### 1. Customer Segmentation (Value Quartiles)
Segments customers into quartiles by composite value score to prioritize retention efforts.
```sql
SELECT
    customer_id,
    purchase_amount_usd,
    previous_purchases,
    customer_value_score,
    NTILE(4) OVER (ORDER BY customer_value_score DESC) AS value_quartile
FROM customer_shopping
ORDER BY customer_value_score DESC;
```

### 2. Top-Selling Category by Location
Identifies the best-performing category in each state for regional inventory/marketing decisions.
```sql
WITH category_location_sales AS (
    SELECT location, category,
           SUM(purchase_amount_usd) AS revenue,
           RANK() OVER (PARTITION BY location ORDER BY SUM(purchase_amount_usd) DESC) AS rnk
    FROM customer_shopping
    GROUP BY location, category
)
SELECT location, category, revenue
FROM category_location_sales
WHERE rnk = 1
ORDER BY revenue DESC;
```

### 3. Discount Dependency Risk
Flags customers who exclusively purchase using discounts — a margin/churn risk if discounts are reduced.
```sql
SELECT
    customer_id,
    COUNT(*) AS total_orders,
    SUM(CASE WHEN discount_applied = 'Yes' THEN 1 ELSE 0 END) AS discounted_orders,
    ROUND(SUM(CASE WHEN discount_applied = 'Yes' THEN 1 ELSE 0 END) / COUNT(*) * 100, 1) AS pct_discounted
FROM customer_shopping
GROUP BY customer_id
HAVING pct_discounted = 100 AND total_orders > 1
ORDER BY total_orders DESC;
```

### 4. Revenue Concentration (Pareto Analysis)
Tests whether the top 20% of customers drive a disproportionate share of revenue.
```sql
WITH ranked_customers AS (
    SELECT customer_id, purchase_amount_usd,
           PERCENT_RANK() OVER (ORDER BY purchase_amount_usd DESC) AS pct_rank
    FROM customer_shopping
)
SELECT
    SUM(CASE WHEN pct_rank <= 0.2 THEN purchase_amount_usd ELSE 0 END) AS top20pct_revenue,
    SUM(purchase_amount_usd) AS total_revenue,
    ROUND(SUM(CASE WHEN pct_rank <= 0.2 THEN purchase_amount_usd ELSE 0 END)
          / SUM(purchase_amount_usd) * 100, 1) AS pct_of_total_from_top20
FROM ranked_customers;
```

### 5. Rating vs. Repeat Purchase Behavior
Tests whether review rating is actually predictive of customer loyalty.
```sql
SELECT
    CASE
        WHEN review_rating < 3 THEN 'Low (1-2.9)'
        WHEN review_rating < 4 THEN 'Medium (3-3.9)'
        ELSE 'High (4-5)'
    END AS rating_band,
    ROUND(AVG(previous_purchases), 1) AS avg_previous_purchases,
    COUNT(*) AS customers
FROM customer_shopping
GROUP BY rating_band
ORDER BY avg_previous_purchases DESC;
```

### 6. Subscriber Value Justification
Compares subscribers vs. non-subscribers on spend, loyalty, and total revenue contribution.
```sql
SELECT
    subscription_status,
    COUNT(DISTINCT customer_id) AS customers,
    ROUND(AVG(purchase_amount_usd), 2) AS avg_order_value,
    ROUND(AVG(previous_purchases), 1) AS avg_previous_purchases,
    ROUND(SUM(purchase_amount_usd), 2) AS total_revenue,
    ROUND(SUM(purchase_amount_usd) / COUNT(DISTINCT customer_id), 2) AS revenue_per_customer
FROM customer_shopping
GROUP BY subscription_status;
```

### 7. Seasonal Category Variance
Compares each category's seasonal average order value against its own overall average to spot true seasonal spikes.
```sql
WITH category_avg AS (
    SELECT category, AVG(purchase_amount_usd) AS overall_avg
    FROM customer_shopping
    GROUP BY category
)
SELECT
    cs.category,
    cs.season,
    ROUND(AVG(cs.purchase_amount_usd), 2) AS season_avg,
    ROUND(ca.overall_avg, 2) AS category_overall_avg,
    ROUND(AVG(cs.purchase_amount_usd) - ca.overall_avg, 2) AS variance_from_avg
FROM customer_shopping cs
JOIN category_avg ca ON cs.category = ca.category
GROUP BY cs.category, cs.season, ca.overall_avg
ORDER BY variance_from_avg DESC;
```

### 8. Payment Method vs. Order Size
Checks whether high-value orders skew toward specific payment methods.
```sql
SELECT
    payment_method,
    ROUND(AVG(purchase_amount_usd), 2) AS avg_order_value,
    COUNT(*) AS orders,
    ROUND(MAX(purchase_amount_usd), 2) AS highest_order
FROM customer_shopping
GROUP BY payment_method
ORDER BY avg_order_value DESC;
```

### 9. Frequency vs. Spend Mismatch (Upsell Targets)
Finds frequent buyers with below-average order value — prime candidates for upsell/bundle campaigns.
```sql
SELECT
    customer_id,
    frequency_of_purchases,
    purchase_amount_usd,
    previous_purchases,
    ROUND(purchase_amount_usd / NULLIF(previous_purchases, 0), 2) AS spend_per_past_order
FROM customer_shopping
WHERE frequency_of_purchases IN ('Weekly', 'Fortnightly')
  AND purchase_amount_usd < (SELECT AVG(purchase_amount_usd) FROM customer_shopping)
ORDER BY previous_purchases DESC
LIMIT 20;
```

### 10. Age Group × Category Affinity Matrix
Surfaces each age group's top 3 preferred categories to inform targeted marketing.
```sql
WITH ranked_affinity AS (
    SELECT
        age_group,
        category,
        COUNT(*) AS orders,
        ROUND(SUM(purchase_amount_usd), 2) AS revenue,
        RANK() OVER (PARTITION BY age_group ORDER BY COUNT(*) DESC) AS popularity_rank
    FROM customer_shopping
    GROUP BY age_group, category
)
SELECT age_group, category, orders, revenue, popularity_rank
FROM ranked_affinity
WHERE popularity_rank <= 3
ORDER BY age_group, popularity_rank;
```
> **Note:** This query uses a CTE instead of filtering directly with `HAVING` on the `RANK()` alias, since MySQL evaluates window functions after the `HAVING` clause and does not allow referencing them there (`Error #4015: Window function is allowed only in SELECT list and ORDER BY clause`). Wrapping the ranked result in a CTE and filtering in the outer query with `WHERE` resolves this.

## Power BI Dashboard

Three-page interactive report connected live to the `customerssales` MySQL database:

**Page 1 — Executive Overview:** KPI cards (Total Revenue, Total Orders, Avg Order Value, Total Customers), revenue by category, revenue by season, subscription status split.

**Page 2 — Customer Segmentation:** Spend tier × age group matrix, top 10 customers by value score, purchase frequency vs. spend scatter plot.

**Page 3 — Operations:** Revenue by payment method, revenue by location, discount impact on average order value.

Key DAX measures:
```dax
Total Revenue = SUM(customer_shopping[purchase_amount_usd])
Avg Order Value = AVERAGE(customer_shopping[purchase_amount_usd])
Total Customers = DISTINCTCOUNT(customer_shopping[customer_id])

Discounted Revenue % = 
    DIVIDE(
        CALCULATE(SUM(customer_shopping[purchase_amount_usd]), customer_shopping[discount_applied] = "Yes"),
        [Total Revenue]
    )

Revenue per Customer = DIVIDE([Total Revenue], [Total Customers])
```

All slicers (`category`, `season`, `gender`, `age_group`, `location`) are synced across pages for consistent filtering.

*(Add dashboard screenshots here once finalized — see `/screenshots`)*

## Key Insights

*(Replace with your actual findings once you run the queries/dashboard on the full dataset — specific numbers make this section far more credible to a reviewer.)*

- [Category] generated the highest revenue at $[X], accounting for [X]% of total sales
- The top 20% of customers by value score contributed [X]% of total revenue
- [X]% of customers only purchase when a discount is applied, indicating [insight]
- Subscribers spend $[X] more on average than non-subscribers and show [X]% higher repeat purchase rates
- [Season] shows the strongest seasonal lift for [category], exceeding its yearly average by $[X]

## Future Improvements

- Automate the Python → MySQL pipeline with a scheduled script (e.g., Airflow or a simple cron job)
- Add a real transaction date column to enable true time-series trend analysis
- Deploy the Power BI report to Power BI Service for online sharing
- Build a customer churn prediction model using the engineered features

## Author

**Hridyansh Kohli**
[LinkedIn](https://www.linkedin.com/in/hridyansh-kohli-608a44373/) • hridyansh.kohli2005@gmail.com
