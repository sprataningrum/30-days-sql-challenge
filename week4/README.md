# Week 4 — Advanced SQL

[← Back to Main](../README.md)

The final week covers advanced SQL techniques: complex window functions, string and date manipulation, query optimization, and a full end-to-end business analysis mini project.

---

### Day 22 — `LAG()`, `LEAD()`

The finance team wants to compare monthly revenue performance against the previous month.

> Calculate revenue per month (2022–2023), then add columns for `prev_month_revenue`, `revenue_growth`, and `growth_pct` (month-over-month growth percentage).

<details>
<summary>Solution</summary>

```sql

WITH monthly AS (
  SELECT 
        FORMAT_DATE('%Y-%m', DATE(created_at)) AS year_month,
        SUM(sale_price) AS monthly_revenue
  FROM `bigquery-public-data.thelook_ecommerce.order_items`
  WHERE EXTRACT(YEAR FROM created_at) IN (2022, 2023)
  GROUP BY year_month
),
lagged AS (
  SELECT
        year_month,
        monthly_revenue,
        LAG(monthly_revenue) OVER (ORDER BY year_month) AS prev_month_revenue
  FROM monthly
)
SELECT
      year_month,
      monthly_revenue,
      prev_month_revenue,
      monthly_revenue - prev_month_revenue AS revenue_growth,
      ROUND(
        SAFE_DIVIDE(monthly_revenue - prev_month_revenue,prev_month_revenue) * 100, 2
      ) AS growth_pct
FROM lagged
ORDER BY year_month;

```

</details>

<details>
<summary>Output</summary>

![Day 22 Output](../assets/week4/day22.png)

</details>

---

### Day 23 — `PARTITION BY`

The regional team wants to see each country's revenue contribution against the global total.

> Calculate `total_revenue` per country, `global_revenue` (all countries combined), and `pct_contribution` (contribution percentage).
> Display the **TOP 15 countries**.

<details>
<summary>Solution</summary>

```sql

WITH country_revenue AS (
  SELECT
        u.country,
        SUM(oi.sale_price) AS total_revenue_country
  FROM `bigquery-public-data.thelook_ecommerce.order_items` AS oi
  INNER JOIN `bigquery-public-data.thelook_ecommerce.users` AS u
    ON oi.user_id = u.id
  GROUP BY country
)
SELECT
      country,
      total_revenue_country,
      SUM(total_revenue_country) OVER () AS global_revenue,
      SAFE_DIVIDE(total_revenue_country, SUM(total_revenue_country) OVER ()) AS pct_contribution
FROM country_revenue
ORDER BY total_revenue_country DESC
LIMIT 15;

```

</details>

<details>
<summary>Output</summary>

![Day 23 Output](../assets/week4/day23.png)

</details>

---

### Day 24 — String Functions

The data engineering team needs text column transformation and normalization.

> From the `users` table, create the following columns:
> - `full_name` → first and last name combined
> - `email_domain` → domain extracted from email (e.g. `gmail.com`)
> - `name_length` → character length of full_name
> - `upper_country` → country name in all uppercase
>
> Display 15 rows.

<details>
<summary>Solution</summary>

```sql

SELECT
      id,
      CONCAT(first_name, ' ', last_name) AS full_name,
      SUBSTR(email, STRPOS(email, '@') + 1) AS email_domain,
      LENGTH(CONCAT(first_name, ' ', last_name)) AS name_length,
      UPPER(country) AS upper_country
FROM `bigquery-public-data.thelook_ecommerce.users`
LIMIT 15;

```

</details>

<details>
<summary>Output</summary>

![Day 24 Output](../assets/week4/day24.png)

</details>

---

### Day 25 — Date & Time Functions

Analyze customer loyalty based on how long they've been registered and their age group.

> From the `users` table, display:
> - `days_since_joined` → days since the customer registered
> - `join_year` → year of registration
> - `join_month_name` → month name of registration
> - `age_group` → age category: `< 25`, `25–40`, `> 40`

<details>
<summary>Solution</summary>

```sql

SELECT
      id,
      first_name,
      last_name,
      created_at,
      DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) AS days_since_joined,
      EXTRACT(YEAR FROM created_at) AS join_year,
      FORMAT_DATE('%B', DATE(created_at)) AS join_month_name,
      CASE
        WHEN age < 25 THEN '< 25'
        WHEN age >= 25 AND age <= 40 THEN '25-40'
      ELSE '> 40'
      END AS age_group
FROM `bigquery-public-data.thelook_ecommerce.users`

```

</details>

<details>
<summary>Output</summary>

![Day 25 Output](../assets/week4/day25.png)

</details>

---

### Day 26 — Multiple CTEs Chaining

A simple cohort analysis to measure customer retention across years.

> Of customers who **placed their first order in 2022**, what percentage **returned to order again in 2023**?
> Use 3 chained CTEs: `first_order_2022` → `repeat_in_2023` → `cohort_summary`.

<details>
<summary>Solution</summary>

