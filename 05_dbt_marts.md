# Module 05: Intermediate Models and Marts

## The Three-Layer Architecture

Before writing any SQL, understand what each layer is for:

| Layer | Purpose | Reads From | Materialised As |
|---|---|---|---|
| Staging | Clean and flatten source data | Raw source only | Views |
| Intermediate | Business logic, sessionisation, joins | Staging only | Views |
| Marts | Business-ready tables for BI and Vanna | Intermediate + Staging | Tables |

The rule is strict: **marts never read directly from staging**. Staging → Intermediate → Marts. This keeps logic in the right layer and makes debugging straightforward.

---

## Part 1: Intermediate Models

### Model 1: int_ga4__sessions.sql

This model reconstructs sessions from events. GA4 does not have a sessions table — sessions are derived.

Create `models/intermediate/int_ga4__sessions.sql`:

```sql
{{
  config(
    materialized = 'view',
    description = 'Reconstructed sessions from GA4 events. One row per session.'
  )
}}

WITH events AS (

  SELECT * FROM {{ ref('stg_ga4__events') }}

),

session_params AS (

  SELECT
    event_id,
    MAX(CASE WHEN key = 'ga_session_number' THEN CAST(value AS INT64) END) AS session_number,
    MAX(CASE WHEN key = 'session_engaged'   THEN CAST(value AS INT64) END) AS session_engaged,
    MAX(CASE WHEN key = 'engagement_time_msec' THEN CAST(value AS INT64) END) AS engagement_time_msec
  FROM {{ ref('stg_ga4__event_params') }}
  WHERE key IN ('ga_session_number', 'session_engaged', 'engagement_time_msec')
  GROUP BY event_id

),

session_events AS (

  SELECT
    e.session_id,
    e.user_pseudo_id,
    e.device_category,
    e.device_operating_system,
    e.device_browser,
    e.geo_country,
    e.traffic_source,
    e.traffic_medium,
    e.traffic_campaign,
    e.event_name,
    e.event_timestamp_utc,
    e.event_date,
    p.session_number,
    p.session_engaged,
    p.engagement_time_msec
  FROM events e
  LEFT JOIN session_params p ON e.event_id = p.event_id

),

sessions AS (

  SELECT
    session_id,
    user_pseudo_id,

    -- Session timing
    MIN(event_timestamp_utc)  AS session_start_at,
    MAX(event_timestamp_utc)  AS session_end_at,
    MIN(event_date)           AS session_date,

    -- Session attributes (use first non-null value)
    MAX(device_category)      AS device_category,
    MAX(device_operating_system) AS device_operating_system,
    MAX(device_browser)       AS device_browser,
    MAX(geo_country)          AS geo_country,
    MAX(traffic_source)       AS traffic_source,
    MAX(traffic_medium)       AS traffic_medium,
    MAX(traffic_campaign)     AS traffic_campaign,
    MAX(session_number)       AS session_number,

    -- Session metrics
    COUNT(*)                  AS event_count,
    COUNTIF(event_name = 'page_view')     AS page_view_count,
    COUNTIF(event_name = 'view_item')     AS product_view_count,
    COUNTIF(event_name = 'add_to_cart')   AS add_to_cart_count,
    COUNTIF(event_name = 'begin_checkout') AS checkout_count,
    COUNTIF(event_name = 'purchase')      AS purchase_count,

    -- Conversion flag
    MAX(CASE WHEN event_name = 'purchase' THEN 1 ELSE 0 END) AS converted,

    -- Engagement
    MAX(session_engaged)        AS session_engaged,
    SUM(engagement_time_msec)   AS total_engagement_ms

  FROM session_events
  GROUP BY session_id, user_pseudo_id

)

SELECT
  *,
  TIMESTAMP_DIFF(session_end_at, session_start_at, SECOND) AS session_duration_seconds
FROM sessions
```

---

### Model 2: int_ga4__funnel_steps.sql

This model tags each user-session combination with the furthest funnel step they reached.

Create `models/intermediate/int_ga4__funnel_steps.sql`:

