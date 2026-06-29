# [SQL (BigQuery)] E-commerce Website Analysis & User Behavior
This repository showcases my SQL skills through a practical eCommerce data analysis project. Using Google BigQuery and the public Google Analytics sample dataset

---

## 📑 Table of Contents  
1. [📌 Overview](#-overview)  
2. [📂 Data Source & Data Dictionary](#-data-source--data-dictionary)
3. [🔎 Exploring the Dataset](#-exploring-the-dataset)
4. [🚩 Final Conclusion](#-final-conclusion)

---

## 📌 Overview  
This project focuses on analyzing an eCommerce dataset from Google Analytics using **Google BigQuery**. 
Through  SQL queries i handling nested event and raw data to extracted key business metrics.
The analysis focuses on traffic quality (bounce and conversion rates by source), revenue attribution across devices .

---
## 📂 Data Source & Data Dictionary

### 📌 Data Source  
The e-commerce dataset is stored in a public Google BigQuery dataset (`bigquery-public-data.google_analytics_sample`).

* **Table Sharding:** The data is partitioned into daily tables (e.g., `ga_sessions_20170101`). To query multiple months efficiently, I utilized the table wildcard `ga_sessions_2017*` combined with the `_TABLE_SUFFIX`.
* **Nested Data:** Uses Structs (`RECORD`) and Arrays (`REPEATED RECORD`) to store multiple user interactions within a single session.

### 📖 Data Dictionary
Below are the key fields from the schema utilized in the SQL queries:

| Field Name | Type | Description |
| :--- | :--- | :--- |
| `fullVisitorId` | `STRING` | The unique visitor ID. |
| `date` | `STRING` | The date of the session in `YYYYMMDD` format. |
| `totals` | `RECORD` | Aggregate values across the session (e.g., total bounces, hits, pageviews, visits, transactions). |
| `trafficSource.source` | `STRING` | The source of the traffic (search engine, referring hostname, or `utm_source`). |
| `device.deviceCategory` | `STRING` | The type of device used (Mobile, Tablet, Desktop). |
| `hits` | `RECORD` | Repeated field containing granular data for all types of hits within the session. |
| `hits.eCommerceAction` | `RECORD` | Contains all ecommerce hit actions (e.g., Add to cart, Check out, Completed purchase). |
| `hits.product` | `RECORD` | Nested field containing Enhanced Ecommerce data (e.g., `v2ProductName`, `productQuantity`, `productRevenue`). |

*(Note: For a complete breakdown of all available fields and their definitions, please refer to the [Official Google Analytics BigQuery Export Schema](https://support.google.com/analytics/answer/3437719?hl=en)).*

---

## 🔎 Exploring the Dataset
<details>
  <summary><b>Query 01: Calculate total visit, pageview, transaction for Q1/2017</b></summary>
  
  <br>

  **🎯 Business Purpose:** To track overall website performance by measuring the volume of traffic and conversion events over the first quarter, identifying high-level growth trends.

  ```sql
  SELECT FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) AS month,
         SUM(totals.visits) AS visits,
         SUM(totals.pageviews) AS pageviews,
         SUM(totals.transactions) AS transactions
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
  WHERE _table_suffix BETWEEN '0101' AND '0331'
  GROUP BY month
  ORDER BY month;
  ```

  **Query Results:**
  
  | Month | Visits | Pageviews | Transactions |
  | :--- | :--- | :--- | :--- |
  | 201701 | 64,694 | 257,708 | 713 |
  | 201702 | 62,192 | 233,373 | 733 |
  | 201703 | 69,931 | 259,522 | 993 |
  
</details>

  **💡 Business Insights:**
  * **Traffic Quality Improved:** Although visits dropped in February, transactions increased (from 713 to 733)
  * **Strong Conversion Growth:** The conversion rate (Transactions/Visits) grew from **1.10%** in Jan, to **1.18%** in Feb, peaking at **1.42%** in March.


<details>
  <summary><b>Query 02: Bounce rate per traffic source in July 2017</b></summary>
  <br>
  **🎯 Business Purpose:** To evaluate the quality of traffic from various acquisition channels by identifying which sources bring in the most non-interacting users. 
  
  **SQL Code:**
  ```sql
SELECT
    trafficSource.source as source,
    sum(totals.visits) as total_visits,
    sum(totals.Bounces) as total_no_of_bounces,
    (sum(totals.Bounces)/sum(totals.visits))* 100.00 as bounce_rate
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
GROUP BY source
ORDER BY total_visits DESC;
  ```

  **Query Results:**

  | Source | Total Visits | Total No. of Bounces | Bounce Rate (%) |
  | :--- | :--- | :--- | :--- |
  | google | 38,400 | 19,798 | 51.56% |
  | (direct) | 19,891 | 8,606 | 43.27% |
  | youtube.com | 6,351 | 4,238 | 66.73% |
  | analytics.google.com | 1,972 | 1,064 | 53.96% |
  | Partners | 1,788 | 936 | 52.35% |
  | m.facebook.com | 669 | 430 | 64.28% |
  | google.com | 368 | 183 | 49.73% |
  | dfa | 302 | 124 | 41.06% |
  | sites.google.com | 230 | 97 | 42.17% |
  | facebook.com | 191 | 102 | 53.40% |
  </details>

<details>
  <summary><b>Query 03: Revenue by traffic source by week, by month in June 2017</b></summary>
  
  <br>

  **🎯 Business Purpose:** To measure the financial return of different marketing channels by tracking revenue on a weekly and monthly

  **SQL Code:**
  ```sql
  WITH month_data AS (
    SELECT
      "Month" AS time_type,
      FORMAT_DATE("%Y%m", PARSE_DATE("%Y%m%d", date)) AS time,
      trafficSource.source AS source,
      SUM(p.productRevenue)/1000000 AS revenue
    FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
      UNNEST(hits) hits,
      UNNEST(hits.product) p
    WHERE p.productRevenue IS NOT NULL
    GROUP BY 1, 2, 3
  ),

  week_data AS (
    SELECT
      "Week" AS time_type,
      FORMAT_DATE("%Y%W", PARSE_DATE("%Y%m%d", date)) AS time,
      trafficSource.source AS source,
      SUM(p.productRevenue)/1000000 AS revenue
    FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
      UNNEST(hits) hits,
      UNNEST(hits.product) p
    WHERE p.productRevenue IS NOT NULL
    GROUP BY 1, 2, 3
  )

  SELECT * FROM month_data
  UNION ALL
  SELECT * FROM week_data
  ORDER BY time_type;
  ```

  ***Query Results:**

  | Time Type | Time | Source | Revenue (USD) |
  | :--- | :--- | :--- | :--- |
  | Month | 201706 | (direct) | $97,333.62 |
  | Month | 201706 | google | $18,757.18 |
  | Month | 201706 | dfa | $8,862.23 |
  | Week | 201724 | (direct) | $30,908.91 |
  | Week | 201725 | (direct) | $27,295.32 |
  | Week | 201723 | (direct) | $17,325.68 |
  | Week | 201726 | (direct) | $14,914.81 |
  | Week | 201724 | google | $9,217.17 |
  | Week | 201722 | (direct) | $6,888.90 |
  | Week | 201726 | google | $5,330.57 |

  </details>

  **💡 Business Insights:**
  
* By connecting Queries 02 and 03, we can see : this website gain revenue base on brand loyalty rather than discovery.

 **The Insight:** While Google Search brings in the sheer volume of users, they bounce at 51.5%, and Social (YouTube/Facebook) bounces at over 64%. Yet, Direct traffic generates almost 84% of the monthly revenue ($97k vs Google's $18k). Most of our revenue comes from loyal customers who already know the website. Our marketing on Google and Social Media is bringing in people, but they aren't buying.
**Strategic Implication:** Your acquisition channels (Social, Organic/Paid Search) are deeply misaligned with your landing page experience. Users clicking from YouTube or Facebook expect a specific story or product, but hit a wall of friction upon landing. The marketing budget spent on these channels is currently bleeding money. 
**Actionable Step:** We need to improve the landing pages for Google and Social Media visitors so they stay and shop, rather than leaving right away.

<details>
  <summary><b>Query 04: Average number of pageviews by purchaser type (June & July 2017)</b></summary>
  
  <br>

  **🎯 Business Purpose:** Comparing the browsing behavior (average pageviews) of paying customers and non-purchasers, identifying potential friction in the shopping experience.

  **SQL Code:**
  ```sql
  WITH purchase AS (
    SELECT
      FORMAT_DATE('%Y-%m', PARSE_DATE('%Y%m%d', date)) AS Month,
      SUM(totals.pageviews)/COUNT(DISTINCT fullVisitorId) AS avg_pageviews_purchase
    FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
      UNNEST(hits) AS hits,
      UNNEST(hits.product) AS product
    WHERE date BETWEEN '20170601' AND '20170731'
      AND totals.transactions >= 1
      AND product.productRevenue IS NOT NULL
    GROUP BY Month 
  ),
  
  no_purchase AS (
    SELECT
      FORMAT_DATE('%Y-%m', PARSE_DATE('%Y%m%d', date)) AS Month,
      SUM(totals.pageviews)/COUNT(DISTINCT fullVisitorId) AS avg_pageviews_non_purchase
    FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
      UNNEST(hits) AS hits,
      UNNEST(hits.product) AS product
    WHERE date BETWEEN '20170601' AND '20170731'
      AND totals.transactions IS NULL
      AND product.productRevenue IS NULL
    GROUP BY Month 
  )

  SELECT
    p.Month,
    p.avg_pageviews_purchase,
    np.avg_pageviews_non_purchase
  FROM purchase p
  LEFT JOIN no_purchase np ON p.Month = np.Month
  ORDER BY p.Month ASC;
  ```

  **Query Results:**

  | Month | Avg Pageviews (Purchasers) | Avg Pageviews (Non-Purchasers) |
  | :--- | :--- | :--- |
  | 2017-06 | 94.02 | 316.87 |
  | 2017-07 | 124.24 | 334.06 |

  </details>

  **💡 Business Insights:**
  * **Focused Buyers vs Window Shoppers:** Non-purchasers view significantly more pages per user (~316-334 pages) compared to actual purchasers (~94-124 pages).
A user willing to click through 300+ pages wants to buy something (high intent), but they cannot find what they are looking for, or they are hitting out-of-stock roadblocks, broken filters, or a confusing site architecture. This also show that paying customers: they know exactly what they want, find it quickly, and check out.
  * **Actionable Step:** Audit the search bar, filters, and categories to identify why non-purchasers are scrolling through 300+ pages without finding products


<details>
  <summary><b>Query 05: Average number of transactions per user that made a purchase in July 2017</b></summary>
  
  <br>

  **🎯 Business Purpose:** To calculate the average purchase frequency per buying customer or repeat purchasing behavior within a specific timeframe.

  **SQL Code:**
  ```sql
  SELECT
      FORMAT_DATE("%Y%m", PARSE_DATE("%Y%m%d", date)) AS Month,
      SUM(totals.transactions) / COUNT(DISTINCT fullVisitorId) AS Avg_total_transactions_per_user
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`,
      UNNEST(hits) AS hits,
      UNNEST(hits.product) AS product
  WHERE totals.transactions >= 1
      AND product.productRevenue IS NOT NULL
  GROUP BY Month;
  ```

  **Query Results:**

  | Month | Avg Total Transactions Per User |
  | :--- | :--- |
  | 201707 | 4.16390041493776 |

  </details>

<details>
  <summary><b>Query 06: Average amount of money spent per session. Only include purchaser data in July 2017</b></summary>
  
  <br>

  **🎯 Business Purpose:** To determine the average revenue generated per purchasing session (effectively measuring the Average Order Value - AOV)

  **SQL Code:**
  ```sql
SELECT
      FORMAT_DATE('%Y-%m', PARSE_DATE('%Y%m%d', date)) AS Month,
      ROUND(SUM(product.productRevenue)/1e6/ COUNT(totals.visits),2)
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*` 
,UNNEST(hits) AS hits
,UNNEST(hits.product) AS product
WHERE product.productRevenue IS NOT NULL 
AND totals.transactions IS NOT NULL 
GROUP BY Month
ORDER BY Month;
  ```

  **Query Results:**

  | Month | Avg Revenue Per Session (USD) |
  | :--- | :--- |
  | 2017-07 | 43.86

</details>
Combining Queries 05 and 06 found a unusual e-commerce purchasing pattern.
The Insight: In July, the average paying customer completed 4.16 transactions in a single month. For a standard B2C apparel/merchandise store, ordering 4 times a month is high. However, the Average Order Value (AOV) is relatively low at $43.86.

Actionable Step: Since these users buy frequently but spend little per session, implementing a loyalty program, or aggressive "Buy 2 Get 1" could increase the AOV without hurting their high purchase frequency.

<details>
  <summary><b>Query 07: Other products purchased by customers who bought "YouTube Men's Vintage Henley" (July 2017)</b></summary>
  
  <br>

  **🎯 Business Purpose:** To perform Market Basket Analysis by identifying product affinities (items frequently bought together). 

  **SQL Code:**
  ```sql
 WITH customer_id as (
  SELECT 
      DISTINCT (fullVisitorId) AS user_id
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*` ,
  UNNEST(hits) hits,
  UNNEST(hits.product) product
WHERE totals.transactions >= 1
  AND productRevenue IS NOT NULL
  AND product.v2ProductName = "YouTube Men's Vintage Henley" 
)

SELECT 
    product.v2ProductName AS other_purchased_products,
    SUM(product.productQuantity) quantity
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*` 
JOIN customer_id
ON customer_id.user_id = fullVisitorId,
UNNEST(hits) hits,
UNNEST(hits.product) product
WHERE totals.transactions >= 1
  AND productRevenue IS NOT NULL
  AND v2ProductName != "YouTube Men's Vintage Henley"
GROUP BY other_purchased_products
ORDER BY quantity DESC;
  ```

 **Query Results:**

  | Other Purchased Products | Quantity |
  | :--- | :--- |
  | Google Sunglasses | 20 |
  | Google Women's Vintage Hero Tee Black | 7 |
  | SPF-15 Slim & Slender Lip Balm | 6 |
  | Google Women's Short Sleeve Hero Tee Red Heather | 4 |
  | YouTube Men's Fleece Hoodie Black | 3 |
  | Google Men's Short Sleeve Badge Tee Charcoal | 3 |
  | Crunch Noise Dog Toy | 2 |
  | Android Wool Heather Cap Heather/Black | 2 |
  | Recycled Mouse Pad | 2 |
  | Red Shine 15 oz Mug | 2 |

  </details>


<details>
  <summary><b>Query 08: Calculate cohort map from product view to add to cart to purchase in Q1 2017</b></summary>
  
  <br>

  **🎯 Business Purpose:** To analyze the e-commerce conversion funnel by tracking the user journey from product views to cart additions and final purchases. This helps identify major drop-off points and optimize the shopping experience.

  **SQL Code:**
  ```sql
  WITH product_view AS (
    SELECT
      FORMAT_DATE("%Y%m", PARSE_DATE("%Y%m%d", date)) AS month,
      COUNT(product.productSKU) AS num_product_view
    FROM `bigquery-public-data.google_analytics_sample.ga_sessions_*`,
      UNNEST(hits) AS hits,
      UNNEST(hits.product) AS product
    WHERE _TABLE_SUFFIX BETWEEN '20170101' AND '20170331'
      AND hits.eCommerceAction.action_type = '2'
    GROUP BY 1
  ),

  add_to_cart AS (
    SELECT
      FORMAT_DATE("%Y%m", PARSE_DATE("%Y%m%d", date)) AS month,
      COUNT(product.productSKU) AS num_addtocart
    FROM `bigquery-public-data.google_analytics_sample.ga_sessions_*`,
      UNNEST(hits) AS hits,
      UNNEST(hits.product) AS product
    WHERE _TABLE_SUFFIX BETWEEN '20170101' AND '20170331'
      AND hits.eCommerceAction.action_type = '3'
    GROUP BY 1
  ),

  purchase AS (
    SELECT
      FORMAT_DATE("%Y%m", PARSE_DATE("%Y%m%d", date)) AS month,
      COUNT(product.productSKU) AS num_purchase
    FROM `bigquery-public-data.google_analytics_sample.ga_sessions_*`,
      UNNEST(hits) AS hits,
      UNNEST(hits.product) AS product
    WHERE _TABLE_SUFFIX BETWEEN '20170101' AND '20170331'
      AND hits.eCommerceAction.action_type = '6'
      AND product.productRevenue IS NOT NULL
    GROUP BY 1
  )

  SELECT
    pv.*,
    num_addtocart,
    num_purchase,
    ROUND(num_addtocart * 100 / num_product_view, 2) AS add_to_cart_rate,
    ROUND(num_purchase * 100 / num_product_view, 2) AS purchase_rate
  FROM product_view pv
  LEFT JOIN add_to_cart a ON pv.month = a.month
  LEFT JOIN purchase p ON pv.month = p.month
  ORDER BY pv.month;
  ```

**Query Results:**

  | Month | Num Product View | Num Add To Cart | Num Purchase | Add To Cart Rate (%) | Purchase Rate (%) |
  | :--- | :--- | :--- | :--- | :--- | :--- |
  | 201701 | 25,787 | 7,342 | 2,143 | 28.47% | 8.31% |
  | 201702 | 21,489 | 7,360 | 2,060 | 34.25% | 9.59% |
  | 201703 | 23,549 | 8,782 | 2,977 | 37.29% | 12.64% |

  
</details>
Queries 01 and 08 show an interesting trend in the first quarter of the year:

* **The Insight:** From January to March, total traffic dropped, but the final purchase rate actually went up steadily from 8.3% to 12.6%.
* **Strategic Implication:** Having fewer visitors isn't a bad thing if the site is getting better at turning them into buyers. The quality of our traffic is improving.
---

## 🚩 Final Conclusion
---
* Working with the Google Analytics dataset helped me bridge the gap between technical SQL skills and business strategy:
* **CTEs (WITH):** Structured complex logic (like conversion funnels) into clean, readable queries.
* **Nested Data:** Used UNNEST() in BigQuery to extract information from complex arrays.
* **KPI Calculations:** Applied advanced aggregations (SUM, COUNT(DISTINCT)) to measure metrics like Bounce Rate and Average Order Value.
* **Date Functions:** Used FORMAT_DATE and PARSE_DATE to analyze time-based trends.
* **Business Mindset:** Learned to use SQL not just to pull data, but to uncover actionable insights and solve real business problems.
---
**Thank you for reading!** Feel free to reach out or connect if you have any questions or feedback regarding this analysis.


