# CRM Sales Opportunities Analysis

## Introduction

In this project, I analysed a CRM Sales Opportunities dataset using SQL in BigQuery. My goal was to understand how the sales teams were performing, which products were generating revenue, how sales performance changed over time, and whether some products had better win rates than others.

I worked with four tables and used SQL to answer business questions that could help a business understand its sales performance and identify areas that may need further investigation.

## Background

The dataset contains sales opportunities, sales agents, managers, customer accounts, and product information.

I wanted to use the data to answer the following questions:

1. How is each sales team performing compared to the others?
2. How do products perform across different sales teams?
3. Are there any quarter-over-quarter (QoQ) trends in sales performance?
4. Do any products have a better win rate than others?

Before starting the analysis, I cleaned and prepared the dataset in Google Sheets. I then moved to BigQuery to perform data quality checks, write SQL queries, and analyse the results.

## Tools I Used

- **Google Sheets:** For data cleaning and preparation.
- **Google BigQuery (SQL):** For data quality checks, data analysis, aggregations, and calculating sales performance metrics.

## The Analysis

### 1. Sales Team Performance

I compared sales managers using won deals, lost deals, win rate, won revenue, and average won-deal value.

The results showed why it is important to look at more than one metric when evaluating sales performance. A team may win many deals, but its revenue and average deal value may differ from those of another team.

I also compared managers' revenue quarter by quarter to understand how their performance changed throughout the year.