```sql
{{
  config(
    materialized = 'view',
    description = 'Funnel step progression per user per session. Used for drop-off analysis.'
  )
}}

WITH events AS (

  SELECT
    session_id,
    user_pseudo_id,
    event_name,
    event_date,
    device_category,
    traffic_source,
    traffic_medium
  FROM {{ ref('stg_ga4__events') }}
  WHERE event_name IN ('view_item', 'add_to_cart', 'begin_checkout', 'purchase')

),

funnel_step_mapping AS (

  SELECT
    session_id,
    user_pseudo_id,
    event_date,
    device_category,
    traffic_source,
    traffic_medium,
    event_name,
    CASE event_name
      WHEN 'view_item'      THEN 1
      WHEN 'add_to_cart'    THEN 2
      WHEN 'begin_checkout' THEN 3
      WHEN 'purchase'       THEN 4
    END AS funnel_step
  FROM events

),

session_max_step AS (

  SELECT
    session_id,
    user_pseudo_id,
    event_date,
    device_category,
    traffic_source,
    traffic_medium,
    MAX(funnel_step) AS max_funnel_step_reached,
    CASE MAX(funnel_step)
      WHEN 1 THEN 'Viewed product'
      WHEN 2 THEN 'Added to cart'
      WHEN 3 THEN 'Began checkout'
      WHEN 4 THEN 'Purchased'
    END AS furthest_step_name,
    -- Flags for each step reached
    MAX(CASE WHEN funnel_step >= 1 THEN 1 ELSE 0 END) AS reached_view_item,
    MAX(CASE WHEN funnel_step >= 2 THEN 1 ELSE 0 END) AS reached_add_to_cart,
    MAX(CASE WHEN funnel_step >= 3 THEN 1 ELSE 0 END) AS reached_checkout,
    MAX(CASE WHEN funnel_step >= 4 THEN 1 ELSE 0 END) AS reached_purchase
  FROM funnel_step_mapping
  GROUP BY 1, 2, 3, 4, 5, 6

)

SELECT * FROM session_max_step
```

---

## Part 2: Mart Models

### Create the Marts Documentation File

Create `models/marts/_marts.yml`:

```yaml
version: 2

models:
  - name: fct_purchases
    description: >
      One row per completed purchase transaction. This is the primary fact table
      for revenue analysis. Join to dim_products on item_id for product details.
      Join to dim_users on user_pseudo_id for user attributes.
      Answers business question: Which channels drive revenue? (Q1 and Q3)
    columns:
      - name: purchase_id
        description: "Surrogate key for the purchase event"
        tests:
          - unique
          - not_null
      - name: transaction_id
        description: "GA4 transaction identifier"
      - name: user_pseudo_id
        description: "User who made the purchase"
        tests:
          - not_null
      - name: session_id
        description: "Session in which the purchase occurred"
      - name: purchase_date
        description: "Date of the purchase"
      - name: revenue
        description: "Total revenue from this transaction in USD"
      - name: item_count
        description: "Number of distinct items in the transaction"
      - name: total_quantity
        description: "Total units purchased across all items"
      - name: device_category
        description: "Device used to make the purchase (desktop, mobile, tablet)"
      - name: traffic_source
        description: "Traffic source of the user who purchased"
      - name: traffic_medium
        description: "Traffic medium of the user who purchased"

  - name: fct_product_interactions
    description: >
      One row per product-level event. Covers view_item, add_to_cart, begin_checkout
      and purchase events where an item is involved. Use this table for funnel analysis
      and product performance. Answers business question: Which products convert? (Q2 and Q3)
    columns:
      - name: interaction_id
        description: "Surrogate key"
        tests:
          - unique
          - not_null
      - name: event_name
        description: "The type of interaction: view_item, add_to_cart, begin_checkout, purchase"
      - name: item_id
        description: "Product identifier"
      - name: item_name
        description: "Product name"
      - name: item_category
        description: "Product category"
      - name: user_pseudo_id
        description: "User who performed the interaction"
      - name: session_id
        description: "Session in which the interaction occurred"
      - name: interaction_date
        description: "Date of the interaction"
      - name: device_category
        description: "Device used"
      - name: revenue
        description: "Revenue from this item (only populated for purchase events)"

  - name: fct_sessions
    description: >
      One row per session. A session is a group of events from one user within
      a 30-minute window. Use this table for traffic analysis and conversion rates.
      Answers business question: Which channels convert sessions to purchases? (Q1)
    columns:
      - name: session_id
        description: "Unique session identifier"
        tests:
          - unique
          - not_null
      - name: user_pseudo_id
        description: "User the session belongs to"
      - name: session_date
        description: "Date the session started"
      - name: session_start_at
        description: "Timestamp when the session started"
      - name: session_duration_seconds
        description: "Length of the session in seconds"
      - name: page_view_count
        description: "Number of pages viewed in the session"
      - name: converted
        description: "1 if the session resulted in a purchase, 0 otherwise"
      - name: device_category
        description: "Device used in the session"
      - name: traffic_source
        description: "Where the session originated"
      - name: traffic_medium
        description: "Medium of the traffic source"
```

---

### Mart Model 1: fct_purchases.sql

Create `models/marts/fct_purchases.sql`:

```sql
{{
  config(
    materialized = 'table',
    partition_by = {
      'field': 'purchase_date',
      'data_type': 'date',
      'granularity': 'day'
    },
    cluster_by = ['traffic_source', 'device_category'],
    description = 'One row per purchase transaction.'
  )
}}

WITH purchase_events AS (

  SELECT * FROM {{ ref('stg_ga4__events') }}
  WHERE event_name = 'purchase'

),

purchase_items AS (

  SELECT
    event_id,
    COUNT(DISTINCT item_id) AS item_count,
    SUM(quantity)           AS total_quantity
  FROM {{ ref('stg_ga4__items') }}
  WHERE event_name = 'purchase'
  GROUP BY event_id

)

SELECT
  {{ dbt_utils.generate_surrogate_key(['e.event_id']) }} AS purchase_id,
  e.transaction_id,
  e.user_pseudo_id,
  e.session_id,
  e.event_date                       AS purchase_date,
  e.event_timestamp_utc              AS purchased_at,
  COALESCE(e.purchase_revenue, 0)    AS revenue,
  COALESCE(i.item_count, 0)          AS item_count,
  COALESCE(i.total_quantity, 0)      AS total_quantity,
  e.device_category,
  e.device_operating_system,
  e.geo_country,
  e.traffic_source,
  e.traffic_medium,
  e.traffic_campaign
FROM purchase_events e
LEFT JOIN purchase_items i ON e.event_id = i.event_id
```

---

### Mart Model 2: fct_product_interactions.sql

Create `models/marts/fct_product_interactions.sql`:

```sql
{{
  config(
    materialized = 'table',
    partition_by = {
      'field': 'interaction_date',
      'data_type': 'date',
      'granularity': 'day'
    },
    cluster_by = ['event_name', 'item_category'],
    description = 'One row per product-level interaction event.'
  )
}}

WITH items AS (

  SELECT * FROM {{ ref('stg_ga4__items') }}

),

events AS (

  SELECT
    event_id,
    user_pseudo_id,
    session_id,
    event_date,
    device_category,
    traffic_source,
    traffic_medium
  FROM {{ ref('stg_ga4__events') }}
  WHERE event_name IN ('view_item', 'add_to_cart', 'begin_checkout', 'purchase')

)

SELECT
  {{ dbt_utils.generate_surrogate_key(['i.event_id', 'i.item_id']) }} AS interaction_id,
  i.event_name,
  CASE i.event_name
    WHEN 'view_item'      THEN 1
    WHEN 'add_to_cart'    THEN 2
    WHEN 'begin_checkout' THEN 3
    WHEN 'purchase'       THEN 4
  END                                AS funnel_step,
  i.item_id,
  i.item_name,
  i.item_brand,
  i.item_category,
  i.item_category_2,
  i.item_variant,
  i.price,
  i.quantity,
  CASE WHEN i.event_name = 'purchase' THEN i.item_revenue ELSE NULL END AS revenue,
  e.user_pseudo_id,
  e.session_id,
  e.event_date                       AS interaction_date,
  e.device_category,
  e.traffic_source,
  e.traffic_medium
FROM items i
INNER JOIN events e ON i.event_id = e.event_id
```

---

### Mart Model 3: fct_sessions.sql

Create `models/marts/fct_sessions.sql`:

```sql
{{
  config(
    materialized = 'table',
    partition_by = {
      'field': 'session_date',
      'data_type': 'date',
      'granularity': 'day'
    },
    cluster_by = ['traffic_source', 'device_category'],
    description = 'One row per session. Includes conversion flag and engagement metrics.'
  )
}}

SELECT
  session_id,
  user_pseudo_id,
  session_date,
  session_start_at,
  session_end_at,
  session_duration_seconds,
  session_number,
  session_engaged,
  event_count,
  page_view_count,
  product_view_count,
  add_to_cart_count,
  checkout_count,
  purchase_count,
  converted,
  device_category,
  device_operating_system,
  device_browser,
  geo_country,
  traffic_source,
  traffic_medium,
  traffic_campaign,
  total_engagement_ms
FROM {{ ref('int_ga4__sessions') }}
```

---

## Step 3: Run the Full dbt Project

### Development Run (one week of data)
```bash
dbt run
```

### Check the lineage
```bash
dbt docs generate
dbt docs serve
```

This opens a browser with the full lineage graph. You should see:
```
source:ga4_ecommerce.events
  └── stg_ga4__events
  └── stg_ga4__event_params
  └── stg_ga4__items
        └── int_ga4__sessions
        └── int_ga4__funnel_steps
              └── fct_purchases
              └── fct_product_interactions
              └── fct_sessions
```

### Run all tests
```bash
dbt test
```

### Production Run (full 3 months of data)
```bash
dbt run --target prod
```

> **Do this only once before connecting Tableau.** The prod run scans the full dataset and uses more of your BigQuery quota.

---

## What You Have Built

- [ ] `int_ga4__sessions` — reconstructed sessions with engagement metrics
- [ ] `int_ga4__funnel_steps` — funnel progression per user per session
- [ ] `fct_purchases` — one row per transaction, partitioned by date
- [ ] `fct_product_interactions` — one row per product event, partitioned by date
- [ ] `fct_sessions` — one row per session, partitioned by date
- [ ] Full documentation in `_marts.yml`
- [ ] dbt lineage graph confirmed
- [ ] All tests passing

When you are ready, move on to [06_tableau.md](06_tableau.md).
