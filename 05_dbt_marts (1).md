# Module 05: Intermediate Models, Fact Tables, and Dimension Tables

## The Three-Layer Rule

| Layer | Job | Reads from | Written as |
|---|---|---|---|
| Staging | Flatten and rename source fields | Raw source only | Views |
| Intermediate | Business logic, joins, reconstruction | Staging only | Views |
| Marts | Business-ready tables for BI and Vanna | Intermediate only | Tables |

Marts never read from staging directly. Every model in the mart layer reads from an intermediate model.

---

## What Gets Built in This Module

**Three intermediate models:**

| Model | Job |
|---|---|
| `int_ga4__product_events.sql` | Joins events to items, assigns funnel steps. Feeds `fct_product_interactions` and `fct_purchases`. |
| `int_ga4__session_traffic.sql` | Extracts session-level traffic source from `event_params` on `session_start` events. |
| `int_ga4__sessions.sql` | Reconstructs sessions from all events. Joins `int_ga4__session_traffic`. Feeds `fct_sessions`. |

**Three fact tables:**

| Model | Grain |
|---|---|
| `fct_product_interactions.sql` | One row per item per product event |
| `fct_purchases.sql` | One row per transaction |
| `fct_sessions.sql` | One row per session |

**Five dimension tables:**

| Model | Grain |
|---|---|
| `dim_users.sql` | One row per `user_pseudo_id` |
| `dim_products.sql` | One row per `item_id` |
| `dim_date.sql` | One row per calendar date (reads from seed) |
| `dim_device.sql` | One row per device/OS/browser combination |
| `dim_traffic_source.sql` | One row per source/medium combination |

**The complete lineage:**

```
source: ga4_ecommerce.events
  ├── stg_ga4__events ──────────────────────────────────────────────────────┐
  ├── stg_ga4__event_params ────────────────────────────────────────────────┤
  └── stg_ga4__items                                                        │
                                                                            │
stg_ga4__events + stg_ga4__items                                            │
  └── int_ga4__product_events ─────────── fct_product_interactions          │
                               └───────── fct_purchases                     │
                                                                            │
stg_ga4__events + stg_ga4__event_params ────────────────────────────────────┤
  ├── int_ga4__session_traffic ──────────────────────────────────────────────┤
  └── int_ga4__sessions (joins int_ga4__session_traffic) ──── fct_sessions  │
                                                                            │
stg_ga4__events ──────────────────── dim_users                              │
stg_ga4__items ───────────────────── dim_products                           │
seed: dim_date.csv ───────────────── dim_date                               │
stg_ga4__events ──────────────────── dim_device                             │
stg_ga4__events ──────────────────── dim_traffic_source ────────────────────┘
```

---

## Part 1: Intermediate Layer Documentation

Create `models/intermediate/_intermediate.yml`:

