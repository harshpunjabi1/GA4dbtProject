# Module 02: BEAM Workshop
## Business Event Analysis & Modeling

---

## What is BEAM?

**Business Event Analysis & Modeling (BEAM)** is a methodology for designing data warehouse and BI solutions developed by Lawrence Corr, described in the book *Agile Data Warehouse Design*. The core principle is that a data model should be designed from business events — not from the shape of the source data and not from a list of reports someone wants.

The output of BEAM is a **dimensional model**: fact tables at the centre holding one row per business event, surrounded by dimension tables that describe the context of those events. This is the same Kimball-style star schema used in Project 2, but now you understand how to derive it systematically rather than guess at it.

BEAM uses a framework called the **7Ws** to interrogate each business event. The answers to the 7Ws produce the columns, tables, and relationships of the dimensional model.

---

## The 7Ws Framework

For every business event, ask these seven questions:

| W | Question | What it produces |
|---|---|---|
| **Who** | Who performs or is involved in the event? | A dimension table for a person or entity |
| **What** | What is acted upon? | A dimension table for the object of the action |
| **When** | When does the event happen? | A date dimension + a timestamp on the fact table |
| **Where** | Where does the event happen? | A dimension table for location or context |
| **Why** | Why does the event happen? | A dimension for causal context (channel, campaign, reason) |
| **How Many** | How much, how many, how often? | The numeric measures stored on the fact table itself |
| **How** | How is it performed or classified? | Additional dimension attributes or a natural key |

**The critical rule:** The first six Ws produce dimension tables. The seventh — How Many — produces the measures that live on the fact table. If an answer does not fit a dimension or a measure, it becomes an attribute on one of those.

---

## Step 1: Identify the Business Events

Before applying the 7Ws, identify which business events are worth modeling. The test is simple: if this event happened more or less, would someone in the business care?

From the data exploration in Module 01, the Google Merchandise Store fires these events. Working from richest to simplest:

| Event name in GA4 | Business event | Worth modeling? |
|---|---|---|
| `purchase` | Customer completes a purchase | **Yes** — the most important event, directly tied to revenue |
| `add_to_cart` | Customer adds a product to cart | **Yes** — signals purchase intent, funnel step |
| `view_item` | Customer views a product detail page | **Yes** — signals product interest, funnel step |
| `begin_checkout` | Customer starts the checkout process | **Yes** — funnel step between cart and purchase |
| `session_start` | User begins a session on the site | **Yes** — needed to analyse traffic and conversion at session level |
| `page_view` | User loads a page | **No** — too granular for the business questions we defined |
| `scroll` | User scrolls | **No** — no actionable business use in this project |
| `first_visit` | User visits for the first time | **Partial** — used as an attribute on the user dimension, not its own fact |
| `user_engagement` | User is actively engaged | **No** — an engagement time metric, not a modeled event |

**Three fact tables will be built:**

1. `fct_purchases` — one row per completed transaction
2. `fct_product_interactions` — one row per product per event (view, cart, checkout, purchase)
3. `fct_sessions` — one row per session

> **Why combine view_item, add_to_cart, begin_checkout, and purchase into one table?**
> These four events share an identical set of dimensions — same user, same product, same device, same session. The only thing that differs between them is which funnel step they represent. A single `fct_product_interactions` table with a `funnel_step` column is simpler to query for drop-off analysis and more efficient than three separate tables with duplicate joins. This is a deliberate dimensional modeling decision: when multiple events share the same grain and the same dimensions, consolidate them.

---

## Step 2: Apply the 7Ws — Event 1: Customer Completes a Purchase

**Event story:** A customer buys one or more products on the Google Merchandise Store website.
**Grain:** One row per transaction. One `purchase` event in GA4 = one row in `fct_purchases`.

---

### WHO?

**Question:** Who completed the purchase?

