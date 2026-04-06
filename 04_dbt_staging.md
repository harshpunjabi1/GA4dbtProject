# Module 04: dbt Staging Layer

## What is the Staging Layer?

The staging layer is the first transformation step in the dbt project. Its job is to:

1. **Rename** raw field names to consistent, readable names
2. **Type cast** fields to the correct data types
3. **Flatten** nested structures one level at a time
4. **Standardise** nulls and placeholder values
5. **Add no business logic** — that belongs in the intermediate and mart layers

Think of staging as a **contract**: "this is what the source gives us, cleaned and typed." Nothing more.

Every staging model reads from the raw GA4 source. No staging model should read from another staging model.

---

## Step 1: Declare the Source

Before writing any models, we tell dbt where the source data lives. Create the file `models/staging/_sources.yml`:

```yaml
version: 2

sources:
  - name: ga4_ecommerce
    description: "Raw GA4 event export from the Google Merchandise Store in BigQuery"
    project: bigquery-public-data
    dataset: ga4_obfuscated_sample_ecommerce
    tables:
      - name: events
        identifier: events_*
        description: >
          Daily GA4 event tables. Each row is one event fired by one user.
          Event parameters are stored in a nested array (event_params).
          Product data is stored in a nested array (items).
          Tables are named events_YYYYMMDD and queried using wildcard.
```

---

## Step 2: Create the Staging Documentation File

Create `models/staging/_staging.yml`. This file documents every staging model and column. Write this as you build each model — do not skip it. These descriptions will be used to train Vanna AI in Module 08.

```yaml
version: 2

models:
  - name: stg_ga4__events
    description: >
      Base event record from GA4. One row per event. Flattens the top-level
      event fields and resolves the event timestamp to a usable datetime.
      Does not unnest event_params or items arrays.
    columns:
      - name: event_id
        description: "Surrogate key for the event, created from user_pseudo_id and event_timestamp"
        tests:
          - unique
          - not_null
      - name: user_pseudo_id
        description: "Pseudonymous identifier for the user. This is not a real user ID."
        tests:
          - not_null
      - name: event_name
        description: "The name of the GA4 event e.g. page_view, purchase, add_to_cart"
        tests:
          - not_null
      - name: event_timestamp_utc
        description: "Event timestamp converted from microseconds to UTC datetime"
      - name: event_date
        description: "Date of the event as a DATE type"
      - name: session_id
        description: "Session identifier constructed from user_pseudo_id and ga_session_id"
      - name: platform
        description: "Platform the event was fired on e.g. WEB, IOS, ANDROID"
      - name: device_category
        description: "Device category e.g. desktop, mobile, tablet"
      - name: device_operating_system
        description: "Operating system of the device"
      - name: device_browser
        description: "Browser used"
      - name: geo_country
        description: "Country of the user based on IP address"
      - name: traffic_source
        description: "User-level traffic source (where the user first came from)"
      - name: traffic_medium
        description: "User-level traffic medium"
      - name: traffic_campaign
        description: "User-level campaign name"

  - name: stg_ga4__event_params
    description: >
      Unnested event parameters. One row per parameter per event.
      Use this model to extract specific event parameters using WHERE key = 'param_name'.
      Join back to stg_ga4__events on event_id to get event context.
    columns:
      - name: event_id
        description: "Foreign key to stg_ga4__events"
        tests:
          - not_null
      - name: key
        description: "The parameter name e.g. transaction_id, page_location, session_engaged"
        tests:
          - not_null
      - name: value_string
        description: "Parameter value if it is a string type"
      - name: value_int
        description: "Parameter value if it is an integer type"
      - name: value_float
        description: "Parameter value if it is a float type"
      - name: value_double
        description: "Parameter value if it is a double type"
      - name: value
        description: >
          Coalesced value across all types. Use this column in most cases.
          Returns the first non-null value across string, int, float, double.

  - name: stg_ga4__items
    description: >
      Unnested product items from GA4 events. One row per item per event.
      Only events with items have rows in this table (view_item, add_to_cart, purchase).
      Join to stg_ga4__events on event_id to get event context.
    columns:
      - name: event_id
        description: "Foreign key to stg_ga4__events"
        tests:
          - not_null
      - name: item_id
        description: "Product SKU or identifier"
      - name: item_name
        description: "Product name"
      - name: item_brand
        description: "Product brand"
      - name: item_category
        description: "Primary product category"
      - name: item_category_2
        description: "Secondary product category"
      - name: item_variant
        description: "Product variant e.g. size, colour"
      - name: price
        description: "Unit price of the item in the transaction currency"
      - name: quantity
        description: "Number of units of this item in the event"
      - name: item_revenue
        description: "Revenue from this item (price x quantity)"
```

---

## Step 3: Write the Staging Models

### Model 1: stg_ga4__events.sql

Create `models/staging/stg_ga4__events.sql`:

```sql
{{
  config(
    materialized = 'view',
    description = 'Base GA4 event record. One row per event.'
  )
}}

/*
  This model reads from the GA4 wildcard table and flattens
  the top-level fields. It does NOT unnest event_params or items.
  Cost saving: we use a date range macro to limit dev runs.
*/

WITH source AS (

  SELECT
    *,
    _TABLE_SUFFIX AS table_date
  FROM {{ source('ga4_ecommerce', 'events') }}

  {% if target.name == 'dev' %}
    WHERE _TABLE_SUFFIX BETWEEN '20201101' AND '20201107'
  {% endif %}

),

renamed AS (

  SELECT
    -- Surrogate key
    {{ dbt_utils.generate_surrogate_key(['user_pseudo_id', 'event_timestamp']) }} AS event_id,

    -- User identifier
    user_pseudo_id,

    -- Event details
    event_name,
    TIMESTAMP_MICROS(event_timestamp)           AS event_timestamp_utc,
    PARSE_DATE('%Y%m%d', event_date)            AS event_date,

    -- Session identifier (constructed from user + session id param)
    CONCAT(
      user_pseudo_id,
      '_',
      CAST(
        (SELECT value.int_value
         FROM UNNEST(event_params)
         WHERE key = 'ga_session_id'
         LIMIT 1)
      AS STRING)
    )                                            AS session_id,

    -- Platform and device
    platform,
    device.category                             AS device_category,
    device.operating_system                     AS device_operating_system,
    device.web_info.browser                     AS device_browser,
    device.mobile_brand_name                    AS device_brand,

    -- Geography
    geo.country                                 AS geo_country,
    geo.region                                  AS geo_region,
    geo.city                                    AS geo_city,

    -- User-level traffic source (first touch)
    traffic_source.source                       AS traffic_source,
    traffic_source.medium                       AS traffic_medium,
    traffic_source.name                         AS traffic_campaign,

    -- Ecommerce fields (populated on purchase events)
    ecommerce.transaction_id,
    ecommerce.purchase_revenue,
    ecommerce.purchase_revenue_in_usd    AS purchase_revenue_usd,
    ecommerce.unique_items

  FROM source

)

SELECT * FROM renamed
```

---

### Model 2: stg_ga4__event_params.sql

Create `models/staging/stg_ga4__event_params.sql`:

```sql
{{
  config(
    materialized = 'view',
    description = 'Unnested event parameters. One row per param per event.'
  )
}}

WITH source AS (

  SELECT
    {{ dbt_utils.generate_surrogate_key(['user_pseudo_id', 'event_timestamp']) }} AS event_id,
    event_params
  FROM {{ source('ga4_ecommerce', 'events') }}

  {% if target.name == 'dev' %}
    WHERE _TABLE_SUFFIX BETWEEN '20201101' AND '20201107'
  {% endif %}

),

unnested AS (

  SELECT
    event_id,
    param.key                          AS key,
    param.value.string_value           AS value_string,
    param.value.int_value              AS value_int,
    param.value.float_value            AS value_float,
    param.value.double_value           AS value_double,
    COALESCE(
      param.value.string_value,
      CAST(param.value.int_value    AS STRING),
      CAST(param.value.float_value  AS STRING),
      CAST(param.value.double_value AS STRING)
    )                                  AS value
  FROM source,
  UNNEST(event_params) AS param

)

SELECT * FROM unnested
```

---

### Model 3: stg_ga4__items.sql

Create `models/staging/stg_ga4__items.sql`:

```sql
{{
  config(
    materialized = 'view',
    description = 'Unnested product items. One row per item per event.'
  )
}}

WITH source AS (

  SELECT
    {{ dbt_utils.generate_surrogate_key(['user_pseudo_id', 'event_timestamp']) }} AS event_id,
    event_name,
    items
  FROM {{ source('ga4_ecommerce', 'events') }}

  WHERE event_name IN ('view_item', 'add_to_cart', 'begin_checkout', 'purchase')

  {% if target.name == 'dev' %}
    AND _TABLE_SUFFIX BETWEEN '20201101' AND '20201107'
  {% endif %}

),

unnested AS (

  SELECT
    event_id,
    event_name,
    item.item_id,
    item.item_name,
    item.item_brand,
    item.item_category,
    item.item_category2                AS item_category_2,
    item.item_variant,
    item.price,
    item.quantity,
    COALESCE(item.item_revenue, item.price * item.quantity) AS item_revenue
  FROM source,
  UNNEST(items) AS item

)

SELECT * FROM unnested
```

---

## Step 4: Install dbt-utils Package

The staging models use `dbt_utils.generate_surrogate_key`. We need to install the `dbt-utils` package first.

Create the file `packages.yml` in the root of the `ga4_ecommerce` folder:

```yaml
packages:
  - package: dbt-labs/dbt_utils
    version: [">=1.0.0", "<2.0.0"]
```

Install it:
```bash
dbt deps
```

---

## Step 5: Run the Staging Models

```bash
# Run only the staging layer
dbt run --select staging

# Check for errors in the output
# Each model should show [OK]
```

In dev mode, this runs against one week of data only — keeping BigQuery costs minimal.

---

## Step 6: Test the Staging Models

```bash
dbt test --select staging
```

The tests defined in `_staging.yml` (unique, not_null) will run. Fix any failures before moving on.

---

## Step 7: Verify in BigQuery

1. Open BigQuery
2. Look for a dataset called `YOUR_PROJECT_dbt_ga4_dev_staging`
3. You should see three views: `stg_ga4__events`, `stg_ga4__event_params`, `stg_ga4__items`
4. Preview each one to confirm the data looks correct

---

## Understanding the Dev Limit

Notice the pattern in every staging model:

```sql
{% if target.name == 'dev' %}
  WHERE _TABLE_SUFFIX BETWEEN '20201101' AND '20201107'
{% endif %}
```

This is a **Jinja if statement**. When you run `dbt run` with the default `dev` target, it adds a date filter that limits the data to one week. When you run with `--target prod`, no filter is applied and the full dataset is used.

This saves significant BigQuery query costs during development. We will switch to prod only for the final run before connecting Tableau.

---

## What You Have Built

- [ ] Source declaration pointing at the GA4 public dataset
- [ ] `stg_ga4__events` — base event record, flattened
- [ ] `stg_ga4__event_params` — unnested parameters, one row per param
- [ ] `stg_ga4__items` — unnested products, one row per item
- [ ] Full column documentation in `_staging.yml`
- [ ] dbt tests passing on all three models

When you are ready, move on to [05_dbt_marts.md](05_dbt_marts.md).