```yaml
version: 2

models:

  - name: int_ga4__product_events
    description: >
      One row per item per product event. Joins stg_ga4__events to stg_ga4__items
      on event_id. Filters to the four funnel events: view_item, add_to_cart,
      begin_checkout, purchase. Assigns a funnel_step integer to each event type.
      This model feeds both fct_product_interactions and fct_purchases so that
      the item-to-event join logic is written once and not duplicated across two
      mart models. item_revenue is NULL for non-purchase funnel steps by design.
    columns:
      - name: event_id
        description: "FK to stg_ga4__events."
        tests:
          - not_null
      - name: item_id
        description: "Product SKU from items[].item_id."
        tests:
          - not_null
      - name: event_name
        description: "GA4 event name: view_item, add_to_cart, begin_checkout, purchase."
        tests:
          - not_null
          - accepted_values:
              values: ['view_item', 'add_to_cart', 'begin_checkout', 'purchase']
      - name: funnel_step
        description: "Integer funnel position: 1=view_item, 2=add_to_cart, 3=begin_checkout, 4=purchase."
        tests:
          - not_null
      - name: user_pseudo_id
        description: "Pseudonymous user identifier from the parent event."
        tests:
          - not_null
      - name: session_id
        description: "Constructed session identifier."
      - name: event_date
        description: "Date of the event as a DATE type."
      - name: event_timestamp_utc
        description: "Exact event timestamp as UTC TIMESTAMP."
      - name: item_name
        description: "Product display name from items[].item_name."
      - name: item_brand
        description: "Product brand from items[].item_brand."
      - name: item_category
        description: "Primary product category from items[].item_category."
      - name: item_category_2
        description: "Secondary product category from items[].item_category2."
      - name: item_variant
        description: "Product variant such as size or colour."
      - name: price
        description: "Unit price at time of this event from items[].price."
      - name: quantity
        description: "Units in this event from items[].quantity. Typically 1 for view_item."
      - name: item_revenue
        description: >
          Revenue from this item (items[].item_revenue). NULL for view_item,
          add_to_cart, begin_checkout. Populated only on purchase events.
          This is correct by design.
      - name: transaction_id
        description: "GA4 transaction ID from ecommerce struct. Populated only on purchase events."
      - name: purchase_revenue
        description: "Total transaction revenue. Populated only on purchase events."
      - name: purchase_revenue_usd
        description: "Total transaction revenue in USD. Populated only on purchase events."
      - name: unique_items
        description: "Distinct product count for the transaction. Populated only on purchase events."
      - name: total_item_quantity
        description: "Total units across all items. Populated only on purchase events."
      - name: device_category
        description: "Device category from parent event: desktop, mobile, tablet."
      - name: device_operating_system
        description: "Operating system from parent event."
      - name: geo_country
        description: "Country from parent event geo record."
      - name: traffic_source
        description: "User-level first-touch traffic source."
      - name: traffic_medium
        description: "User-level first-touch traffic medium."
      - name: traffic_campaign
        description: "User-level first-touch campaign name."

  - name: int_ga4__session_traffic
    description: >
      One row per session containing session-level traffic attribution extracted
      from event_params on session_start events. This is distinct from the
      user-level traffic_source field which records first-touch acquisition only.
      Session-level traffic reflects where each specific session originated,
      which may differ from the user's original acquisition source.
    columns:
      - name: session_id
        description: "Session identifier. Primary key."
        tests:
          - unique
          - not_null
      - name: session_source
        description: "Traffic source for this session from event_params[key='source'] on session_start."
      - name: session_medium
        description: "Traffic medium for this session from event_params[key='medium'] on session_start."
      - name: session_campaign
        description: "Campaign for this session from event_params[key='campaign'] on session_start."

  - name: int_ga4__sessions
    description: >
      One row per reconstructed session. GA4 does not export a sessions table.
      Sessions are derived by grouping all events sharing the same session key
      (user_pseudo_id + ga_session_id from event_params). Aggregates event counts,
      engagement metrics, and the conversion flag per session. Joins
      int_ga4__session_traffic to add session-level traffic attribution.
      This model feeds fct_sessions directly.
    columns:
      - name: session_id
        description: "Natural key: CONCAT(user_pseudo_id, '_', CAST(ga_session_id AS STRING))."
        tests:
          - unique
          - not_null
      - name: user_pseudo_id
        description: "User who owns this session."
        tests:
          - not_null
      - name: session_date
        description: "Date of the first event in this session."
      - name: session_start_at
        description: "Timestamp of the first event in this session."
      - name: session_end_at
        description: "Timestamp of the last event in this session."
      - name: session_duration_seconds
        description: "Seconds between first and last event in session."
      - name: session_number
        description: "Which session this is for the user. 1 = first ever session."
      - name: session_engaged
        description: "1 if GA4 considers this an engaged session, 0 if bounce."
      - name: total_engagement_ms
        description: "Sum of engagement_time_msec across all events in the session."
      - name: event_count
        description: "Total events fired in this session."
      - name: page_view_count
        description: "Number of page_view events in this session."
      - name: product_view_count
        description: "Number of view_item events in this session."
      - name: add_to_cart_count
        description: "Number of add_to_cart events in this session."
      - name: checkout_count
        description: "Number of begin_checkout events in this session."
      - name: purchase_count
        description: "Number of purchase events in this session."
      - name: converted
        description: "1 if purchase_count > 0, 0 otherwise."
      - name: device_category
        description: "Device category used in this session."
      - name: device_operating_system
        description: "Operating system used in this session."
      - name: device_browser
        description: "Browser used in this session."
      - name: geo_country
        description: "Country of the user in this session."
      - name: session_source
        description: "Session-level traffic source from int_ga4__session_traffic."
      - name: session_medium
        description: "Session-level traffic medium."
      - name: session_campaign
        description: "Session-level campaign name."
      - name: traffic_source
        description: "User-level first-touch traffic source. Set at first visit, never changes."
      - name: traffic_medium
        description: "User-level first-touch traffic medium."
      - name: traffic_campaign
        description: "User-level first-touch campaign name."
```

---

## Part 2: Intermediate Model SQL

### int_ga4__product_events.sql

Create `models/intermediate/int_ga4__product_events.sql`:

```sql
{{
  config(
    materialized = 'view',
    description = 'One row per item per product event. Joins events to items. Feeds fct_product_interactions and fct_purchases.'
  )
}}

WITH events AS (

  SELECT
    event_id,
    event_name,
    user_pseudo_id,
    session_id,
    event_date,
    event_timestamp_utc,
    device_category,
    device_operating_system,
    geo_country,
    traffic_source,
    traffic_medium,
    traffic_campaign,
    transaction_id,
    purchase_revenue,
    purchase_revenue_usd,
    unique_items
  FROM {{ ref('stg_ga4__events') }}
  WHERE event_name IN ('view_item', 'add_to_cart', 'begin_checkout', 'purchase')

),

items AS (

  SELECT
    event_id,
    event_name,
    item_id,
    item_name,
    item_brand,
    item_category,
    item_category_2,
    item_variant,
    price,
    quantity,
    item_revenue
  FROM {{ ref('stg_ga4__items') }}

)

SELECT
  e.event_id,
  i.item_id,
  e.event_name,
  CASE e.event_name
    WHEN 'view_item'      THEN 1
    WHEN 'add_to_cart'    THEN 2
    WHEN 'begin_checkout' THEN 3
    WHEN 'purchase'       THEN 4
  END                                   AS funnel_step,
  e.user_pseudo_id,
  e.session_id,
  e.event_date,
  e.event_timestamp_utc,
  i.item_name,
  i.item_brand,
  i.item_category,
  i.item_category_2,
  i.item_variant,
  i.price,
  i.quantity,
  CASE WHEN e.event_name = 'purchase'
    THEN i.item_revenue
    ELSE NULL
  END                                   AS item_revenue,
  e.transaction_id,
  e.purchase_revenue,
  e.purchase_revenue_usd,
  e.unique_items,
  e.device_category,
  e.device_operating_system,
  e.geo_country,
  e.traffic_source,
  e.traffic_medium,
  e.traffic_campaign
FROM events e
INNER JOIN items i ON e.event_id = i.event_id
```