**GA4 field:** `user_pseudo_id` (STRING) — the pseudonymous cookie-based identifier that GA4 assigns to each browser. This is not a real user ID; it is obfuscated. The `user_id` field exists for logged-in users but is not reliably populated in the public sample dataset.

**Additional user-level fields available on every event row:**
- `user_first_touch_timestamp` (INTEGER) — microseconds since epoch when user first visited
- `user_ltv.revenue` (FLOAT64) — lifetime value of the user per GA4
- `user_ltv.currency` (STRING) — currency code for the LTV value
- `traffic_source.source` (STRING) — the source the user originally came from (first-touch, user-level, never changes)
- `traffic_source.medium` (STRING) — the medium (organic, cpc, referral, etc.)
- `traffic_source.name` (STRING) — the campaign name

**Model output:** → `dim_users`

---

### WHAT?

**Question:** What was purchased?

**GA4 field:** `items[]` — a REPEATED ARRAY of STRUCTs, one element per product in the transaction. Each element contains:
- `item_id` (STRING) — the product SKU
- `item_name` (STRING) — the product display name
- `item_brand` (STRING)
- `item_variant` (STRING) — size, colour, etc.
- `item_category` (STRING) — primary category
- `item_category2` (STRING) — secondary category
- `price` (FLOAT64) — unit price
- `quantity` (INT64) — units purchased
- `item_revenue` (FLOAT64) — price × quantity for this item
- `item_revenue_in_usd` (FLOAT64) — USD-normalised item revenue

**Important grain note:** One purchase event can contain multiple products. For `fct_purchases`, the grain is the **transaction**, not the individual item. `ecommerce.unique_items` gives the count of distinct products; `ecommerce.total_item_quantity` gives total units. Item-level purchase detail is captured in `fct_product_interactions`.

**Model output:** → `dim_products` (populated from item data across all product events, not just purchases)

---

### WHEN?

**Question:** When did the purchase happen?

**GA4 fields:**
- `event_timestamp` (INTEGER) — microseconds since Unix epoch. Transform: `TIMESTAMP_MICROS(event_timestamp)`
- `event_date` (STRING) — YYYYMMDD format. Transform: `PARSE_DATE('%Y%m%d', event_date)`

**Model output:** → `dim_date` as a calendar dimension table, plus `purchased_at` TIMESTAMP and `purchase_date` DATE as columns on `fct_purchases`

---

### WHERE?

**Question:** Where did the purchase happen — on what device, from what location?

**GA4 fields from the `device` STRUCT:**
- `device.category` (STRING) — values: `desktop`, `mobile`, `tablet`
- `device.operating_system` (STRING) — e.g. `Windows`, `iOS`, `Android`, `Macintosh`
- `device.browser` (STRING) — e.g. `Chrome`, `Safari`, `Firefox`
- `device.mobile_brand_name` (STRING) — e.g. `Apple`, `Samsung` — NULL for desktop

**GA4 fields from the `geo` STRUCT:**
- `geo.country` (STRING) — e.g. `United States`
- `geo.city` (STRING)
- `geo.continent` (STRING)
- `geo.region` (STRING)

**Model output:** → `dim_device` (from device fields). Geo fields are denormalised directly onto the fact tables as they have too high a cardinality to make a useful dimension in this dataset.

---

### WHY?

**Question:** Why did the user come to the site — what brought them here?

**GA4 field:** `traffic_source` (STRUCT) — this is a **user-level, first-touch attribution** field. It records the source/medium/campaign from when the user *first* ever visited the site. It does not change after the user's first visit, and it is identical on every event row for a given user regardless of which session the event belongs to.

- `traffic_source.source` (STRING) — e.g. `google`, `facebook.com`, `(direct)`
- `traffic_source.medium` (STRING) — e.g. `organic`, `cpc`, `referral`, `(none)`
- `traffic_source.name` (STRING) — campaign name

**Important limitation to teach:** Because `traffic_source` is first-touch and user-level, it answers "how did we originally acquire this user?" not "what brought them to the site today?". Session-level traffic attribution is a different field, reconstructed from `event_params` on `session_start` events — covered in the sessions 7W below.