```sql
/* To get a clear picture of the performance of sales teams, i grouped based on the managers leading each teams. */

WITH performance AS (
  SELECT
    teams.manager AS manager,
    sales.sales_agent AS sales_agent,
    sales.deal_stage AS deal_stage,
    sales.close_value AS close_value
  FROM crm_sales_opportunities.sales_pipeline as sales
  INNER JOIN crm_sales_opportunities. sales_teams AS teams
    ON sales.sales_agent = teams.sales_agent
)

SELECT
  performance.manager,
  COUNTIF(performance.deal_stage = 'Won') AS deals_won,
  COUNTIF(performance.deal_stage = 'Lost') AS deal_lost,
  ROUND(SAFE_DIVIDE(COUNTIF(performance.deal_stage = 'Won'), COUNTIF(performance.deal_stage IN ('Won', 'Lost'))) * 100, 2) AS win_success_pct,
  SUM(IF(performance.deal_stage = 'Won', performance.close_value, 0)) AS won_revenue,
  ROUND(AVG(IF(performance.deal_stage = 'Won', performance.close_value, NULL)), 2) AS avg_deal_won_value
FROM performance
GROUP BY
  performance.manager
ORDER BY
  win_success_pct DESC;

-- checking if there is a problem with won deals

SELECT
  COUNT(*) AS won_deals,
  COUNTIF(close_value IS NULL) AS missing_values,
  COUNTIF(close_value = 0) AS zero_value_won_deals,
  MIN(close_value) AS min_value,
  MAX(close_value) AS max_value
FROM crm_sales_opportunities.sales_pipeline
WHERE
  deal_stage = 'Won';

-- I want to run a check for products cause i'm interested in why Dustin's team average is lowers than other's

WITH product1 AS (
  SELECT DISTINCT
    product
  FROM crm_sales_opportunities.sales_pipeline
),
product2 AS (
  SELECT
    product,
    COUNT(*) AS product_app
  FROM crm_sales_opportunities.products
  GROUP BY
    product
)

SELECT
  COUNT(*) AS product_count,
  SUM(CASE WHEN p2.product_app is NULL THEN 1 ELSE 0 END) AS missing_product,
  SUM(CASE WHEN p2.product_app > 1 THEN 1 ELSE 0 END) AS duplicate_product
FROM product1 AS p1
LEFT JOIN product2 AS p2
  ON p1.product = p2.product;

-- What are the contributions of each product to a teams successful deals.

WITH manager_product_win AS (
  SELECT
    teams.manager AS manager,
    sales.product AS product,
    COUNT(*) AS won_deals,
    SUM(sales.close_value) AS won_revenue,
    AVG(sales.close_value) AS avg_won_price
  FROM crm_sales_opportunities.sales_pipeline AS sales
  INNER JOIN crm_sales_opportunities.sales_teams AS teams
    ON sales.sales_agent = teams.sales_agent
  WHERE
    sales.deal_stage = 'Won'
  GROUP BY
    teams.manager,
    sales.product
),
manager_total AS (
  SELECT
    manager_product_win.manager AS manager,
    SUM(manager_product_win.won_deals) As total_won_deals
  FROM manager_product_win
  GROUP BY
    manager_product_win.manager
)

SELECT
  mp.manager,
  mp.product,
  mp.won_deals,
  mt.total_won_deals,
  ROUND(SAFE_DIVIDE(mp.won_deals, mt.total_won_deals) * 100, 2) AS won_deal_pct,
  mp.won_revenue,
  ROUND(mp.avg_won_price, 2) AS avg_won_price
FROM manager_product_win AS mp
INNER JOIN manager_total AS mt
  ON mp.manager = mt.manager
ORDER BY
  mp.manager,
  won_deal_pct DESC;

-- Now let's check the win percentage of each manager in different products

SELECT
  teams.manager,
  sales.product,
  SUM(CASE WHEN sales.deal_stage = 'Won' THEN 1 ELSE 0 END) AS won_deals,
  SUM(CASE WHEN deal_stage = 'Lost' THEN 1 ELSE 0 END) AS lost_deals,
  SUM(CASE WHEN deal_stage IN ('Won', 'Lost') THEN 1 ELSE 0 END) AS closed_opportunities,
  ROUND(SAFE_DIVIDE(SUM(CASE WHEN sales.deal_stage = 'Won' THEN 1 ELSE 0 END), SUM(CASE WHEN deal_stage IN ('Won', 'Lost') THEN 1 ELSE 0 END)) * 100, 2) AS product_win_pct
FROM crm_sales_opportunities.sales_pipeline AS sales
INNER JOIN crm_sales_opportunities.sales_teams AS teams
  ON sales.sales_agent = teams.sales_agent
GROUP BY
  teams.manager,
  sales.product
ORDER BY
  sales.product,
  product_win_pct DESC;

-- let's see how long it takes a team to win a deal for each product

SELECT
  teams.manager,
  sales.product,
  ROUND(AVG(DATE_DIFF(sales.close_date, sales.engage_date, DAY)), 2) AS avg_won_deals_days
FROM crm_sales_opportunities.sales_pipeline AS sales
INNER JOIN crm_sales_opportunities.sales_teams AS teams
  ON sales.sales_agent = teams.sales_agent
WHERE
  sales.deal_stage = 'Won'
GROUP BY
  teams.manager,
  sales.product
ORDER BY
  sales.product,
  avg_won_deals_days
```

### 2. Product Performance Across Sales Teams

I analysed product performance by manager to compare won deals, won revenue, average won-deal value, win rate, and average days to win.

This helped me examine how different products performed across the teams and how sales outcomes varied by product.

One important lesson from this analysis was that the number of deals won does not tell the whole story. Revenue, win rate, and the time taken to close deals provide additional information about sales performance.

### 3. Quarter-over-Quarter (QoQ) Trends

I analysed quarterly won revenue at four levels: the overall business, sales managers, individual sales agents, and products.

I used `DATE_TRUNC()` to group closing dates into quarters, `SUM()` to calculate quarterly revenue, and `LAG()` with `OVER()` to retrieve the previous quarter's revenue. I then calculated the percentage change to measure quarter-over-quarter growth or decline.

#### Overall Business Performance

| Quarter | Won Revenue | QoQ Change |
|---|---:|---:|
| Q1 2017 | $1,134,672 | — |
| Q2 2017 | $3,086,111 | +171.98% |
| Q3 2017 | $2,982,255 | -3.37% |
| Q4 2017 | $2,802,496 | -6.03% |