---

### int_ga4__session_traffic.sql

Create `models/intermediate/int_ga4__session_traffic.sql`:

```sql
{{
  config(
    materialized = 'view',
    description = 'Session-level traffic attribution from event_params on session_start events. One row per session.'
  )
}}

WITH session_start_events AS (

  SELECT
    session_id,
    event_id
  FROM {{ ref('stg_ga4__events') }}
  WHERE event_name = 'session_start'

),

session_params AS (

  SELECT
    p.event_id,
    MAX(CASE WHEN p.key = 'source'   THEN p.value_string END) AS session_source,
    MAX(CASE WHEN p.key = 'medium'   THEN p.value_string END) AS session_medium,
    MAX(CASE WHEN p.key = 'campaign' THEN p.value_string END) AS session_campaign
  FROM {{ ref('stg_ga4__event_params') }} p
  WHERE p.key IN ('source', 'medium', 'campaign')
  GROUP BY p.event_id

)

SELECT
  s.session_id,
  COALESCE(p.session_source,   '(direct)') AS session_source,
  COALESCE(p.session_medium,   '(none)')   AS session_medium,
  COALESCE(p.session_campaign, '(none)')   AS session_campaign
FROM session_start_events s
LEFT JOIN session_params p ON s.event_id = p.event_id
```

---

### int_ga4__sessions.sql

Create `models/intermediate/int_ga4__sessions.sql`:

```sql
{{
  config(
    materialized = 'view',
    description = 'One row per reconstructed session. Aggregates event counts and engagement. Joins session-level traffic attribution.'
  )
}}

WITH events AS (

  SELECT
    session_id,
    user_pseudo_id,
    event_id,
    event_name,
    event_date,
    event_timestamp_utc,
    device_category,
    device_operating_system,
    device_browser,
    geo_country,
    traffic_source,
    traffic_medium,
    traffic_campaign
  FROM {{ ref('stg_ga4__events') }}

),

session_level_params AS (

  SELECT
    e.event_id,
    MAX(CASE WHEN p.key = 'ga_session_number'    THEN CAST(p.value_int AS INT64) END) AS session_number,
    MAX(CASE WHEN p.key = 'session_engaged'      THEN CAST(p.value_int AS INT64) END) AS session_engaged,
    MAX(CASE WHEN p.key = 'engagement_time_msec' THEN CAST(p.value_int AS INT64) END) AS engagement_time_msec
  FROM {{ ref('stg_ga4__events') }} e
  LEFT JOIN {{ ref('stg_ga4__event_params') }} p
    ON e.event_id = p.event_id
   AND p.key IN ('ga_session_number', 'session_engaged', 'engagement_time_msec')
  GROUP BY e.event_id

),

events_with_params AS (

  SELECT
    e.session_id,
    e.user_pseudo_id,
    e.event_name,
    e.event_date,
    e.event_timestamp_utc,
    e.device_category,
    e.device_operating_system,
    e.device_browser,
    e.geo_country,
    e.traffic_source,
    e.traffic_medium,
    e.traffic_campaign,
    p.session_number,
    p.session_engaged,
    p.engagement_time_msec
  FROM events e
  LEFT JOIN session_level_params p ON e.event_id = p.event_id

),

aggregated AS (

  SELECT
    session_id,
    user_pseudo_id,
    MIN(event_date)                                             AS session_date,
    MIN(event_timestamp_utc)                                    AS session_start_at,
    MAX(event_timestamp_utc)                                    AS session_end_at,
    MAX(device_category)                                        AS device_category,
    MAX(device_operating_system)                                AS device_operating_system,
    MAX(device_browser)                                         AS device_browser,
    MAX(geo_country)                                            AS geo_country,
    MAX(traffic_source)                                         AS traffic_source,
    MAX(traffic_medium)                                         AS traffic_medium,
    MAX(traffic_campaign)                                       AS traffic_campaign,
    MAX(session_number)                                         AS session_number,
    MAX(session_engaged)                                        AS session_engaged,
    SUM(COALESCE(engagement_time_msec, 0))                      AS total_engagement_ms,
    COUNT(*)                                                    AS event_count,
    COUNTIF(event_name = 'page_view')                           AS page_view_count,
    COUNTIF(event_name = 'view_item')                           AS product_view_count,
    COUNTIF(event_name = 'add_to_cart')                         AS add_to_cart_count,
    COUNTIF(event_name = 'begin_checkout')                      AS checkout_count,
    COUNTIF(event_name = 'purchase')                            AS purchase_count,
    MAX(CASE WHEN event_name = 'purchase' THEN 1 ELSE 0 END)    AS converted
  FROM events_with_params
  GROUP BY session_id, user_pseudo_id

),

with_duration AS (

  SELECT
    *,
    TIMESTAMP_DIFF(session_end_at, session_start_at, SECOND) AS session_duration_seconds
  FROM aggregated

)

SELECT
  s.*,
  COALESCE(t.session_source,   s.traffic_source)   AS session_source,
  COALESCE(t.session_medium,   s.traffic_medium)   AS session_medium,
  COALESCE(t.session_campaign, s.traffic_campaign) AS session_campaign
FROM with_duration s
LEFT JOIN {{ ref('int_ga4__session_traffic') }} t
  ON s.session_id = t.session_id
```

---

## Part 3: Mart Layer Documentation