**Model output:** → `dim_traffic_source`

---

### HOW MANY?

**Question:** How much revenue? How many items?

**GA4 fields from the `ecommerce` STRUCT:**
- `ecommerce.purchase_revenue` (FLOAT64) — total transaction revenue in the property's native currency
- `ecommerce.purchase_revenue_in_usd` (FLOAT64) — USD-normalised total revenue
- `ecommerce.total_item_quantity` (INT64) — total units across all items in the transaction
- `ecommerce.unique_items` (INT64) — number of distinct product SKUs in the transaction
- `ecommerce.tax_value` (FLOAT64) — tax component of the transaction
- `ecommerce.shipping_value` (FLOAT64) — shipping component of the transaction

**Model output:** These become the numeric measure columns on `fct_purchases`. They are not dimensions.

---

### HOW?

**Question:** What is the unique reference for this transaction?

**GA4 field:** `ecommerce.transaction_id` (STRING) — the merchant's own order reference number. This is the natural key that uniquely identifies one purchase.

**Model output:** `transaction_id` as a natural key column on `fct_purchases`.

---

### Resulting table: fct_purchases

**Grain:** One row per completed purchase transaction.

| Column | Source | Type | Notes |
|---|---|---|---|
| `purchase_id` | Generated surrogate key | STRING | Primary key |
| `transaction_id` | `ecommerce.transaction_id` | STRING | Natural key — HOW |
| `user_pseudo_id` | `user_pseudo_id` | STRING | FK → dim_users — WHO |
| `session_id` | Constructed: `CONCAT(user_pseudo_id, '_', ga_session_id)` | STRING | FK → fct_sessions — WHEN/HOW |
| `purchase_date` | `PARSE_DATE('%Y%m%d', event_date)` | DATE | FK → dim_date, partition key — WHEN |
| `purchased_at` | `TIMESTAMP_MICROS(event_timestamp)` | TIMESTAMP | Exact time — WHEN |
| `revenue` | `ecommerce.purchase_revenue` | FLOAT64 | Total transaction value — HOW MANY |
| `revenue_usd` | `ecommerce.purchase_revenue_in_usd` | FLOAT64 | USD-normalised — HOW MANY |
| `total_quantity` | `ecommerce.total_item_quantity` | INT64 | Units purchased — HOW MANY |
| `unique_items` | `ecommerce.unique_items` | INT64 | Distinct products — HOW MANY |
| `tax_value` | `ecommerce.tax_value` | FLOAT64 | — HOW MANY |
| `shipping_value` | `ecommerce.shipping_value` | FLOAT64 | — HOW MANY |
| `device_category` | `device.category` | STRING | FK → dim_device — WHERE |
| `operating_system` | `device.operating_system` | STRING | WHERE |
| `browser` | `device.browser` | STRING | WHERE |
| `geo_country` | `geo.country` | STRING | WHERE |
| `geo_city` | `geo.city` | STRING | WHERE |
| `traffic_source` | `traffic_source.source` | STRING | FK → dim_traffic_source — WHY |
| `traffic_medium` | `traffic_source.medium` | STRING | WHY |
| `traffic_campaign` | `traffic_source.name` | STRING | WHY |

---

## Step 3: Apply the 7Ws — Event 2: Customer Interacts with a Product

This fact table covers four ordered funnel events that all involve a product: `view_item`, `add_to_cart`, `begin_checkout`, and `purchase`. They share the same grain, dimensions, and physical structure. This consolidation is justified because the primary analytical use is funnel analysis — comparing how many sessions reach each step — which requires all steps in one query.

**Event stories:**
- A customer views a product detail page
- A customer adds a product to their cart
- A customer starts the checkout process with items in their cart
- A customer purchases products (items appear in the `items[]` array on purchase events too)

