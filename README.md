# TheLook Ecommerce : Exploratory Data Analysis (EDA) and Cohort Analysis in BigQuery 🛒
Managed to do EDA and find out the user retention for the company to see the monthly retention until three following months.
- Tools : SQL in BiqQuery and Google Sheets

## Overview of Dataset
TheLook is a fictitious Ecommerce clothing site developed by the Looker team. The dataset contains information about customers, products, orders, logistics, web events and digital marketing campaigns. The contents of this dataset are synthetic, and are provided to industry practitioners for the purpose of product discovery, testing, and evaluation. (source : thelook-ecommerce [dataset](https://console.cloud.google.com/marketplace/product/bigquery-public-data/thelook-ecommerce?q=search&referrer=search&project=sincere-torch-350709)) 
## Exploratory Data Analysis
### 1. Get the number of unique users, number of orders, and total sale price per status and month
 ``` sql
SELECT DATE_TRUNC(DATE(created_at), MONTH) AS month_year,
       status AS status,
       COUNT(DISTINCT user_id) AS total_unique_users,
       COUNT(DISTINCT order_id) AS total_orders,
       SUM(sale_price) AS total_sales_price
FROM `bigquery-public-data.thelook_ecommerce.order_items`
WHERE created_at BETWEEN '2019-01-01' AND '2022-09-01'
GROUP BY 1, 2
ORDER BY 2, 1
  ```
 **Table results:**
 
![image](https://user-images.githubusercontent.com/100940506/215675443-156c8ae4-5d68-40d7-b2fc-76334517fe89.png)

 **Chart results:**
 
![image](https://user-images.githubusercontent.com/100940506/215696973-768b443e-e01f-432a-ae37-01086fcf4e49.png)
![image](https://user-images.githubusercontent.com/100940506/215697070-4fe0937b-817b-45e5-8960-49d17be67d55.png)
![image](https://user-images.githubusercontent.com/100940506/215697201-10c51987-9767-42df-8879-acbaa7fab796.png)

**Insight:**  

From Jan 2019- August 2022 we can see that the Number of Users, Numbers of Orders and Total Sales relatively growing over years.
### 2. Get frequencies, average order value and total number of unique users where status is complete grouped by month
```sql
SELECT DATE_TRUNC(DATE(shipped_at), MONTH) AS month_year,
       ROUND(COUNT(DISTINCT order_id)/COUNT(DISTINCT user_id), 2) AS frequency,
       ROUND(SUM(sale_price)/COUNT(DISTINCT order_id), 2) AS aov,
       COUNT(DISTINCT(user_id)) AS number_of_unique_users
FROM `bigquery-public-data.thelook_ecommerce.order_items` 
WHERE shipped_at BETWEEN '2019-01-01' AND '2022-09-01' 
      AND status = 'Complete'
GROUP BY 1
ORDER BY 1
```
 **Table results:**
 
 ![image](https://user-images.githubusercontent.com/100940506/215676610-0d053c3f-9cf7-4f53-8a56-4f24a399fb53.png)
 
 **Chart results:**
 
 ![image](https://user-images.githubusercontent.com/100940506/215699983-ab238f0f-bbaf-4dd5-be68-6574269c45f3.png)

**Insight:**  

From Jan 2019 - August 2022 we can see that the Number of Users increasing over the year but Frequencies and Aov looks stagnant.
 ### 3. Find the user id, email, first and last name of users whose status is refunded on Aug 22
 ```sql
SELECT DISTINCT (u.id) AS user_id,
       u.email AS email,
       u.first_name AS first_name,
       u.last_name AS last_name
FROM `bigquery-public-data.thelook_ecommerce.orders` AS o
LEFT JOIN `bigquery-public-data.thelook_ecommerce.users` AS u
ON o.user_id = u.id
WHERE o.returned_at BETWEEN '2022-08-01' AND '2022-09-01' 
      AND o.status = 'Returned'
ORDER BY 3
```
**Table results:**

![image](https://user-images.githubusercontent.com/100940506/215677520-6d42893b-c5a5-4dba-b009-f038b92db799.png)

**Insight:**

There are 500 Users that status is refunded on August 2022
### 4. Get the top 5 least and most profitable product over all time
```sql
WITH product_sales AS
( SELECT i.product_id AS product_id,
         p.name AS product_name, 
         ROUND(p.retail_price,2) AS retail_price, 
         ROUND(p.cost,2) AS cost,
         ROUND(i.sale_price - p.cost,2) AS profit
  FROM `bigquery-public-data.thelook_ecommerce.order_items` i
  JOIN `bigquery-public-data.thelook_ecommerce.products` p
  ON i.product_id = p.id
  WHERE status = 'Complete'), 

product_profit AS
( SELECT product_id,
         product_name,  
         retail_price, 
         cost,
         ROUND(SUM(profit),2) AS total_profit
  FROM product_sales
  WHERE product_name IS NOT NULL
  GROUP BY 1,2,3,4
  ORDER BY 2)

(SELECT *
FROM product_profit
ORDER BY 5 DESC
LIMIT 5)
UNION ALL
(SELECT *
FROM product_profit
ORDER BY 5 ASC
LIMIT 5)
```
**Table results:**

![image](https://user-images.githubusercontent.com/100940506/215678281-9e705aeb-e042-4268-8f65-0b0257ea7227.png)

**Chart results:**

![image](https://user-images.githubusercontent.com/100940506/215708629-4451f173-2148-475a-9445-c60b8d5e6a89.png)
![image](https://user-images.githubusercontent.com/100940506/215708770-88f26bc7-8439-4b80-b1ac-53a9f4881dde.png)

**Insight:**

- Canada Goose, Magaschoni, North Face Freedom, North Face Nuptse and Jordan Durasheen are top 5 Most profitable product with total profit $1598 - $1973. 
- Genuine Leather, Blank Long Cuff, Hyp Women's, Sock Co. and Aviator Sunglasses are top 5 Least profitable product with total profit $0.88 - $1.35
### 5. Get Month to Date of total profit in each product categories of past 3 months (current date 15 Aug 2022), breakdown by date
```sql
WITH profit AS
( SELECT DATE(o.shipped_at) AS order_date,
         p.category AS product_category,
         ROUND(SUM(o.sale_price - p.cost),2) AS category_profit
  FROM `bigquery-public-data.thelook_ecommerce.order_items` AS o
  INNER JOIN `bigquery-public-data.thelook_ecommerce.products` AS p
  ON o.product_id = p.id
  WHERE o.shipped_at BETWEEN "2022-01-01" AND "2022-08-16" AND status = "Complete"
  GROUP BY 1,2
  ORDER BY 2,1),

mtd_table AS
( SELECT order_date,
         product_category,
         category_profit,
         ROUND(SUM(category_profit) OVER(PARTITION BY product_category, EXTRACT(MONTH FROM order_date) ORDER BY product_category,order_date),2) AS mtd
  FROM profit
  ORDER BY 2,1)

SELECT order_date,product_category, mtd  
FROM mtd_table 
WHERE order_date BETWEEN "2022-06-01" AND "2022-08-16" 
      AND EXTRACT(DAY FROM order_date) = 15
```
**Table results:**

![image](https://user-images.githubusercontent.com/100940506/215678684-e6bf36b2-e790-4e6d-8801-478f27b0b1f8.png)
### 6. Find monthly growth of inventory in percentage breakdown by product categories, ordered by time descendingly. 
```sql
WITH inventory AS
( SELECT DATE_TRUNC(DATE(created_at), MONTH) AS month_year,
         product_category AS category,
         COUNT(id) AS stok
  FROM `bigquery-public-data.thelook_ecommerce.inventory_items`
  GROUP BY 2,1
  ORDER BY 2,1),

-- Calculate previous inventory using LAG() function
inventory_previous AS
( SELECT month_year,
         category,
         stok, 
         LAG(stok) OVER (PARTITION BY category ORDER BY month_year) AS previous_stok
  FROM inventory)

-- Calculate the inventory growth percentage
SELECT month_year, 
       category,
       CONCAT(ROUND((stok-previous_stok)/previous_stok,2), "%") AS growth
FROM inventory_previous
WHERE month_year BETWEEN '2019-01-01' AND '2022-05-01'
ORDER BY 2 ASC, 1 DESC
```
**Table results:**

![image](https://user-images.githubusercontent.com/100940506/215680349-fa8372bd-00e6-412a-9e30-d4120faaa6d1.png)

**Chart results:**

![image](https://user-images.githubusercontent.com/100940506/215683884-7d0924e2-e08d-4e20-b711-69ca1195aa90.png)

**Insights :**

There is a spike in growth of inventory for each category during Dec 2018 - Jan 2019 and Dec 2019 - Januari 2020 and continue to show stagnation ever since.

**Recommendation:**

- Need further investigation on this trend, is there any activity related to this spike? Such as stock build - up, campaign, etc.
- Will collaborate with operational, marketing & product team.

**Notes :**

- There are 26 categories of products in TheLook Ecommerce
- The time frame for analysis is from Jan 2019 until August 2022.
### 7. Create Monthly retention cohorts and then how many of them (%) coming back for the following months in 2022.
```sql
WITH complete_order AS
(   SELECT *
    FROM`bigquery-public-data.thelook_ecommerce.order_items`
    WHERE status = 'Complete' AND DATE_TRUNC(DATE(created_at), MONTH) < '2022-10-01'
),

cohort_items AS
(   SELECT user_id,
        MIN(DATE(DATE_TRUNC(created_at, MONTH))) AS cohort_month
    FROM complete_order
    GROUP BY user_id
),

user_activities AS 
(  SELECT o.user_id AS user_id,
          DATE_DIFF(DATE(DATE_TRUNC(created_at, MONTH)), cohort.cohort_month, MONTH) AS month_number
    FROM complete_order AS o
    LEFT JOIN cohort_items AS cohort
    ON o.user_id = cohort.user_id
    WHERE EXTRACT(YEAR FROM cohort.cohort_month) IN (2022)
    AND EXTRACT(YEAR FROM DATE_TRUNC(o.created_at, YEAR)) IN (2022)
    GROUP BY 1,2
),
cohort_size AS 
(   SELECT cohort_month, COUNT(cohort_month) as num_users
    FROM cohort_items
    GROUP BY cohort_month 
    ORDER BY cohort_month),

retention_table AS 
(   SELECT items.cohort_month AS cohort_month,
           act.month_number AS month_num,
           COUNT(items.cohort_month) AS num_users
    FROM user_activities AS act
    LEFT JOIN cohort_items AS items
    ON act.user_id = items.user_id
    GROUP BY cohort_month, month_number 
    ORDER BY cohort_month, month_number)

SELECT  ret.cohort_month AS cohort_month,
        size.num_users AS cohort_size,
        ret.month_num AS month_number, 
        ret.num_users AS total_user,
        ret.num_users/size.num_users AS percentage
FROM retention_table AS ret
LEFT JOIN cohort_size AS size
ON ret.cohort_month = size.cohort_month
WHERE ret.cohort_month IS NOT NULL 
      AND ret.month_num BETWEEN 0 AND 6
ORDER BY cohort_month, month_number;
```
**Table results:**

![image](https://user-images.githubusercontent.com/100940506/215680095-a721e369-56bd-4b00-909f-ef39a9be3cd8.png)

**Chart results:**

![image](https://user-images.githubusercontent.com/100940506/215683327-22d4b8b4-3c9c-4d6a-b3b2-d98456bb7c7f.png)

**Insights :**

- TheLook Ecommerce was not able to retain users, shown by the downtrend (around 3-4% to 1-1.3%) in avg retention rate from month 1 forward.
- But in the first month after first purchase, there was a slight uptrend (2.7% to 4%) in retention rate from Apr-22 cohort to Jul-22 cohort, probably caused by the increasing number of users.
- We need to identify other causes of the uptrend by deep diving into marketing activity & products features and find the characteristic from the recent cohort.

**Recommendation :**

- In order to reduce churn rate, TheLook Ecommerce can consider to figuring out marketing and campaign strategy to retain new user, increase user engagement and maintain the loyal user.
- The decreasing of user monthly retention can be caused by the lack of engagement of user and the website. TheLook can optimized their communication channel (email, social media etc), offering discount or deals, and improving service and quality. 

**Notes :**

- Cohorts are divided by month, initial start date can be defined using first purchase from each user.
- The scope of cohort analysis is for users who purchases first time at website between 2019-2022, and to see the monthly retention until three following months.