Create `models/marts/_marts.yml`:

```yaml
version: 2

models:

  # ─── FACT TABLES ────────────────────────────────────────────────────────────

  - name: fct_product_interactions
    description: >
      One row per item per product event. Covers view_item (funnel_step=1),
      add_to_cart (funnel_step=2), begin_checkout (funnel_step=3), and
      purchase (funnel_step=4). Use for funnel drop-off analysis and
      product-level performance. revenue is NULL for non-purchase rows
      by design. Reads from int_ga4__product_events.
      Answers business questions Q2 (funnel) and Q3 (products).
    columns:
      - name: interaction_id
        description: "Surrogate key from event_id and item_id."
        tests:
          - unique
          - not_null
      - name: event_name
        description: "GA4 event type: view_item, add_to_cart, begin_checkout, purchase."
        tests:
          - not_null
          - accepted_values:
              values: ['view_item', 'add_to_cart', 'begin_checkout', 'purchase']
      - name: funnel_step
        description: "Integer funnel position: 1=view_item 2=add_to_cart 3=begin_checkout 4=purchase."
        tests:
          - not_null
      - name: item_id
        description: "Product SKU. FK to dim_products."
        tests:
          - not_null
      - name: item_name
        description: "Product display name. Denormalised for query convenience."
      - name: item_category
        description: "Primary product category."
      - name: item_brand
        description: "Product brand."
      - name: item_variant
        description: "Product variant such as size or colour."
      - name: price
        description: "Unit price at time of this event."
      - name: quantity
        description: "Units. Typically 1 for view_item, variable for add_to_cart and purchase."
      - name: revenue
        description: >
          Item revenue (price x quantity). NULL for view_item, add_to_cart,
          begin_checkout. Populated only for purchase events. This is correct.
      - name: user_pseudo_id
        description: "Pseudonymous user identifier. FK to dim_users."
        tests:
          - not_null
      - name: session_id
        description: "Session identifier. FK to fct_sessions."
      - name: interaction_date
        description: "Date of the interaction. Partition key. FK to dim_date."
        tests:
          - not_null
      - name: interacted_at
        description: "Exact UTC timestamp."
      - name: device_category
        description: "Device used: desktop, mobile, tablet. FK to dim_device."
      - name: device_operating_system
        description: "Operating system."
      - name: geo_country
        description: "Country of the user."
      - name: traffic_source
        description: "User-level first-touch traffic source. FK to dim_traffic_source."
      - name: traffic_medium
        description: "User-level first-touch traffic medium."
      - name: traffic_campaign
        description: "User-level first-touch campaign name."

  - name: fct_purchases
    description: >
      One row per completed purchase transaction. Grain is the transaction,
      not the individual item. item_count and total_quantity come from the
      ecommerce struct fields on the GA4 event which are already transaction-level
      aggregates. For item-level purchase detail, use fct_product_interactions
      filtered to event_name = 'purchase'. Reads from int_ga4__product_events.
      Answers business questions Q1 (acquisition) and Q3 (products).
    columns:
      - name: purchase_id
        description: "Surrogate key from event_id."
        tests:
          - unique
          - not_null
      - name: transaction_id
        description: "GA4 transaction identifier. Natural key."
      - name: user_pseudo_id
        description: "User who completed the purchase. FK to dim_users."
        tests:
          - not_null
      - name: session_id
        description: "Session in which the purchase occurred. FK to fct_sessions."
      - name: purchase_date
        description: "Date of the purchase. Partition key. FK to dim_date."
        tests:
          - not_null
      - name: purchased_at
        description: "Exact UTC timestamp of the purchase."
      - name: revenue
        description: "Total transaction revenue in native currency from ecommerce.purchase_revenue."
      - name: revenue_usd
        description: "Total transaction revenue in USD from ecommerce.purchase_revenue_in_usd."
      - name: item_count
        description: "Distinct product SKU count from ecommerce.unique_items."
      - name: total_quantity
        description: "Total units across all items from ecommerce.total_item_quantity."
      - name: device_category
        description: "Device used. FK to dim_device."
      - name: device_operating_system
        description: "Operating system used."
      - name: geo_country
        description: "Country of the user."
      - name: traffic_source
        description: "User-level first-touch traffic source. FK to dim_traffic_source."
      - name: traffic_medium
        description: "User-level first-touch traffic medium."
      - name: traffic_campaign
        description: "User-level first-touch campaign name."

  - name: fct_sessions
    description: >
      One row per session. Sessions are reconstructed from GA4 events as GA4
      does not export a sessions table. A session is the unique combination of
      user_pseudo_id and ga_session_id. Contains event counts, engagement metrics,
      and the conversion flag. session_source and session_medium are session-level
      attribution from event_params. traffic_source and traffic_medium are
      user-level first-touch attribution. Use session_source for channel analysis.
      Use traffic_source for acquisition cohort analysis. Reads from int_ga4__sessions.
      Answers business question Q1 (acquisition and conversion by channel).
    columns:
      - name: session_id
        description: "Natural key: CONCAT(user_pseudo_id, '_', ga_session_id)."
        tests:
          - unique
          - not_null
      - name: user_pseudo_id
        description: "User who owns this session. FK to dim_users."
        tests:
          - not_null
      - name: session_date
        description: "Date the session started. Partition key. FK to dim_date."
        tests:
          - not_null
      - name: session_start_at
        description: "Timestamp of the first event in the session."
      - name: session_end_at
        description: "Timestamp of the last event in the session."
      - name: session_duration_seconds
        description: "Seconds between first and last event."
      - name: session_number
        description: "Which session this is for the user. 1 = first ever session."
      - name: session_engaged
        description: "1 if GA4 considers this an engaged session, 0 if bounce."
      - name: total_engagement_ms
        description: "Sum of engagement_time_msec across all events in the session."
      - name: event_count
        description: "Total events fired in this session."
      - name: page_view_count
        description: "page_view events in this session."
      - name: product_view_count
        description: "view_item events in this session."
      - name: add_to_cart_count
        description: "add_to_cart events in this session."
      - name: checkout_count
        description: "begin_checkout events in this session."
      - name: purchase_count
        description: "purchase events in this session."
      - name: converted
        description: >
          1 if this session resulted in a purchase (purchase_count > 0), 0 otherwise.
          Conversion rate = SUM(converted) / COUNT(session_id).
      - name: device_category
        description: "Device category. FK to dim_device."
      - name: device_operating_system
        description: "Operating system."
      - name: device_browser
        description: "Browser."
      - name: geo_country
        description: "Country of the user."
      - name: session_source
        description: >
          Session-level traffic source from event_params on session_start.
          Use this for channel performance analysis. Reflects where this specific
          session originated. Different from traffic_source which is first-touch only.
      - name: session_medium
        description: "Session-level medium. Use for channel performance analysis."
      - name: session_campaign
        description: "Session-level campaign name."
      - name: traffic_source
        description: "User-level first-touch source. Use for acquisition cohort analysis."
      - name: traffic_medium
        description: "User-level first-touch medium."
      - name: traffic_campaign
        description: "User-level first-touch campaign."

  # ─── DIMENSION TABLES ───────────────────────────────────────────────────────

  - name: dim_users
    description: >
      One row per user_pseudo_id. Built from the first observed event per user
      using MIN(event_timestamp) to identify the earliest record.
      Contains first-touch acquisition attributes and GA4 lifetime value fields.
      Note: user_ltv fields may be NULL in the obfuscated sample dataset.
    columns:
      - name: user_pseudo_id
        description: "Pseudonymous user identifier. Primary key."
        tests:
          - unique
          - not_null
      - name: first_visit_date
        description: "Date of the user's first observed event in the dataset."
      - name: first_visit_at
        description: "Timestamp of the user's first observed event."
      - name: acquisition_source
        description: "First-touch traffic source from traffic_source.source."
      - name: acquisition_medium
        description: "First-touch traffic medium from traffic_source.medium."
      - name: acquisition_campaign
        description: "First-touch campaign name from traffic_source.name."
      - name: user_ltv_revenue
        description: "Lifetime revenue per GA4 from user_ltv.revenue. May be NULL in sample dataset."
      - name: user_ltv_currency
        description: "Currency code for LTV from user_ltv.currency. May be NULL in sample dataset."

  - name: dim_products
    description: >
      One row per item_id. Built by deduplicating item_id values seen across
      all product events (view_item, add_to_cart, begin_checkout, purchase)
      and taking the most recently observed non-null values for name and category.
      GA4 does not export a product catalogue so this dimension is derived from
      event-level item records. If a product name changes in GA4 over time this
      dimension reflects the most recent name seen in the dataset.
    columns:
      - name: item_id
        description: "Product SKU. Primary key."
        tests:
          - unique
          - not_null
      - name: item_name
        description: "Most recently observed product display name."
      - name: item_brand
        description: "Product brand."
      - name: item_category
        description: "Primary product category."
      - name: item_category_2
        description: "Secondary product category."
      - name: item_variant
        description: "Product variant such as size or colour."

  - name: dim_date
    description: >
      One row per calendar date covering the full dataset range 2020-11-01 to
      2021-01-31. Built from the dim_date seed CSV file — not derived from GA4
      event data. Used as a date spine for time-based analysis in all three
      fact tables via purchase_date, interaction_date, and session_date.
    columns:
      - name: date_day
        description: "The calendar date. Primary key."
        tests:
          - unique
          - not_null
      - name: year
        description: "Calendar year."
      - name: quarter
        description: "Calendar quarter: 1, 2, 3, or 4."
      - name: month
        description: "Calendar month number: 1 through 12."
      - name: month_name
        description: "Month name: January through December."
      - name: week_of_year
        description: "ISO week number."
      - name: day_of_week
        description: "ISO day of week: 1=Monday, 7=Sunday."
      - name: day_name
        description: "Day name: Monday through Sunday."
      - name: is_weekend
        description: "TRUE if Saturday or Sunday, FALSE otherwise."

  - name: dim_device
    description: >
      One row per unique combination of device_category, operating_system,
      and browser. Built by deduplicating these three fields across all events.
      Low-cardinality dimension — expect fewer than 50 rows in this dataset.
      device_key is a surrogate generated from the three fields.
    columns:
      - name: device_key
        description: "Surrogate key generated from device_category, operating_system, browser."
        tests:
          - unique
          - not_null
      - name: device_category
        description: "Device type: desktop, mobile, tablet."
        tests:
          - not_null
          - accepted_values:
              values: ['desktop', 'mobile', 'tablet']
      - name: operating_system
        description: "Operating system: Windows, iOS, Android, Macintosh, etc."
      - name: browser
        description: "Browser: Chrome, Safari, Firefox, etc."
      - name: mobile_brand_name
        description: "Mobile device brand such as Apple or Samsung. NULL for desktop."

  - name: dim_traffic_source
    description: >
      One row per unique source and medium combination. Built by deduplicating
      traffic_source.source and traffic_source.medium across all events.
      Includes a channel_grouping column that maps source/medium combinations
      to human-readable channel labels used in dashboards. Campaign is not
      included in the grain of this dimension because campaigns are transient —
      a new campaign name does not warrant a new dimension row. Campaign names
      are stored directly on the fact tables.
    columns:
      - name: traffic_source_key
        description: "Surrogate key generated from source and medium."
        tests:
          - unique
          - not_null
      - name: source
        description: "Traffic source from traffic_source.source. e.g. google, facebook.com, (direct)."
      - name: medium
        description: "Traffic medium from traffic_source.medium. e.g. organic, cpc, referral, (none)."
      - name: channel_grouping
        description: >
          Business-friendly channel label derived from source and medium.
          Values: Direct, Organic Search, Paid Search, Referral, Email, Social, Other.
          Logic: Direct = source (direct) AND medium in ((none),(not set)).
          Organic Search = medium organic. Paid Search = medium cpc.
          Referral = medium referral. Email = medium email.
          Social = medium social/social-network OR source in known social domains.
          Other = everything else.
```