```sql

WITH users_first_order_2022 AS (
  SELECT
        user_id,
        MIN(DATE(created_at)) as first_order_date
  FROM `bigquery-public-data.thelook_ecommerce.orders`
  GROUP BY user_id
  HAVING EXTRACT(YEAR FROM MIN(DATE(created_at))) = 2022 
),
users_repeat_2023 AS (
  SELECT
        DISTINCT ufo.user_id
  FROM users_first_order_2022 AS ufo
  INNER JOIN `bigquery-public-data.thelook_ecommerce.orders` AS o
    ON ufo.user_id = o.user_id
  WHERE EXTRACT(YEAR FROM o.created_at) = 2023
),
cohort_summary AS (
  SELECT
        (SELECT COUNT(*) FROM users_first_order_2022) AS total_users_2022,
        (SELECT COUNT(*) FROM users_repeat_2023) AS repeat_users_2023,
        SAFE_DIVIDE(
          (SELECT COUNT(*) FROM users_repeat_2023), (SELECT COUNT(*) FROM users_first_order_2022)
        ) AS retention_rate
)
SELECT *
FROM cohort_summary;

```

</details>

<details>
<summary>Output</summary>

![Day 26 Output](../assets/week4/day26.png)

</details>

---

### Day 27 — Self JOIN

Find pairs of customers with similar characteristics from the same city.

> Use a Self JOIN on the `users` table to find pairs of customers who are from the **same city** and have an **age difference of exactly 1 year**. Display 15 rows.

<details>
<summary>Solution</summary>

```sql

SELECT 
      u1.id AS user_1,
      u2.id AS user_2,
      u1.city,
      u1.age AS user_1_age,
      u2.age AS user_2_age
FROM `bigquery-public-data.thelook_ecommerce.users` AS u1
JOIN `bigquery-public-data.thelook_ecommerce.users` AS u2
  ON u1.id < u2.id AND u1.city = u2.city
WHERE ABS(u1.age - u2.age) = 1
LIMIT 15;

```

</details>

<details>
<summary>Output</summary>

![Day 27 Output](../assets/week4/day27.png)

</details>

---

### Day 28 — PIVOT with `CASE WHEN`

Management wants a summary of order counts by status in a pivot table format.

> Build a pivot table from the `orders` table showing the number of orders per status (`Complete`, `Cancelled`, `Returned`, `Processing`) for each month in 2023.

<details>
<summary>Solution</summary>

```sql

SELECT 
      FORMAT_DATE('%Y-%m', DATE(created_at)) AS year_month,
      SUM(CASE WHEN status = 'Complete' THEN 1 ELSE 0 END) AS complete_orders,
      SUM(CASE WHEN status = 'Cancelled' THEN 1 ELSE 0 END) AS cancelled_orders,
      SUM(CASE WHEN status = 'Returned' THEN 1 ELSE 0 END) AS returned_orders,
      SUM(CASE WHEN status = 'Processing' THEN 1 ELSE 0 END) AS processing_orders
FROM `bigquery-public-data.thelook_ecommerce.orders`
WHERE EXTRACT(YEAR FROM created_at) = 2023
GROUP BY year_month
ORDER BY year_month;

```

</details>

<details>
<summary>Output</summary>

![Day 28 Output](../assets/week4/day28.png)

</details>

---

### Day 29 — Query Optimization

Increasingly complex analytical queries need to be optimized to run efficiently in BigQuery.

> a) Write a query to calculate **total revenue per brand per category** (status = `Complete`).
>
> b) Optimize the query: apply early filters, avoid `SELECT *`, and separate logic using CTEs.
>
> c) Record the **bytes processed** before and after optimization from the Execution Details tab in BigQuery.

<details>
<summary>Solution</summary>

```sql
-- Day 29a: Original Query

SELECT 
      p.brand,
      p.category,
      SUM(oi.sale_price) AS total_revenue
FROM `bigquery-public-data.thelook_ecommerce.order_items` AS oi
INNER JOIN `bigquery-public-data.thelook_ecommerce.products` AS p
 ON oi.product_id = p.id
WHERE oi.status = 'Complete'
GROUP BY p.brand, p.category;


-- Day 29b: Optimized Query (CTE + early filter)

WITH filtered_orders AS (
  SELECT
        product_id,
        sale_price
  FROM `bigquery-public-data.thelook_ecommerce.order_items`
  WHERE status = 'Complete'
)
SELECT 
      p.category,
      p.brand,
      SUM(fo.sale_price) AS total_revenue
  FROM filtered_orders AS fo
  INNER JOIN `bigquery-public-data.thelook_ecommerce.products` AS p
    ON fo.product_id = p.id
  GROUP BY p.category, p.brand;

```

</details>

<details>
<summary>Output</summary>

**Part a — Original Query**
![Day 29a Output](../assets/week4/day29a.png)
 
**Part b — Optimized Query (CTE + early filter)**
![Day 29b Output](../assets/week4/day29b.png)


</details>

---