**Grain:** One row per item per event. If a customer adds two different products to cart in the same session, that is two rows. If a single `add_to_cart` event contains one product (the typical case for `view_item` and `add_to_cart`), that is one row.

---

### WHO?

**GA4 field:** `user_pseudo_id` — identical to fct_purchases.
**Model output:** → `dim_users` (same table, same FK)

---

### WHAT?

**GA4 fields from `items[]`:**
- `items[].item_id` (STRING) — product SKU
- `items[].item_name` (STRING) — product display name
- `items[].item_brand` (STRING)
- `items[].item_category` (STRING) — primary category
- `items[].item_category2` (STRING) — secondary category
- `items[].item_variant` (STRING)
- `items[].price` (FLOAT64) — unit price at time of this event
- `items[].quantity` (INT64) — units (1 for view events, variable for cart/checkout/purchase)
- `items[].item_revenue` (FLOAT64) — populated only on purchase events; NULL on view and cart events

**Model output:** → `dim_products` (same table). Item name and category are also denormalised directly onto the fact row for query convenience — analysts should not need to join to `dim_products` for basic product filtering.

---

### WHEN?

**GA4 fields:** `event_timestamp`, `event_date` — same transformation as purchases.
**Model output:** `interaction_date` DATE + `interacted_at` TIMESTAMP as columns on the fact row. FK → `dim_date`.

---

### WHERE?

**GA4 fields:** `device.category`, `device.operating_system`, `device.browser`
**Identical to fct_purchases.** → `dim_device`

---

### WHY?

**GA4 field:** `traffic_source` (user-level first-touch)
**Identical to fct_purchases.** → `dim_traffic_source`

---

### HOW MANY?

- `items[].price` (FLOAT64) — unit price at time of the event
- `items[].quantity` (INT64) — units in this event (typically 1 for views)
- `items[].item_revenue` (FLOAT64) — revenue from this item, **NULL unless event_name = 'purchase'**

**Model output:** `price`, `quantity`, and `revenue` as columns on the fact row. Revenue is NULL for non-purchase funnel steps by design — this is correct and expected.

---

### HOW?

- `event_name` (STRING) — the event type: `view_item`, `add_to_cart`, `begin_checkout`, `purchase`
- `funnel_step` (derived INT64): 1 = view_item, 2 = add_to_cart, 3 = begin_checkout, 4 = purchase

**Model output:** `event_name` and `funnel_step` as columns on the fact row. These are the classification fields that enable drop-off analysis.

---

### Resulting table: fct_product_interactions

**Grain:** One row per item per product event (view, cart, checkout, purchase).

| Column | Source | Type | Notes |
|---|---|---|---|
| `interaction_id` | Generated surrogate key | STRING | Primary key |
| `event_name` | `event_name` | STRING | view_item / add_to_cart / begin_checkout / purchase — HOW |
| `funnel_step` | Derived from event_name | INT64 | 1 / 2 / 3 / 4 — HOW |
| `item_id` | `items[].item_id` | STRING | FK → dim_products — WHAT |
| `item_name` | `items[].item_name` | STRING | Denormalised — WHAT |
| `item_category` | `items[].item_category` | STRING | Denormalised — WHAT |
| `item_brand` | `items[].item_brand` | STRING | WHAT |
| `item_variant` | `items[].item_variant` | STRING | WHAT |
| `price` | `items[].price` | FLOAT64 | Unit price — HOW MANY |
| `quantity` | `items[].quantity` | INT64 | Units — HOW MANY |
| `revenue` | `items[].item_revenue` | FLOAT64 | NULL unless purchase — HOW MANY |
| `user_pseudo_id` | `user_pseudo_id` | STRING | FK → dim_users — WHO |
| `session_id` | Constructed | STRING | FK → fct_sessions |
| `interaction_date` | `PARSE_DATE('%Y%m%d', event_date)` | DATE | FK → dim_date, partition key — WHEN |
| `interacted_at` | `TIMESTAMP_MICROS(event_timestamp)` | TIMESTAMP | WHEN |
| `device_category` | `device.category` | STRING | FK → dim_device — WHERE |
| `operating_system` | `device.operating_system` | STRING | WHERE |
| `geo_country` | `geo.country` | STRING | WHERE |
| `traffic_source` | `traffic_source.source` | STRING | FK → dim_traffic_source — WHY |
| `traffic_medium` | `traffic_source.medium` | STRING | WHY |