---

## Part 4: Fact Table SQL

### fct_product_interactions.sql

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
    description = 'One row per item per product event. Funnel steps 1-4.'
  )
}}

SELECT
  {{ dbt_utils.generate_surrogate_key(['event_id', 'item_id']) }} AS interaction_id,
  event_name,
  funnel_step,
  item_id,
  item_name,
  item_brand,
  item_category,
  item_category_2,
  item_variant,
  price,
  quantity,
  item_revenue                 AS revenue,
  user_pseudo_id,
  session_id,
  event_date                   AS interaction_date,
  event_timestamp_utc          AS interacted_at,
  device_category,
  device_operating_system,
  geo_country,
  traffic_source,
  traffic_medium,
  traffic_campaign
FROM {{ ref('int_ga4__product_events') }}
```

---

### fct_purchases.sql

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
    description = 'One row per purchase transaction. Grain is transaction not item.'
  )
}}

WITH purchase_rows AS (

  SELECT *
  FROM {{ ref('int_ga4__product_events') }}
  WHERE event_name = 'purchase'
    AND transaction_id IS NOT NULL

),

-- One purchase event can theoretically fire more than once for the same
-- transaction_id in GA4. Deduplicate to guarantee one row per transaction.
deduplicated AS (

  SELECT
    *,
    ROW_NUMBER() OVER (
      PARTITION BY transaction_id
      ORDER BY event_timestamp_utc
    ) AS row_num
  FROM purchase_rows

)

SELECT
  {{ dbt_utils.generate_surrogate_key(['event_id']) }} AS purchase_id,
  transaction_id,
  user_pseudo_id,
  session_id,
  event_date                  AS purchase_date,
  event_timestamp_utc         AS purchased_at,
  purchase_revenue            AS revenue,
  purchase_revenue_usd        AS revenue_usd,
  unique_items                AS item_count,
  total_item_quantity         AS total_quantity,
  device_category,
  device_operating_system,
  geo_country,
  traffic_source,
  traffic_medium,
  traffic_campaign
FROM deduplicated
WHERE row_num = 1
```