### Day 30 — Mini Project: End-to-End Business Analysis

A comprehensive business analysis as the final challenge — ready to be showcased as a portfolio piece.

> Build a complete analysis in 4 parts:
>
> **Part 1 — Revenue Trend (2022–2023)**
> Monthly revenue, MoM growth, and cumulative revenue.
>
> **Part 2 — Top 10 Customers**
> Highest-spending customers with their country, total orders, and average order value.
>
> **Part 3 — Product Performance**
> Top 5 categories by revenue + top 3 best-selling products per category.
>
> **Part 4 — Order Status Summary**
> Order status distribution with percentage breakdown and total revenue per status.

<details>
<summary>Solution</summary>

```sql
-- Part 1: Revenue Trend

WITH revenue AS (
  SELECT
        DATE(DATE_TRUNC(created_at, MONTH)) AS order_month,
        SUM(sale_price) AS monthly_revenue
  FROM`bigquery-public-data.thelook_ecommerce.order_items`
  WHERE EXTRACT(YEAR FROM created_at) IN (2022, 2023) AND status = 'Complete'
  GROUP BY order_month
),
lagged AS (
  SELECT
        order_month,
        monthly_revenue,
        LAG(monthly_revenue) OVER (ORDER BY order_month) AS prev_month_revenue
  FROM revenue
)
SELECT 
      order_month,
      monthly_revenue,
      prev_month_revenue,
      SAFE_DIVIDE((monthly_revenue - prev_month_revenue), prev_month_revenue) AS mom_growth,
      SUM(monthly_revenue) OVER (ORDER BY order_month) AS cumulative_revenue
FROM lagged;


-- Part 2: Top 10 Customers

SELECT
      oi.user_id,
      u.country,
      SUM(oi.sale_price) AS total_revenue,
      COUNT(DISTINCT oi.order_id) AS total_orders,
      SAFE_DIVIDE(SUM(oi.sale_price), COUNT(DISTINCT oi.order_id)) AS avg_order_value
FROM`bigquery-public-data.thelook_ecommerce.order_items` AS oi
INNER JOIN `bigquery-public-data.thelook_ecommerce.orders` AS o
  ON oi.order_id = o.order_id
INNER JOIN `bigquery-public-data.thelook_ecommerce.users` AS u
  ON o.user_id = u.id
WHERE oi.status = 'Complete'
GROUP BY oi.user_id, u.country
ORDER BY total_revenue DESC
LIMIT 10;

-- Part 3: Product Performance

WITH product_revenue AS (
  SELECT
        p.category,
        oi.product_id,
        p.name AS product_name,
        SUM(oi.sale_price) AS revenue
  FROM `bigquery-public-data.thelook_ecommerce.order_items` AS oi
  INNER JOIN `bigquery-public-data.thelook_ecommerce.products` AS p
    ON oi.product_id = p.id
  WHERE oi.status = 'Complete'
  GROUP BY p.category, oi.product_id, p.name
),
ranked_products AS (
  SELECT
        category,
        product_id,
        product_name,
        revenue,
        RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS product_rank
  FROM product_revenue
), 
top_products AS (
  SELECT *
  FROM ranked_products
  WHERE product_rank <= 3  
),
category_revenue AS (
  SELECT 
        category,
        SUM(revenue) AS total_revenue
  FROM product_revenue
  GROUP BY category
),
rank_categories AS (
  SELECT
        category,
        total_revenue,
        RANK() OVER (ORDER BY total_revenue DESC) AS category_rank
  FROM category_revenue
),
top_categories AS (
  SELECT *
  FROM rank_categories
  WHERE category_rank <= 5
)
SELECT
      tp.category,
      tp.product_name,
      tp.revenue,
      tp.product_rank,
      tc.category_rank
FROM top_products AS tp
JOIN top_categories AS tc
  USING (category)
ORDER BY category_rank, product_rank;

-- Part 4: Order Status Summary

SELECT 
      status,
      COUNT(DISTINCT order_id) AS total_orders,
      SUM(sale_price) AS total_revenue,
      SAFE_DIVIDE(COUNT(DISTINCT order_id), SUM(COUNT(DISTINCT order_id)) OVER () ) AS order_pct,
      SAFE_DIVIDE(SUM(sale_price), SUM(SUM(sale_price)) OVER () )  AS revenue_pct
FROM `bigquery-public-data.thelook_ecommerce.order_items`
GROUP BY status
ORDER BY total_orders DESC;

```

</details>

<details>
<summary>Output</summary>

**Part 1 — Revenue Trend**
![Day Part 1 Output](../assets/week4/day30a.png)
 
**Part 2 — Top 10 Customers**
![Day Part 2 Output](../assets/week4/day30b.png)

**Part 3 —  Product Performance**
![Day Part 3 Output](../assets/week4/day30c.png)
 
**Part 4 — Order Status Summary**
![Day Part 4 Output](../assets/week4/day30d.png)

</details>

---

[← Week 3](../week3/) | [Back to Main](../README.md)