---

## Step 4: Apply the 7Ws — Event 3: Customer Starts a Session

A session in GA4 is not a stored object — it is a derived construct. GA4 fires a `session_start` event at the beginning of each session and attaches a `ga_session_id` integer to every subsequent event in that session via the `event_params` array. There is no pre-built sessions table in the export.

To model sessions, the intermediate layer reconstructs them by grouping all events that share the same `user_pseudo_id` and `ga_session_id` value from `event_params`.

**Event story:** A user visits the Google Merchandise Store, triggering a session that contains one or more events.
**Grain:** One row per session. A session is uniquely identified by the combination of `user_pseudo_id` and the integer value of `event_params[key='ga_session_id']`.

---

### WHO?

**GA4 field:** `user_pseudo_id`
**Same as above.** → `dim_users`

---

### WHAT?

Sessions have no product object. A session is about site-level behaviour, not a specific product. There is no FK to `dim_products` on `fct_sessions`. The number of product interactions within a session is captured as a measure (How Many).

---

### WHEN?

Session timing is derived by aggregating across all events in the session:
- Session start: `TIMESTAMP_MICROS(MIN(event_timestamp))` across all events with the same session key
- Session end: `TIMESTAMP_MICROS(MAX(event_timestamp))` across all events with the same session key
- Session date: the date of the first event in the session
- Session duration: `TIMESTAMP_DIFF(session_end_at, session_start_at, SECOND)`

**Model output:** `session_date` DATE (FK → dim_date) + `session_start_at` TIMESTAMP + `session_end_at` TIMESTAMP + `session_duration_seconds` INT64.

---

### WHERE?

**GA4 fields:** `device.category`, `device.operating_system`, `device.browser`
Device values are consistent within a session (a user does not change device mid-session). Take the first non-null value per session.
→ `dim_device`

---

### WHY?

This is where sessions differ critically from the other fact tables. `traffic_source` on the events table is user-level first-touch — it does not tell us what brought the user to the site **in this specific session**.

Session-level traffic attribution comes from `event_params` on the `session_start` event:
- `event_params[key='source']` → string_value — the source for this session
- `event_params[key='medium']` → string_value — the medium for this session
- `event_params[key='campaign']` → string_value — the campaign for this session

These values reflect the actual entry point of each session, not the user's original acquisition source. A user first acquired via Google Organic may return via a Direct session — the session-level fields capture that.

**Model output:** `session_source`, `session_medium`, `session_campaign` as columns on `fct_sessions`. These are separate from the `dim_traffic_source` FK, which uses user-level first-touch data.

---

### HOW MANY?

Events within each session are counted and aggregated:

| Measure | Derivation | GA4 source |
|---|---|---|
| `event_count` | `COUNT(*)` per session | All events |
| `page_view_count` | `COUNTIF(event_name = 'page_view')` | `event_name` |
| `product_view_count` | `COUNTIF(event_name = 'view_item')` | `event_name` |
| `add_to_cart_count` | `COUNTIF(event_name = 'add_to_cart')` | `event_name` |
| `checkout_count` | `COUNTIF(event_name = 'begin_checkout')` | `event_name` |
| `purchase_count` | `COUNTIF(event_name = 'purchase')` | `event_name` |
| `total_engagement_ms` | `SUM(event_params[engagement_time_msec])` | `event_params` key |
| `session_engaged` | `MAX(event_params[session_engaged])` | `event_params` key, 1=engaged |