---

### fct_sessions.sql

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
    cluster_by = ['session_source', 'device_category'],
    description = 'One row per session. All logic lives in int_ga4__sessions.'
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
  total_engagement_ms,
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
  session_source,
  session_medium,
  session_campaign,
  traffic_source,
  traffic_medium,
  traffic_campaign
FROM {{ ref('int_ga4__sessions') }}
```

---

## Part 5: Dimension Table SQL

### Step 1 — Generate the dim_date seed

Run this Python script once from your project root to generate the CSV seed file. This covers the full GA4 sample dataset date range.

```python
# run_once: generate_dim_date_seed.py
# Run from project root: python generate_dim_date_seed.py

import csv
from datetime import date, timedelta

start = date(2020, 11, 1)
end   = date(2021, 1, 31)

rows = []
d = start
while d <= end:
    rows.append({
        "date_day":     d.isoformat(),
        "year":         d.year,
        "quarter":      (d.month - 1) // 3 + 1,
        "month":        d.month,
        "month_name":   d.strftime("%B"),
        "week_of_year": int(d.strftime("%W")),
        "day_of_week":  d.isoweekday(),
        "day_name":     d.strftime("%A"),
        "is_weekend":   str(d.isoweekday() >= 6).upper()
    })
    d += timedelta(days=1)

with open("seeds/dim_date.csv", "w", newline="") as f:
    w = csv.DictWriter(f, fieldnames=rows[0].keys())
    w.writeheader()
    w.writerows(rows)

print(f"Written {len(rows)} rows to seeds/dim_date.csv")
```

After running the script, load the seed into BigQuery:

```bash
dbt seed
```

This creates a table called `dim_date` in your dev dataset from the CSV.

---

### dim_users.sql

Create `models/marts/dim_users.sql`:

```sql
{{
  config(
    materialized = 'table',
    description = 'One row per user_pseudo_id. Built from first observed event per user.'
  )
}}

WITH first_event_per_user AS (

  SELECT
    user_pseudo_id,
    MIN(event_timestamp_utc)  AS first_visit_at,
    MIN(event_date)           AS first_visit_date
  FROM {{ ref('stg_ga4__events') }}
  GROUP BY user_pseudo_id

),

