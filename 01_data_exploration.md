# Module 01: Data Exploration

## Objective

Before designing anything, we need to understand what data we actually have. This module walks through the GA4 dataset in BigQuery so that the design decisions in Module 02 are grounded in reality.

---

## Step 1: Access the Dataset

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Open **BigQuery** from the left menu
3. In the Explorer panel, click **+ ADD** → **Star a project by name**
4. Type `bigquery-public-data` and click **Star**
5. Expand `bigquery-public-data` → scroll to `ga4_obfuscated_sample_ecommerce`
6. Click the `events_*` table to see the schema

You will see that there is only one logical table here. GA4 stores each day's data in a separate table named `events_YYYYMMDD`. The `events_*` notation means we can query across all of them at once using a wildcard.

---

## Step 2: Understand the Scale

Run this query to understand what you are working with.

```sql
SELECT
  COUNT(*)                    AS total_events,
  COUNT(DISTINCT user_pseudo_id) AS unique_users,
  COUNT(DISTINCT event_date)  AS days_of_data,
  MIN(event_date)             AS first_date,
  MAX(event_date)             AS last_date
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
```

**Expected result:** ~3 months of data, hundreds of thousands of events, tens of thousands of users.

> **Cost note:** This query scans the full dataset. It uses roughly 0.5GB of your 1TB free monthly quota. You will only run this once.

---

## Step 3: What Event Types Exist?

GA4 is entirely event-based. Every user action is recorded as a named event. Run this to see what events are in the dataset.

```sql
SELECT
  event_name,
  COUNT(*)        AS event_count,
  COUNT(DISTINCT user_pseudo_id) AS unique_users
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
GROUP BY event_name
ORDER BY event_count DESC
```

**Events you will see and what they mean:**

| Event Name | What It Means |
|---|---|
| `page_view` | User loaded a page |
| `session_start` | User started a new session |
| `first_visit` | User visited the site for the first time |
| `user_engagement` | User actively engaged with a page |
| `view_item` | User viewed a product detail page |
| `view_item_list` | User viewed a list of products |
| `add_to_cart` | User added an item to their cart |
| `begin_checkout` | User started the checkout process |
| `add_payment_info` | User entered payment information |
| `purchase` | User completed a purchase |
| `scroll` | User scrolled on a page |

The events from `view_item` through `purchase` form the **purchase funnel** — this is the core of what we will model.

---

## Step 4: The Nested Schema Problem

GA4 stores additional information about each event in a nested array called `event_params`. This is what makes GA4 data in BigQuery challenging to work with.

Run this to see what it looks like:

```sql
SELECT
  event_name,
  event_params
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
WHERE event_name = 'purchase'
  AND _TABLE_SUFFIX = '20201101'
LIMIT 3
```

Click on one of the `event_params` values. You will see something like:

```json
[
  {"key": "transaction_id", "value": {"string_value": "T_12345"}},
  {"key": "value",          "value": {"double_value": 59.99}},
  {"key": "currency",       "value": {"string_value": "USD"}}
]
```

This is an array of key-value pairs. To extract a specific value you need to use `UNNEST` — we will cover this in the staging module. For now, understand why this makes raw GA4 data impossible to use directly in BI tools.

---

## Step 5: Explore the Items Array

For purchase and product events, there is also an `items` array containing the products involved.

```sql
SELECT
  event_name,
  event_timestamp,
  items
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
WHERE event_name = 'purchase'
  AND _TABLE_SUFFIX = '20201101'
LIMIT 5
```

Each row in the `items` array contains fields like `item_id`, `item_name`, `item_category`, `price`, `quantity`.

---

## Step 6: Understand the Purchase Funnel in Numbers

This query shows how many unique users reach each stage of the funnel. Keep these numbers — they will be the baseline for your funnel dashboard.

```sql
SELECT
  COUNTIF(event_name = 'view_item')      AS viewed_product,
  COUNTIF(event_name = 'add_to_cart')    AS added_to_cart,
  COUNTIF(event_name = 'begin_checkout') AS began_checkout,
  COUNTIF(event_name = 'purchase')       AS purchased
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
```

> Note the dramatic drop-off at each step. This is normal for ecommerce — and it is exactly what the funnel dashboard will visualise.

---

## Step 7: Traffic Sources

```sql
SELECT
  traffic_source.source,
  traffic_source.medium,
  COUNT(DISTINCT user_pseudo_id) AS users,
  COUNTIF(event_name = 'purchase') AS purchases
FROM `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
GROUP BY 1, 2
ORDER BY users DESC
LIMIT 20
```

This shows which channels are driving traffic. Notice that `traffic_source` here is a user-level attribute — it records where the user first came from, not the source of each individual session.

---

## Step 8: The Reality Check

By now you should have noticed:

1. **The data is rich but inaccessible** — there is a lot of valuable information but it is buried in nested arrays
2. **There is no session table** — sessions have to be reconstructed from events
3. **There is no product table** — product data lives inside the `items` array on individual events
4. **Some fields have placeholder values** — this is obfuscation; certain user IDs and values will be NULL or `<Other>`

These are not problems. They are engineering challenges that the dbt layer will solve.

---

## Exploration Summary

Write down your answers before moving on. You will reference these in the BEAM workshop.

| Question | Your Answer |
|---|---|
| How many total events are in the dataset? | |
| Which event represents a completed purchase? | |
| What nested array holds event parameters? | |
| What nested array holds product data? | |
| What are the 5 funnel events in order? | |
| Which traffic source sends the most users? | |

When you are ready, move on to [02_beam_workshop.md](02_beam_workshop.md).