**Model output:** All of the above as INT64 columns on `fct_sessions`.

---

### HOW?

- `event_params[key='ga_session_id']` (INT64) — the integer session identifier, unique per user (not globally unique — two different users can have the same ga_session_id value)
- `event_params[key='ga_session_number']` (INT64) — which session number this is for the user (1 = first ever session, 2 = second, etc.)
- `converted` (derived) — `IF(purchase_count > 0, 1, 0)` — whether this session resulted in a purchase

**Model output:** `session_number` INT64 + `converted` INT64 on `fct_sessions`.

---

### Resulting table: fct_sessions

**Grain:** One row per session, where session = unique (user_pseudo_id + ga_session_id) pair.

| Column | Source | Type | Notes |
|---|---|---|---|
| `session_id` | `CONCAT(user_pseudo_id, '_', CAST(ga_session_id AS STRING))` | STRING | Natural key — HOW |
| `user_pseudo_id` | `user_pseudo_id` | STRING | FK → dim_users — WHO |
| `session_date` | `MIN(PARSE_DATE('%Y%m%d', event_date))` per session | DATE | FK → dim_date, partition key — WHEN |
| `session_start_at` | `TIMESTAMP_MICROS(MIN(event_timestamp))` per session | TIMESTAMP | WHEN |
| `session_end_at` | `TIMESTAMP_MICROS(MAX(event_timestamp))` per session | TIMESTAMP | WHEN |
| `session_duration_seconds` | `TIMESTAMP_DIFF(session_end_at, session_start_at, SECOND)` | INT64 | WHEN |
| `session_number` | `event_params[ga_session_number]` on session_start event | INT64 | 1 = first session for this user — HOW |
| `session_engaged` | `MAX(event_params[session_engaged])` per session | INT64 | 1 = engaged session, 0 = bounce — HOW MANY |
| `total_engagement_ms` | `SUM(event_params[engagement_time_msec])` per session | INT64 | HOW MANY |
| `event_count` | `COUNT(*)` per session | INT64 | HOW MANY |
| `page_view_count` | `COUNTIF(event_name='page_view')` per session | INT64 | HOW MANY |
| `product_view_count` | `COUNTIF(event_name='view_item')` per session | INT64 | HOW MANY |
| `add_to_cart_count` | `COUNTIF(event_name='add_to_cart')` per session | INT64 | HOW MANY |
| `checkout_count` | `COUNTIF(event_name='begin_checkout')` per session | INT64 | HOW MANY |
| `purchase_count` | `COUNTIF(event_name='purchase')` per session | INT64 | HOW MANY |
| `converted` | `IF(purchase_count > 0, 1, 0)` | INT64 | 1 = session purchased — HOW |
| `device_category` | `device.category` (first non-null per session) | STRING | FK → dim_device — WHERE |
| `operating_system` | `device.operating_system` | STRING | WHERE |
| `browser` | `device.browser` | STRING | WHERE |
| `geo_country` | `geo.country` | STRING | WHERE |
| `session_source` | `event_params[source]` on session_start | STRING | Session-level attribution — WHY |
| `session_medium` | `event_params[medium]` on session_start | STRING | WHY |
| `session_campaign` | `event_params[campaign]` on session_start | STRING | WHY |

---

## Step 5: Derive the Dimension Tables

Each dimension table is derived from the W answers above. The grain of a dimension is always one row per unique entity.

---

### dim_users

**Derived from:** The WHO answer across all three fact tables.
**Grain:** One row per `user_pseudo_id`.
**Source:** The first event per user — `MIN(event_timestamp)` across all events for that `user_pseudo_id`.