user_attributes AS (

  -- Take traffic source and LTV from any event row for this user.
  -- These fields are user-level and identical across all events for
  -- a given user_pseudo_id so MAX() safely returns the single value.
  SELECT
    user_pseudo_id,
    MAX(traffic_source)         AS acquisition_source,
    MAX(traffic_medium)         AS acquisition_medium,
    MAX(traffic_campaign)       AS acquisition_campaign,
    MAX(user_ltv_revenue)       AS user_ltv_revenue,
    MAX(user_ltv_currency)      AS user_ltv_currency
  FROM {{ ref('stg_ga4__events') }}
  GROUP BY user_pseudo_id

)

SELECT
  f.user_pseudo_id,
  f.first_visit_date,
  f.first_visit_at,
  a.acquisition_source,
  a.acquisition_medium,
  a.acquisition_campaign,
  a.user_ltv_revenue,
  a.user_ltv_currency
FROM first_event_per_user f
LEFT JOIN user_attributes a
  ON f.user_pseudo_id = a.user_pseudo_id
```

---

### dim_products.sql

Create `models/marts/dim_products.sql`:

```sql
{{
  config(
    materialized = 'table',
    description = 'One row per item_id. Derived from items array across all product events. Most recently observed values used for name and category.'
  )
}}

WITH all_items AS (

  SELECT
    item_id,
    item_name,
    item_brand,
    item_category,
    item_category_2,
    item_variant,
    event_id
  FROM {{ ref('stg_ga4__items') }}
  WHERE item_id IS NOT NULL

),

-- Join to events to get a timestamp for recency ordering
items_with_timestamp AS (

  SELECT
    i.item_id,
    i.item_name,
    i.item_brand,
    i.item_category,
    i.item_category_2,
    i.item_variant,
    e.event_timestamp_utc,
    ROW_NUMBER() OVER (
      PARTITION BY i.item_id
      ORDER BY e.event_timestamp_utc DESC
    ) AS recency_rank
  FROM all_items i
  LEFT JOIN {{ ref('stg_ga4__events') }} e
    ON i.event_id = e.event_id

)

SELECT
  item_id,
  item_name,
  item_brand,
  item_category,
  item_category_2,
  item_variant
FROM items_with_timestamp
WHERE recency_rank = 1
```

---

### dim_date.sql

Create `models/marts/dim_date.sql`:

```sql
{{
  config(
    materialized = 'table',
    description = 'One row per calendar date 2020-11-01 to 2021-01-31. Built from dim_date seed.'
  )
}}

SELECT
  CAST(date_day    AS DATE)    AS date_day,
  CAST(year        AS INT64)   AS year,
  CAST(quarter     AS INT64)   AS quarter,
  CAST(month       AS INT64)   AS month,
  month_name,
  CAST(week_of_year AS INT64)  AS week_of_year,
  CAST(day_of_week  AS INT64)  AS day_of_week,
  day_name,
  CAST(is_weekend  AS BOOL)    AS is_weekend
FROM {{ ref('dim_date') }}
```

> **Note on naming:** The seed file is named `dim_date.csv` so dbt names the seed table `dim_date`. The mart model is also named `dim_date.sql`. dbt handles this by resolving `{{ ref('dim_date') }}` to the seed in the mart model's FROM clause because the mart model itself takes over the name. If this causes a naming conflict in your dbt version, rename the seed to `dim_date_seed.csv` and update the ref accordingly.

---

### dim_device.sql

Create `models/marts/dim_device.sql`:

```sql
{{
  config(
    materialized = 'table',
    description = 'One row per unique device_category, operating_system, browser combination.'
  )
}}

WITH device_combinations AS (

  SELECT DISTINCT
    device_category,
    device_operating_system  AS operating_system,
    device_browser           AS browser,
    device_brand             AS mobile_brand_name
  FROM {{ ref('stg_ga4__events') }}
  WHERE device_category IS NOT NULL

)

SELECT
  {{ dbt_utils.generate_surrogate_key([
      'device_category',
      'operating_system',
      'browser'
  ]) }}                         AS device_key,
  device_category,
  operating_system,
  browser,
  mobile_brand_name
FROM device_combinations
```

---

### dim_traffic_source.sql

Create `models/marts/dim_traffic_source.sql`:

```sql
{{
  config(
    materialized = 'table',
    description = 'One row per source and medium combination. Includes channel grouping logic.'
  )
}}

WITH source_medium_combinations AS (

  SELECT DISTINCT
    COALESCE(traffic_source, '(direct)') AS source,
    COALESCE(traffic_medium, '(none)')   AS medium
  FROM {{ ref('stg_ga4__events') }}

),

with_channel_grouping AS (

  SELECT
    source,
    medium,
    CASE
      WHEN source = '(direct)'
       AND medium IN ('(none)', '(not set)')
        THEN 'Direct'
      WHEN medium = 'organic'
        THEN 'Organic Search'
      WHEN medium = 'cpc'
        THEN 'Paid Search'
      WHEN medium = 'referral'
        THEN 'Referral'
      WHEN medium = 'email'
        THEN 'Email'
      WHEN medium IN ('social', 'social-network', 'social-media', 'sm', 'social network')
        OR source IN ('facebook.com', 'instagram.com', 'twitter.com', 'linkedin.com',
                      't.co', 'l.facebook.com', 'lnkd.in')
        THEN 'Social'
      ELSE 'Other'
    END AS channel_grouping
  FROM source_medium_combinations

)

