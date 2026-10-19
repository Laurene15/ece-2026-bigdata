## Questions

1. Section S3 configuration

The persistent secret is stored in a file of your home directory. What are the risks, and why are they limited on Onyxia?

Unauthorized access to local files poses a risk but Onyxia workspace isolation and temporary credentials limit it.

The secret is visible to any process running as your user. How would you restrict the access to S3 for a Kubernetes Job?

We can use Kubernetes ServiceAccounts with IAM roles instead of persistent stored secrets.

2. Section Query the bronze layer

The dataset contains 50 users, each one with at least one order, and 2829 orders with unique identifiers. Why does approx_unique return 40 and 3167?

It uses a probabilistic estimation algorithm rather than an exact count.

The types are inferred from a sample of the file. Which issue may occur with a large file whose first rows are not representative?

Incorrect type inference can cause runtime conversion or binder errors.

3. Section Parquet export

Why is the uuid column barely compressed?

Random unique identifiers lack repeating patterns or redundancy for compression algorithms.

Why is the compressed size of some columns larger than their uncompressed size?

Metadata and checksums added by Parquet can outweigh tiny raw data values.

4. Section Hive partitioning

Why is partitioning by uuid a bad idea? Refer to the limitations of object storage presented in the previous module.

It creates excessive folders and small files causing S3 request limits and degraded performance.

Which partition column would you choose for a dataset of orders growing every day?

You would choose the date column to enable effective partition pruning.

5. Section CSV vs. Parquet at scale

How many row groups does the file contain, and how many are skipped by the filter date >= '2100-01-01'?

The file contains 9 row groups and early groups are skipped using footer statistics.

What would happen to the efficiency of the filter if they were shuffled?

Minimum and maximum statistics would overlap across groups breaking row group pruning and forcing a full scan.

Compare the execution time of the CSV and Parquet queries. Which part of the difference is due to the network, which part to the parsing of the CSV file?

Network gains come from downloading only necessary column chunks while CPU gains come from avoiding text parsing.

## Exercises

1. What is the average number of orders per user, and the average quantity per order?

SELECT 
  COUNT(DISTINCT uuid) / COUNT(DISTINCT user_uuid) AS avg_orders_per_user,
  AVG(quantity) AS avg_quantity_per_order
FROM read_csv(getvariable('bucket') || '/bronze/orders.csv');

2. For each user, what is the date of their first and last order, and the number of days between them?

SELECT 
  user_uuid,
  MIN(CAST(date AS DATE)) AS first_order_date,
  MAX(CAST(date AS DATE)) AS last_order_date,
  MAX(CAST(date AS DATE)) - MIN(CAST(date AS DATE)) AS days_between
FROM read_csv(getvariable('bucket') || '/bronze/orders.csv')
GROUP BY user_uuid;

3. Which hour of the day has the highest quantity sold? Use the hour function.

SELECT 
  hour(CAST(date AS TIMESTAMP)) AS order_hour,
  SUM(quantity) AS total_quantity
FROM read_csv(getvariable('bucket') || '/bronze/orders.csv')
GROUP BY order_hour
ORDER BY total_quantity DESC
LIMIT 1;

4. For each product, what is the month-over-month variation of the quantity sold, in percent? Use the lag window function.

WITH monthly_sales AS (
  SELECT 
    product,
    date_trunc('month', CAST(date AS TIMESTAMP)) AS order_month,
    SUM(quantity) AS total_quantity
  FROM read_csv(getvariable('bucket') || '/bronze/orders.csv')
  GROUP BY product, order_month
),
sales_with_lag AS (
  SELECT 
    product,
    order_month,
    total_quantity,
    LAG(total_quantity) OVER (PARTITION BY product ORDER BY order_month) AS prev_month_quantity
  FROM monthly_sales
)
SELECT 
  product,
  order_month,
  total_quantity,
  prev_month_quantity,
  ROUND(((total_quantity - prev_month_quantity) / prev_month_quantity) * 100, 2) AS mom_variation_pct
FROM sales_with_lag
ORDER BY product, order_month;

5. Which users ordered every product at least once?

SELECT user_uuid
FROM read_csv(getvariable('bucket') || '/bronze/orders.csv')
GROUP BY user_uuid
HAVING COUNT(DISTINCT product) = (
  SELECT COUNT(DISTINCT product) 
  FROM read_csv(getvariable('bucket') || '/bronze/orders.csv')
);