| Column | Source | Type | Notes |
|---|---|---|---|
| `user_pseudo_id` | `user_pseudo_id` | STRING | Primary key |
| `first_visit_date` | `PARSE_DATE('%Y%m%d', event_date)` on first event | DATE | |
| `first_visit_at` | `TIMESTAMP_MICROS(MIN(event_timestamp))` per user | TIMESTAMP | |
| `acquisition_source` | `traffic_source.source` | STRING | First-touch source, from any event row for user |
| `acquisition_medium` | `traffic_source.medium` | STRING | First-touch medium |
| `acquisition_campaign` | `traffic_source.name` | STRING | First-touch campaign |
| `user_ltv_revenue` | `user_ltv.revenue` | FLOAT64 | GA4-calculated lifetime value — may be NULL in sample |
| `user_ltv_currency` | `user_ltv.currency` | STRING | Currency for LTV value |

> **Obfuscation note:** In the public sample dataset, `user_ltv.revenue` and `user_ltv.currency` will frequently be NULL or `<Other>`. This is a known limitation of the obfuscated dataset.

---

### dim_products

**Derived from:** The WHAT answer on `fct_product_interactions`.
**Grain:** One row per `item_id`.
**Source:** `items[]` array on `view_item`, `add_to_cart`, `begin_checkout`, and `purchase` events. Product metadata is extracted and deduplicated across all events where the product appears.

| Column | Source | Type | Notes |
|---|---|---|---|
| `item_id` | `items[].item_id` | STRING | Primary key |
| `item_name` | `items[].item_name` | STRING | Most recent non-null value seen |
| `item_brand` | `items[].item_brand` | STRING | |
| `item_category` | `items[].item_category` | STRING | Primary category |
| `item_category_2` | `items[].item_category2` | STRING | Secondary category |
| `item_variant` | `items[].item_variant` | STRING | Size, colour, etc. |

> **Design note:** GA4 does not maintain a separate product catalogue. `dim_products` is built by deduplicating `item_id` values seen across all product events, taking the most recently observed values for name and category. If a product name changes in GA4 over time, this dimension will reflect the most recent name.

---

### dim_date

**Derived from:** The WHEN answer across all three fact tables.
**Grain:** One row per calendar date.
**Source:** Generated as a dbt seed file — not derived from GA4 data. The date range covers the full dataset: 2020-11-01 to 2021-01-31.

| Column | Source | Type | Notes |
|---|---|---|---|
| `date_day` | The date itself | DATE | Primary key |
| `year` | `EXTRACT(YEAR FROM date_day)` | INT64 | |
| `quarter` | `EXTRACT(QUARTER FROM date_day)` | INT64 | |
| `month` | `EXTRACT(MONTH FROM date_day)` | INT64 | 1–12 |
| `month_name` | `FORMAT_DATE('%B', date_day)` | STRING | e.g. November |
| `week_of_year` | `EXTRACT(WEEK FROM date_day)` | INT64 | |
| `day_of_week` | `EXTRACT(DAYOFWEEK FROM date_day)` | INT64 | 1=Sunday, 7=Saturday |
| `day_name` | `FORMAT_DATE('%A', date_day)` | STRING | e.g. Monday |
| `is_weekend` | `day_of_week IN (1, 7)` | BOOL | |

---

### dim_device

**Derived from:** The WHERE answer across all three fact tables.
**Grain:** One row per unique combination of device category, operating system, and browser.
**Source:** `device` STRUCT on the events table.

| Column | Source | Type | Notes |
|---|---|---|---|
| `device_key` | Generated surrogate key | STRING | Primary key |
| `device_category` | `device.category` | STRING | desktop / mobile / tablet |
| `operating_system` | `device.operating_system` | STRING | Windows / iOS / Android / Macintosh |
| `browser` | `device.browser` | STRING | Chrome / Safari / Firefox |
| `mobile_brand_name` | `device.mobile_brand_name` | STRING | Apple / Samsung — NULL for desktop |

---

### dim_traffic_source

**Derived from:** The WHY answer across all three fact tables.
**Grain:** One row per unique (source, medium) combination.
**Source:** `traffic_source` STRUCT on the events table. Campaign is not included in the grain of this dimension because campaigns are transient — a new campaign name should not require inserting a new row. Campaign is left as a column on the fact tables.