SELECT
  {{ dbt_utils.generate_surrogate_key(['source', 'medium']) }} AS traffic_source_key,
  source,
  medium,
  channel_grouping
FROM with_channel_grouping
```

---

## Part 6: Build and Verify

### Step 1 — Load the seed

```bash
dbt seed
```

Confirm `dim_date` appears in BigQuery with 92 rows.

### Step 2 — Run intermediate models

```bash
dbt run --select intermediate
```

### Step 3 — Run all mart models

```bash
dbt run --select marts
```

All eight mart models should build: three facts and five dimensions.

### Step 4 — Run all tests

```bash
dbt test
```

### Step 5 — Verify in BigQuery

```sql
-- Confirm all 8 mart tables exist and row counts make sense

SELECT 'fct_product_interactions' AS table_name, COUNT(*) AS rows FROM `YOUR_PROJECT.dbt_ga4_dev_marts.fct_product_interactions`
UNION ALL
SELECT 'fct_purchases',            COUNT(*) FROM `YOUR_PROJECT.dbt_ga4_dev_marts.fct_purchases`
UNION ALL
SELECT 'fct_sessions',             COUNT(*) FROM `YOUR_PROJECT.dbt_ga4_dev_marts.fct_sessions`
UNION ALL
SELECT 'dim_users',                COUNT(*) FROM `YOUR_PROJECT.dbt_ga4_dev_marts.dim_users`
UNION ALL
SELECT 'dim_products',             COUNT(*) FROM `YOUR_PROJECT.dbt_ga4_dev_marts.dim_products`
UNION ALL
SELECT 'dim_date',                 COUNT(*) FROM `YOUR_PROJECT.dbt_ga4_dev_marts.dim_date`
UNION ALL
SELECT 'dim_device',               COUNT(*) FROM `YOUR_PROJECT.dbt_ga4_dev_marts.dim_device`
UNION ALL
SELECT 'dim_traffic_source',       COUNT(*) FROM `YOUR_PROJECT.dbt_ga4_dev_marts.dim_traffic_source`;
```

**Expected (dev run against one week of data):**

| Table | Expected rows |
|---|---|
| fct_product_interactions | Thousands |
| fct_purchases | Dozens to low hundreds |
| fct_sessions | Hundreds to low thousands |
| dim_users | Hundreds |
| dim_products | Dozens |
| dim_date | 92 (full range, not date-filtered) |
| dim_device | Under 50 |
| dim_traffic_source | Under 30 |

```sql
-- Conversion rate sanity check (expect 1-5% for ecommerce)
SELECT
  ROUND(SUM(converted) / COUNT(*) * 100, 2) AS conversion_rate_pct
FROM `YOUR_PROJECT.dbt_ga4_dev_marts.fct_sessions`;

-- Funnel step check: all 4 steps present
SELECT funnel_step, event_name, COUNT(*) AS rows
FROM `YOUR_PROJECT.dbt_ga4_dev_marts.fct_product_interactions`
GROUP BY 1, 2
ORDER BY 1;

-- Channel grouping check: no nulls, all rows have a channel
SELECT channel_grouping, COUNT(*) AS rows
FROM `YOUR_PROJECT.dbt_ga4_dev_marts.dim_traffic_source`
GROUP BY 1
ORDER BY 2 DESC;
```

### Step 6 — Check the lineage graph

```bash
dbt docs generate
dbt docs serve
```

The lineage graph must show:

```
source: ga4_ecommerce.events
  ├── stg_ga4__events
  ├── stg_ga4__event_params
  └── stg_ga4__items
        │
        ├── int_ga4__product_events ──── fct_product_interactions
        │                           └─── fct_purchases
        │
        ├── int_ga4__session_traffic ─┐
        └── int_ga4__sessions ────────┘──── fct_sessions
        │
        ├── dim_users
        ├── dim_products
        ├── dim_device
        └── dim_traffic_source

seed: dim_date ──── dim_date
```

### Step 7 — Production run

Run once when dev checks pass and before connecting Tableau:

```bash
dbt run --target prod
dbt test --target prod
```

---

## Complete File Checklist

**Seeds:**
- [ ] `seeds/dim_date.csv` — 92 rows, 2020-11-01 to 2021-01-31

**Intermediate:**
- [ ] `models/intermediate/_intermediate.yml`
- [ ] `models/intermediate/int_ga4__product_events.sql`
- [ ] `models/intermediate/int_ga4__session_traffic.sql`
- [ ] `models/intermediate/int_ga4__sessions.sql`

**Marts:**
- [ ] `models/marts/_marts.yml`
- [ ] `models/marts/fct_product_interactions.sql`
- [ ] `models/marts/fct_purchases.sql`
- [ ] `models/marts/fct_sessions.sql`
- [ ] `models/marts/dim_users.sql`
- [ ] `models/marts/dim_products.sql`
- [ ] `models/marts/dim_date.sql`
- [ ] `models/marts/dim_device.sql`
- [ ] `models/marts/dim_traffic_source.sql`

**Verified:**
- [ ] `dbt seed` — 92 rows in dim_date
- [ ] `dbt run --select intermediate` — no errors
- [ ] `dbt run --select marts` — all 8 models build
- [ ] `dbt test` — all tests pass
- [ ] Row counts in BigQuery match expected ranges
- [ ] Lineage graph matches diagram
- [ ] `dbt run --target prod` completed once

When every box is checked, move on to [06_tableau.md](06_tableau.md).