Won revenue increased significantly in Q2 before declining in Q3 and Q4. Even after the two consecutive declines, Q4 revenue remained approximately 147% higher than Q1 revenue.

To investigate the changes further, I also checked the number of won deals and the average won-deal value.

The number of won deals increased from 531 in Q1 to 1,254 in Q2, while average won-deal value increased from $2,136.86 to $2,461.01. This showed that both deal volume and average deal value contributed to the Q2 revenue increase.

#### Sales Manager Trends

All six sales managers recorded higher won revenue in Q2 than in Q1. Their performance then differed in the following quarters.

In Q3, Melvin Marxen was the only manager whose revenue increased, recording 29.83% growth. In Q4, Dustin Brinkmann and Summer Sewald recorded increases of 11.05% and 7.88%, respectively, while the other four managers experienced declines.

This showed that the overall revenue trend did not affect every sales team in the same way.

#### Sales Agent Trends

I extended the QoQ analysis to individual sales agents to understand how their revenue changed from quarter to quarter.

All 30 agents recorded higher won revenue in Q2 than in Q1. In Q3, 14 agents increased revenue while 16 recorded declines. In Q4, 13 agents increased revenue while 17 experienced declines.

Boris Faz and Marty Freudenburg were the only agents whose won revenue increased in every quarter throughout 2017.

#### Product Trends

I also compared quarterly won revenue across the seven products.

All seven products recorded revenue increases in Q2. GTXPro, GTX Plus Pro, and MG Advanced contributed approximately 81.8% of the overall revenue increase from Q1 to Q2.

In Q4, GTXPro experienced a revenue decline of $138,108. This accounted for approximately 76.8% of the overall decline in won revenue between Q3 and Q4.

These findings helped identify which products contributed most to the changes in overall revenue.

### 4. Product Win Rate

To determine whether some products had better win rates than others, I calculated the percentage of closed opportunities that ended in a win.

I counted only won and lost opportunities because open opportunities had not yet reached a final outcome.

| Product | Won Deals | Closed Deals | Win Rate |
|---|---:|---:|---:|
| MG Special | 793 | 1,223 | 64.84% |
| GTX Plus Pro | 479 | 745 | 64.30% |
| GTX Basic | 915 | 1,436 | 63.72% |
| GTXPro | 729 | 1,147 | 63.56% |
| GTX Plus Basic | 653 | 1,051 | 62.13% |
| MG Advanced | 654 | 1,084 | 60.33% |
| GTK 500 | 15 | 25 | 60.00% |

MG Special had the highest observed win rate at 64.84%, while GTX Basic recorded the highest number of won deals, with 915.

GTK 500 had the lowest observed win rate at 60%, but it had only 25 closed opportunities. This meant its result was based on a much smaller number of observations than those of the other products.

The analysis showed that the product with the highest win rate was not necessarily the product with the highest number of won deals. I would need additional metrics to compare products by revenue or profitability.

## Conclusion

Working on this project helped me practise using SQL to answer business questions rather than just retrieve data.

I used joins, aggregate functions, CTEs, conditional counting, date functions, window functions, `LAG()`, and `SAFE_DIVIDE()` to analyse sales performance from different angles.

My main findings were that won revenue increased sharply in Q2 2017 and declined over the next two quarters, that sales trends varied across managers and agents, and that a small group of products contributed most of the Q2 revenue increase. I also found that MG Special had the highest observed win rate, while GTX Basic had the most won deals.

One thing I learned from this project is that a single metric rarely tells the whole story. Looking at revenue, deal volume, average deal value, and win rate together gives a better understanding of sales performance.

There is still more I could investigate, especially the reasons behind the revenue changes and whether the differences in product win rates are meaningful. For now, this project has been a valuable part of my journey in learning data analytics and improving my SQL skills.

It was fun working through the questions, checking my results, and learning something new along the way.
