# [SQL] E-commerce Behavior Analysis

## 🎯 Objective

Analyzing user behavior and revenue performance for an e-commerce platform using Google's public Google Analytics sample dataset. The goal is to answer 10 real-world business questions covering traffic, conversion, revenue, and customer purchase behavior.

## 📁 Dataset

- **Source:** [Google Analytics Sample Dataset](https://console.cloud.google.com/bigquery?ws=!1m4!1m3!3m2!1sbigquery-public-data!2sgoogle_analytics_sample) (BigQuery public dataset)
- **Table:** `bigquery-public-data.google_analytics_sample.ga_sessions_*`
- **Period covered:** August 2016 – August 2017
- **Structure:** Each row represents one user session, with nested/repeated fields (`hits`, `hits.product`) requiring `UNNEST()` to access product- and hit-level data.
- **Note:** Google traffic appears across multiple source variants (google, google.com, sites.google.com, mail.google.com). For this analysis, each source is treated independently as extracted from raw GA data. A production-level analysis would consolidate these into channel groupings."

## 🔧 Tools Used

- **Google BigQuery** (Standard SQL)
- **Techniques:** `UNNEST()` for nested data, CTEs (`WITH`), window functions, `UNION ALL`, `JOIN` , date parsing/formatting

---

## 📊 Queries, Results & Insights

---

### Query 01: Total Visits, Pageviews & Transactions by Month (Jan–Mar 2017)

**Business question:** How did overall site traffic and purchasing activity trend across the first quarter of 2017?

**Insight:**
> - Traffic and transactions both peaked in March 2017.
> - While February saw a dip in visits (-3.9% vs January), March rebounded strongly with the highest visits (69,931), pageviews (259,522), and transactions (993) across the quarter — a 39% jump in transactions compared to January.
> - Notably, the average pageviews per visit remained relatively stable across all three months (~3.97–3.98 pages/visit), suggesting consistent user engagement quality.
> - The growth in transactions outpaced the growth in visits, indicating an improving conversion trend through Q1 2017.

**Key technique:**
Date parsing, `GROUP BY`
<details>
  <summary> Click to view the code and the result</summary>
  
  ```sql
SELECT  
  format_date('%Y%m',parse_date('%Y%m%d',date)) AS month
  ,SUM(totals.visits) as visits
  ,SUM(totals.pageviews) AS pageviews
  ,SUM(totals.transactions) AS transactions
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
WHERE _table_suffix between '01' and '04'
GROUP BY month
ORDER BY month;
```

**Result:**
<img width="643" height="111" alt="Q1" src="https://github.com/user-attachments/assets/646e7896-34c5-4478-b92d-f9443ce19ff9" />
</details>


---

### Query 02: Bounce Rate by Traffic Source (July 2017)

**Business question:** How the low-quality visitors who leave immediately across traffic sources?

**Insight:**
> 📌 Analysis focuses on traffic sources with over 1,000 visits to ensure statistically meaningful comparisons.
> Direct traffic performs best with the lowest bounce rate (43.3%), suggesting visitors who navigate directly to the site have the highest intent and familiarity with the brand.
> YouTube.com stands out as the most concerning source — despite ranking 3rd in visit volume (6,351 visits), it has the highest bounce rate among significant sources at 66.7%, meaning nearly 2 in 3 visitors leave without engaging further. This suggests a disconnect between YouTube ad/content messaging and the actual landing page experience.
> Google, the dominant traffic driver (38,400 visits — nearly double all other sources combined), sits at a mid-range bounce rate of 51.6%. Given its volume, even a small improvement here would have an outsized impact on overall site engagement.
> Partners and analytics.google.com show similar bounce rates (~52-54%), performing close to Google's average — neither a concern nor a standout.

**Key technique:**
Ratio calculation

```sql
SELECT  
  trafficSource.source AS source
  ,SUM(totals.visits) as total_visits
  ,SUM(totals.bounces) as total_no_of_bounces
  ,ROUND(SUM(totals.bounces)/SUM(totals.visits)*100.0,3) as bounce_rate
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*` 
GROUP BY trafficSource.Source
ORDER BY total_visits DESC;
```

**Result:**
<img width="637" height="431" alt="Q2" src="https://github.com/user-attachments/assets/29ac89b2-9257-46ea-b720-87a469f1cd36" />

*Showing top 15 rows ordered by total visits. Full results available by running the query.*

---

### Query 03: Revenue by Traffic Source — Weekly & Monthly (June 2017)

**Business question:** Which traffic sources generate the most revenue, and does the pattern differ by week vs month?

**Insight:**
> Direct traffic dominates revenue by a wide margin, generating $97,334 in June 2017 — over 5x more than Google ($18,757), the second-largest contributor. This suggests a strong base of returning, brand-aware customers who navigate directly to the site without needing paid or organic search.
> Weekly breakdown reveals significant revenue volatility for top sources. Direct traffic peaked sharply in Week 24 ($30,909) and Week 25 ($27,295), accounting for nearly 60% of its monthly total in just two weeks — suggesting a mid-month promotional event or campaign drove a concentrated spike rather than steady organic demand.
> Google revenue is similarly uneven, with Week 24 alone contributing $9,217 out of $18,757 monthly total (~49%) — reinforcing the mid-June spike pattern seen across multiple sources simultaneously, which strongly points to a site-wide promotion or marketing event in Week 24 (mid-June).
> Sources like youtube.com ($16.99), bing ($13.98), and l.facebook.com ($12.48) generate negligible revenue despite presumably driving some traffic, indicating poor revenue conversion from these channels in June 2017.

**Key technique:**
`UNION ALL`, `UNNEST`

```sql
SELECT  
  'Month' AS time_type
  ,format_date('%Y%m',parse_date('%Y%m%d',date)) AS time
  ,trafficSource.source AS source
  ,ROUND(SUM(product.productRevenue)/1000000.0,4) as revenue
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
  UNNEST (hits)AS hits,
  UNNEST (hits.product)AS product
WHERE product.productRevenue IS NOT NULL
GROUP BY time, trafficSource.Source

UNION ALL
SELECT  
  'Week' AS time_type
  ,format_date('%Y%V',parse_date('%Y%m%d',date)) AS time
  ,trafficSource.source AS source
  ,ROUND(SUM(product.productRevenue)/1000000.0,4) as revenue
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
  UNNEST (hits)AS hits,
  UNNEST (hits.product)AS product
WHERE product.productRevenue IS NOT NULL
GROUP BY time, trafficSource.Source
ORDER BY source, time;
```

**Result:**
<img width="792" height="490" alt="Q3" src="https://github.com/user-attachments/assets/954deb5d-e0f3-4bde-949d-2aa8cfa78c1f" />

*Showing top 17 rows ordered by source & time. Full results available by running the query.*

---

### Query 04: Conversion Rate by Traffic Source (2017)

**Business question:** Which traffic sources convert visitors into buyers most effectively?

**Insight:**
> Only 3 traffic sources generated 50 or more transactions in 2017, highlighting a highly concentrated conversion landscape. This filter threshold helps focus analysis on statistically meaningful conversion patterns rather than outliers from low-volume sources.
> DFA (Display & Video advertising) leads in conversion rate at 2.64%, despite having the lowest visit volume among the three (2,728 visits). This suggests display advertising is reaching a highly targeted, purchase-ready audience — delivering quality over quantity.
> Direct traffic ranks second (2.48% conversion rate) with by far the highest visit volume (189,447 visits) and transaction count (4,697). This combination of high volume and strong conversion rate makes direct traffic the single most valuable channel overall — indicating a loyal, brand-aware customer base that converts consistently at scale.
> Google organic/paid traffic underperforms significantly at just 0.92% — less than half the conversion rate of direct and DFA — despite driving 179,804 visits, the second highest volume. This gap suggests Google is bringing in a large number of browsers and researchers who are not yet ready to purchase.
> Cross-referencing with Q2: Google's mid-range bounce rate (51.6%) combined with its low conversion rate (0.92%) paints a consistent picture — Google traffic engages enough to stay on site but lacks the purchase intent of direct or DFA visitors.


** Key technique:**
`HAVING` filter, ratio calculation + filter 
```sql
SELECT  
  trafficSource.source AS source
  ,SUM(totals.visits) AS visits
  ,SUM(totals.transactions) AS transactions
  ,SUM(totals.transactions)/SUM(totals.visits)*100 AS conversion_rate
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
GROUP BY source
HAVING SUM(totals.transactions) >= 50
ORDER BY conversion_rate DESC;
```

**Result:**
<img width="645" height="115" alt="Q4" src="https://github.com/user-attachments/assets/dcdbb0fc-37a2-4bbe-9f27-a2ce5c4fb839" />

---

### Query 05: Average Pageviews — Purchasers vs Non-Purchasers (June–July 2017)

**Business question:** Do customers who end up buying browse significantly more pages before purchasing?

**Insight:**
> Surprisingly, non-purchasers browse significantly more pages than purchasers — roughly 3x more in both June (316.87 vs 94.02) and July (334.06 vs 124.24). This counterintuitive finding suggests that visitors who end up buying navigate with purpose and efficiency, going directly to the products they want rather than browsing extensively. In contrast, non-purchasers may be window-shopping, comparing options, or struggling to find what they need — resulting in higher pageview counts without conversion.
> Both groups show increased pageviews in July vs June — purchasers up ~32% (94 → 124) and non-purchasers up ~5% (317 → 334) — suggesting overall site engagement improved in July, with purchasing visitors showing the more significant behavioral shift.
> The 3x pageview gap between the two groups is consistent across both months, reinforcing that this is a stable behavioral pattern rather than a one-month anomaly — high pageview count alone is not a reliable indicator of purchase intent.

**Key technique:**
`LEFT JOIN` across two CTEs 

```sql
WITH avg_pageviews_purchase AS (SELECT  
                                  format_date('%Y%m',parse_date('%Y%m%d',date)) AS month
                                  ,ROUND(SUM(totals.pageviews)/COUNT(DISTINCT fullvisitorId),4) AS avg_pageviews_purchase
                                FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
                                  ,UNNEST (hits) AS hits
                                  ,UNNEST (hits.product) AS product
                                WHERE _table_suffix between '06' and '08'
                                  AND totals.transactions >= 1
                                  AND product.productRevenue IS NOT NULL
                                GROUP BY format_date('%Y%m',parse_date('%Y%m%d',date)))
    ,avg_pageviews_non_purchase AS (SELECT  
                                      format_date('%Y%m',parse_date('%Y%m%d',date)) AS month
                                      ,ROUND(SUM(totals.pageviews)/COUNT(DISTINCT fullvisitorId),4) AS avg_pageviews_non_purchase
                                    FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
                                      ,UNNEST (hits) AS hits
                                      ,UNNEST (hits.product) AS product
                                    WHERE _table_suffix between '06' and '08'
                                      AND totals.transactions IS NULL
                                      AND product.productRevenue IS NULL
                                    GROUP BY format_date('%Y%m',parse_date('%Y%m%d',date)))
SELECT
  tb1.month
  ,avg_pageviews_purchase
  ,avg_pageviews_non_purchase
FROM avg_pageviews_purchase AS tb1
LEFT JOIN avg_pageviews_non_purchase AS tb2
USING (month)
ORDER BY tb1.month;
```

**Result:**
<img width="691" height="88" alt="Q5" src="https://github.com/user-attachments/assets/b5356577-df66-44c3-a64c-278d2b3c61b8" />

---

### Query 06: Average Transactions per Purchasing User (July 2017)

**Business question:** How often does a single buyer transact within a month?

**Insight:**
> In July 2017, each purchasing user completed an average of 4.16 transactions — a notably high figure suggesting that buyers are not one-time purchasers but rather repeat transactors within the same month. This indicates strong purchasing momentum once a user decides to buy, possibly driven by a multi-item shopping behavior or multiple separate purchase sessions within July.
> This metric is particularly valuable when read alongside Q5 — where purchasers browse only ~124 pages on average. The combination of low pageviews but high transaction frequency paints a picture of a decisive, loyal buyer segment: they navigate efficiently, don't need much browsing to make up their mind, and tend to come back and buy again within the same month.

**Key technique:**
Aggregation with distinct count

```sql
SELECT  
  format_date('%Y%m',parse_date('%Y%m%d',date)) AS month
  ,ROUND(SUM(totals.transactions)/COUNT(DISTINCT fullvisitorId),4) AS Avg_total_transactions_per_user
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
  ,UNNEST (hits) AS hits
  ,UNNEST (hits.product) AS product
WHERE totals.transactions >= 1
  AND product.productRevenue IS NOT NULL
GROUP BY format_date('%Y%m',parse_date('%Y%m%d',date));
```

**Result:**
<img width="488" height="67" alt="Q6" src="https://github.com/user-attachments/assets/33199a20-9729-49ff-be0a-4a57d87145b0" />

---

### Query 07: Revenue Contribution by Device (2017)

**Business question:** Which devices drive the most revenue, and where should the business prioritize UX investment?

**Insight:**
> Desktop dominates revenue with an overwhelming 96.14% contribution ($1,674,746 out of $1,742,047 total), leaving mobile and tablet with a combined share of less than 4%.
> Mobile's 3.25% revenue share is disproportionately low given that mobile typically accounts for the majority of web traffic in e-commerce.
> Tablet contributes just 0.62% — the smallest share despite being a device often associated with comfortable browsing. This further reinforces that the site may not be optimized for touch-based navigation across non-desktop devices.
> The 96% desktop dependency represents a significant business risk — as global traffic continues shifting toward mobile, a site this reliant on desktop revenue is vulnerable to losing buyers who primarily shop on their phones.

**Key technique:**
Window function `SUM() OVER()`
```sql
SELECT
  device.deviceCategory AS category
  ,CAST(SUM(product.productRevenue)/1e6 AS STRING FORMAT '999,999,999.999') AS revenue_by_device
  ,CAST(SUM(SUM(product.productRevenue)) OVER ()/1e6 AS STRING FORMAT '999,999,999.999') AS total_revenue
  ,CAST(SUM(product.productRevenue) / SUM(SUM(product.productRevenue)) OVER () * 100 AS STRING FORMAT '99.99') AS contribution_rate_percent
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201*`,
  UNNEST(hits) AS hits,
  UNNEST(hits.product) AS product
GROUP BY category
ORDER BY SUM(product.productRevenue) DESC;
```

**Result:**
<img width="868" height="120" alt="Q7" src="https://github.com/user-attachments/assets/f05e19aa-0d2f-40e6-9569-0b871ed0836f" />

---

### Query 08: Cross-sell — Other Products Bought by "YouTube Men's Vintage Henley" Buyers (July 2017)

**Business question:** What else do customers who buy this product tend to purchase? Can we build a recommendation or bundle strategy?

**Insight:**
> Customers who purchased the YouTube Men's Vintage Henley showed a clear tendency to buy other Google and YouTube branded merchandise in the same session. The top co-purchased items — Google Sunglasses (20), Google Women's Vintage Hero T-shirt (7), and SPF-15 Slim & Slender Lip Balm (6) — suggest these buyers are brand fans shopping across the Google/YouTube merchandise ecosystem rather than one-time single-item purchasers.
> Google Sunglasses stands out as the strongest cross-sell opportunity, ordered 20 times alongside the Henley — nearly 3x more than any other product. Featuring it as a "You might also like" recommendation on the Henley product page could directly increase average order value.

**Key technique:**
CTE for customer list

```sql
WITH ctm_list AS (SELECT  
                    DISTINCT fullVisitorId
                  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
                    ,UNNEST(hits) AS hits
                    ,UNNEST(hits.product) AS product
                  WHERE product.v2productName = 'YouTube Men\'s Vintage Henley'
                    AND product.productRevenue IS NOT NULL)
SELECT
  product.v2ProductName AS other_purchased_products
  ,SUM(product.productQuantity) AS quantity
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
  ,UNNEST(hits) AS hits
  ,UNNEST(hits.product) AS product
WHERE fullVisitorId IN (SELECT * FROM ctm_list)
  AND product.v2ProductName <> 'YouTube Men\'s Vintage Henley'
  AND product.productRevenue IS NOT NULL
GROUP BY product.v2ProductName
ORDER BY quantity DESC;
```

**Result:**
<img width="391" height="543" alt="Q8" src="https://github.com/user-attachments/assets/ffa23a4e-8d73-4a99-9438-c2bd83d1d63b" />

*Showing top 19 rows ordered by quantity bought. Full results available by running the query.*

---

### Query 09: Conversion Funnel — Product View → Add to Cart → Purchase (Jan–Mar 2017)

**Business question:** Where in the purchase funnel do we lose the most customers?

**Insight:**
> 📌 The conversion funnel improved consistently across all three months, with both add-to-cart rate (28.47% → 37.29%) and purchase rate (8.31% → 12.64%) trending upward from January to March. This suggests either improving site experience, better product-market fit, or increasingly targeted traffic quality through Q1 2017.
> The biggest drop-off occurs at the product view → add-to-cart stage — even at its best in March, 62.71% of product viewers leave without adding anything to cart. This is where the most significant conversion opportunity lies: improving product page experience (better images, clearer pricing, stronger CTAs) could meaningfully move this needle.
> February shows an interesting pattern — product views dropped significantly (-16.7% vs January, from 25,787 to 21,489) yet add-to-cart numbers stayed almost flat (7,342 → 7,360), resulting in the highest add-to-cart rate jump (+5.78 percentage points). This suggests February attracted fewer but higher-intent visitors — people who arrived already knowing what they wanted. This aligns with the lower visit volume seen across Q2 data for February.
> March delivered the strongest overall funnel performance — highest add-to-cart rate (37.29%), highest purchase rate (12.64%), and highest absolute purchases (2,977). The simultaneous improvement across all funnel stages suggests a compounding effect: better traffic quality feeding into a better converting funnel.
**Key technique:**
Multi-CTE and `LEFT JOIN` 

```sql
WITH purchase AS (SELECT
              format_date('%Y%m',parse_date('%Y%m%d',date)) AS month
              ,COUNT(products.productSKU) num_purchased
            FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
              ,UNNEST (hits) AS hits
              ,UNNEST (hits.product) AS products
            WHERE _table_suffix BETWEEN '0101' AND '0331'
              AND products.productRevenue IS NOT NULL
              AND ecommerceaction.action_type = '6'
            GROUP BY month)
    ,product_view AS (SELECT
              format_date('%Y%m',parse_date('%Y%m%d',date)) AS month
              ,COUNT(products.productSKU) AS num_product_view
            FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
              ,UNNEST (hits) AS hits
              ,UNNEST (hits.product) AS products
            WHERE _table_suffix BETWEEN '0101' AND '0331'
              AND ecommerceaction.action_type = '2'
            GROUP BY month)
    ,add_to_cart AS (SELECT
              format_date('%Y%m',parse_date('%Y%m%d',date)) AS month
              ,COUNT(products.productSKU) AS num_addtocart
            FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
              ,UNNEST (hits) AS hits
              ,UNNEST (hits.product) AS products
            WHERE _table_suffix BETWEEN '0101' AND '0331'
              AND ecommerceaction.action_type = '3'
            GROUP BY month)
SELECT
  tb1.month
  ,tb1.num_product_view
  ,tb2.num_addtocart
  ,tb3.num_purchased
  ,ROUND(tb2.num_addtocart/tb1.num_product_view*100.0,2) AS addtocart_rate
  ,ROUND(tb3.num_purchased/tb1.num_product_view*100.0,2) AS purchase_rate
FROM product_view AS tb1
LEFT JOIN add_to_cart AS tb2 USING (month)
JOIN purchase AS tb3 USING (month)
ORDER BY month;
```

**Result:**
<img width="894" height="117" alt="q9" src="https://github.com/user-attachments/assets/cf294752-99e3-4f48-82d3-1f22e490c744" />

---

### Query 10: Weekly Revenue & Cumulative Revenue (May–July 2017)

**Business question:** How does revenue accumulate week over week, and are there any notable spikes or dips?

**Insight:**
> Revenue grew steadily from May to July, reaching a cumulative total of $425,258 over the three-month period.
> Revenue spiked significantly in Week 24 (June, $45,054) and Week 29 (July, $57,111), with multiple traffic sources surging simultaneously in Week 24. The cause of these spikes — whether promotional campaigns, seasonal demand, or external events — cannot be determined from this dataset alone and would require cross-referencing with marketing campaign data or promotion calendars.
> Week 26 shows an anomaly worth noting — it appears twice in the data (rows 10 and 11), split across June and July months, with revenue of $24,033 and $779 respectively. The $779 figure for the July portion of Week 26 is unusually low and likely reflects only a partial day or two at the month boundary rather than a genuine revenue dip. This is a data artifact from how BigQuery handles week-month boundaries rather than a real business signal.
> May showed a declining mid-month trend (Week 19: $34,741 → Week 21: $20,199) before recovering slightly in Week 22, suggesting weaker mid-May performance that could warrant investigation into whether promotional activity or seasonality was a factor.

**Key technique:**
Running total with window function

```sql
WITH tb1 AS (SELECT  
              format_date('%Y-%m',parse_date('%Y%m%d',date)) AS month
              ,format_date('%Y-%V',parse_date('%Y%m%d',date)) AS week
              ,SUM(product.productrevenue)/1000000 as weekly_revenue
            FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
              UNNEST (hits)AS hits,
              UNNEST (hits.product)AS product
            WHERE _table_suffix BETWEEN '0501' AND '0731'
              AND product.productRevenue IS NOT NULL
            GROUP BY month, week
            ORDER BY week)
SELECT
  month
  ,week
  ,CAST(weekly_revenue AS STRING FORMAT '999,999,999.99')
  ,CAST(SUM(weekly_revenue) OVER(ORDER BY month, week ASC) AS STRING FORMAT '999,999,999.99')
FROM tb1
```

**Result:**
<img width="867" height="458" alt="Q10" src="https://github.com/user-attachments/assets/3b52c342-1acf-47df-b36e-2a3c550b646e" />

---
## 🔍 Key Insights
1. Direct traffic is the most valuable channel across all metrics
Consistently ranking #1 in visits, transactions, revenue, and conversion rate (2.48%), direct traffic represents a loyal, brand-aware customer base that converts efficiently and spends the most. This channel alone accounts for the majority of business value.

2. Google drives volume but not value
Despite being the second-largest traffic source (179,804 visits in 2017), Google's conversion rate is just 0.92% — less than half of direct traffic. Combined with its mid-range bounce rate (51.6% in July), Google is bringing in a large top-of-funnel audience that browses but doesn't buy — a volume-quality mismatch that represents both a challenge and an opportunity.

3. The biggest conversion drop-off happens before the cart
Q9's funnel analysis reveals that even in March (the best-performing month), 62.71% of product viewers leave without adding anything to cart. This product view → add-to-cart gap is the single largest friction point in the entire purchase journey — larger than the cart → purchase drop-off.

4. Mobile revenue is critically underperforming
Desktop accounts for 96.14% of total revenue despite mobile typically driving the majority of e-commerce traffic. Combined with mobile Facebook's high bounce rate (64.3% in Q2), this points to a systemic mobile experience issue rather than simply low mobile demand.

5. Conversion quality improved consistently through Q1 2017
Both add-to-cart rate (28.47% → 37.29%) and purchase rate (8.31% → 12.64%) trended upward from January to March — suggesting improving site experience or traffic quality heading into Q2. February's counterintuitive pattern (fewer visits but higher conversion rate) further supports that traffic quality matters more than volume.

6. Purchasing users are decisive and loyal
Q5 and Q6 together paint a clear picture: buyers browse far fewer pages than non-buyers (~124 vs ~334 pageviews) but complete an average of 4.16 transactions per month. This "low browse, high buy" behavior signals a segment of highly loyal, efficient purchasers who know what they want.


## 💡 Overall Recommendations

Based on the analysis across all 10 queries:

1. Invest in mobile experience optimization (Highest Priority)
With desktop generating 96% of revenue, any shift in user behavior toward mobile poses a major business risk. Audit the mobile checkout flow, page load speed, and product page layout to identify friction points.

2. Fix the product page to improve add-to-cart rate
The product view → add-to-cart stage loses over 60% of potential buyers even in the best month. A/B test improvements to product pages — clearer CTAs, better product images, size guides, customer reviews, or urgency signals — to move the needle on this critical conversion point.

3. Improve Google traffic quality through better targeting and landing pages
Google's 0.92% conversion rate vs direct's 2.48% suggests a targeting or landing page mismatch. Consider refining Google Ads audience targeting toward higher-intent keywords, and ensure landing pages are tightly aligned with the search queries driving traffic.

4. Leverage cross-sell opportunities at checkout
Q8 shows Google Sunglasses is purchased alongside the YouTube Men's Vintage Henley at nearly 3x the rate of any other product. Implement "frequently bought together" recommendations at the product and cart pages to increase average order value with minimal development effort.