| Column | Source | Type | Notes |
|---|---|---|---|
| `traffic_source_key` | Generated surrogate key | STRING | Primary key |
| `source` | `traffic_source.source` | STRING | e.g. google / facebook.com / (direct) |
| `medium` | `traffic_source.medium` | STRING | e.g. organic / cpc / referral / (none) |
| `channel_grouping` | Derived (see logic below) | STRING | Business-friendly channel label |

**Channel grouping derivation logic:**

| Condition | channel_grouping value |
|---|---|
| `source = '(direct)'` AND `medium IN ('(none)', '(not set)')` | `Direct` |
| `medium = 'organic'` | `Organic Search` |
| `medium = 'cpc'` | `Paid Search` |
| `medium = 'referral'` | `Referral` |
| `medium = 'email'` | `Email` |
| `medium IN ('social', 'social-network')` OR `source IN ('facebook.com', 'twitter.com', 'instagram.com', 'linkedin.com')` | `Social` |
| All other combinations | `Other` |

---

## Step 6: The Complete Dimensional Model

Every table in this model traces directly to a W answer in the workshop above.

```
                          dim_date
                             │
              dim_device ────┤
                             │
    dim_traffic_source ──────┼────── fct_purchases
                             │           one row per transaction
              dim_users ─────┼────── fct_product_interactions
                             │           one row per item per product event
            dim_products ────┤       fct_sessions
                             │           one row per session
                          dim_date
```

**Connections:**
- `dim_users` → all three fact tables (via `user_pseudo_id`)
- `dim_date` → all three fact tables (via `purchase_date`, `interaction_date`, `session_date`)
- `dim_device` → all three fact tables (via `device_category` / `device_key`)
- `dim_traffic_source` → all three fact tables (via `traffic_source` / `traffic_medium`)
- `dim_products` → `fct_product_interactions` and `fct_purchases` only (sessions have no product FK)

---

## Step 7: Verify Against the Three Business Questions

Before writing any code, confirm the model answers all three questions from Module 00.

**Question 1:** "Which channels bring us users — and which convert them to purchase?"
- Table: `fct_sessions` grouped by `session_source` / `session_medium`
- Metric: `SUM(converted) / COUNT(session_id)` = conversion rate
- ✓ Answerable

**Question 2:** "At which step do we lose the most users — and does device type matter?"
- Table: `fct_product_interactions` grouped by `funnel_step` and `device_category`
- Metric: `COUNT(DISTINCT session_id)` at each funnel_step. Drop-off = step N minus step N+1.
- ✓ Answerable

**Question 3:** "Which products have high interest but low purchase conversion?"
- Table: `fct_product_interactions` grouped by `item_id` / `item_name`
- Metric: sessions at funnel_step=1 (views) vs sessions at funnel_step=4 (purchases). Ratio = purchases / views.
- ✓ Answerable

---

## What You Have Now

- [ ] Business events identified and justified with a clear exclusion rationale
- [ ] Design decision for combining view/cart/checkout/purchase into one table explained
- [ ] 7Ws applied fully to all three fact tables with exact GA4 source field names and types
- [ ] Grain stated explicitly for every fact table
- [ ] Column-level definitions for all three fact tables including source field, type, and W mapping
- [ ] Five dimension tables with grain, source fields, and complete column definitions
- [ ] Important nuances documented: user_pseudo_id obfuscation, traffic_source first-touch vs session-level, items[] array grain, dim_products sourcing limitation
- [ ] Channel grouping logic defined for dim_traffic_source
- [ ] Complete dimensional model diagram drawn with connection notes
- [ ] All three business questions verified as answerable from the model

This document is the design contract. Every model built in dbt implements exactly what was decided here. If a table or column appears in dbt that is not in this document, it does not belong in the project.

When you are ready, move on to [../setup/03_dbt_setup.md](../setup/03_dbt_setup.md).
