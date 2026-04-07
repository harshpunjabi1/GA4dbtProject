# Module 06: Power BI Dashboards

## Setup: Connect Power BI to BigQuery

1. Download and install **Power BI Desktop** (free) from [microsoft.com/power-bi](https://www.microsoft.com/power-bi)
2. Open Power BI Desktop
3. Home → **Get Data** → **Google BigQuery**
4. Sign in with your Google account
5. Select your Project → Dataset: `dbt_ga4` (the prod dataset)
6. Select the three mart tables: `fct_purchases`, `fct_product_interactions`, `fct_sessions`
7. Click **Load**

**Important:** Connect to the `dbt_ga4` prod dataset, not `dbt_ga4_dev`. Make sure you have run `dbt run --target prod` first.

### Set Up Relationships

In Power BI go to **Model view** and confirm or create these relationships:

- `fct_sessions.session_date` → `dim_date.date_day`
- `fct_sessions.user_pseudo_id` → `dim_users.user_pseudo_id`
- `fct_purchases.user_pseudo_id` → `dim_users.user_pseudo_id`
- `fct_product_interactions.item_id` → `dim_products.item_id`
- `fct_sessions.device_category` → `dim_device.device_category`
- `fct_sessions.traffic_source` → `dim_traffic_source.source`

Power BI may auto-detect some of these. Verify them before building visuals.

---

## Dashboard 1: Acquisition Performance

**Business question:** Which channels bring us users and which convert?

**Data source:** `fct_sessions`

### DAX Measures

Create these in **Home → New Measure** before building charts:

```dax
Conversion Rate = DIVIDE(SUM(fct_sessions[converted]), COUNTROWS(fct_sessions))
```

### Visuals to Build

**Visual 1 — Sessions by channel over time**
- Visual type: Line chart
- X axis: `session_date` (set hierarchy to Week)
- Y axis: `COUNT of session_id`
- Legend: `session_medium`

**Visual 2 — Conversion rate by channel**
- Visual type: Bar chart (horizontal)
- Y axis: `traffic_source`
- X axis: `Conversion Rate` measure
- Sort: descending by Conversion Rate

**Visual 3 — Sessions vs conversion scatter**
- Visual type: Scatter chart
- X axis: `COUNT of session_id` (volume)
- Y axis: `Conversion Rate` measure
- Values/Details: `traffic_source` (one dot per source)
- This reveals high-volume/low-conversion vs low-volume/high-conversion channels

### Dashboard title (use this exact wording)

> "Which channels bring us users — and which convert them to purchase?"

---

## Dashboard 2: Purchase Funnel

**Business question:** Where do users drop off in the purchase journey?

**Data source:** `fct_product_interactions`

### DAX Measures

```dax
Step 1 Views =
CALCULATE(
    DISTINCTCOUNT(fct_product_interactions[user_pseudo_id]),
    fct_product_interactions[event_name] = "view_item"
)

Step 2 Cart =
CALCULATE(
    DISTINCTCOUNT(fct_product_interactions[user_pseudo_id]),
    fct_product_interactions[event_name] = "add_to_cart"
)

Step 3 Checkout =
CALCULATE(
    DISTINCTCOUNT(fct_product_interactions[user_pseudo_id]),
    fct_product_interactions[event_name] = "begin_checkout"
)

Step 4 Purchase =
CALCULATE(
    DISTINCTCOUNT(fct_product_interactions[user_pseudo_id]),
    fct_product_interactions[event_name] = "purchase"
)

Cart to View % = DIVIDE([Step 2 Cart], [Step 1 Views])
Checkout to Cart % = DIVIDE([Step 3 Checkout], [Step 2 Cart])
Purchase to Checkout % = DIVIDE([Step 4 Purchase], [Step 3 Checkout])
Overall Conversion % = DIVIDE([Step 4 Purchase], [Step 1 Views])
```

### Visuals to Build

**Visual 1 — Funnel chart**
- Visual type: Funnel chart (built-in Power BI visual)
- Values: `Step 1 Views`, `Step 2 Cart`, `Step 3 Checkout`, `Step 4 Purchase` measures in order
- Power BI will automatically calculate and display drop-off percentages between steps

**Visual 2 — Funnel by device**
- Visual type: Clustered bar chart
- Y axis: `device_category`
- X axis: all four Step measures as separate series
- Shows whether mobile has worse drop-off than desktop

**Visual 3 — Conversion rate trend (weekly)**
- Visual type: Line chart
- X axis: `interaction_date` (set hierarchy to Week)
- Y axis: `Overall Conversion %` measure

### Dashboard title

> "At which step do we lose the most users — and does device type matter?"

---

## Dashboard 3: Product Performance

**Business question:** Which products have high interest but low conversion?

**Data source:** `fct_product_interactions`

### DAX Measures

```dax
Product Views =
CALCULATE(
    DISTINCTCOUNT(fct_product_interactions[session_id]),
    fct_product_interactions[event_name] = "view_item"
)

Product Purchases =
CALCULATE(
    DISTINCTCOUNT(fct_product_interactions[session_id]),
    fct_product_interactions[event_name] = "purchase"
)

View to Purchase Rate = DIVIDE([Product Purchases], [Product Views])

Total Revenue =
CALCULATE(
    SUM(fct_product_interactions[revenue]),
    fct_product_interactions[event_name] = "purchase"
)
```

### Visuals to Build

**Visual 1 — Top products by revenue**
- Visual type: Bar chart (horizontal)
- Y axis: `item_name`
- X axis: `Total Revenue` measure
- Filter: `event_name = purchase` (apply as a visual-level filter)
- Show top 15 using the Top N filter in the Filters pane

**Visual 2 — View-to-purchase rate by product (scatter)**
- Visual type: Scatter chart
- X axis: `Product Views` (volume of interest)
- Y axis: `View to Purchase Rate`
- Values/Details: `item_name`
- This exposes the high-interest/low-conversion opportunity quadrant — annotate it with a text box

**Visual 3 — Category-level funnel**
- Visual type: Clustered bar chart
- Y axis: `item_category`
- X axis: `Product Views`, `Step 2 Cart`, `Total Revenue` as separate series
- Shows how each category performs across the funnel

### Dashboard title

> "Which products are generating revenue — and which have interest but not conversion?"

---

## Publishing

Power BI Desktop reports can be shared in two ways depending on your setup:

**Option A — Power BI Service (recommended)**
1. Sign up for a free Power BI account at [app.powerbi.com](https://app.powerbi.com)
2. In Power BI Desktop: **Home → Publish**
3. Select your workspace
4. Copy the published report URL — add this to your GitHub README

**Option B — Export to PDF**
1. **File → Export → Export to PDF**
2. Commit the PDF to your repo under `docs/dashboard_screenshots/`

Note: The free Power BI Service tier is sufficient. Unlike Tableau Public, Power BI Service reports are private by default — you control sharing.

---

## What You Have Built

- Power BI Desktop connected to BigQuery prod marts
- Dashboard 1: Acquisition Performance
- Dashboard 2: Purchase Funnel
- Dashboard 3: Product Performance
- All DAX measures documented above
- Report published to Power BI Service or exported as PDF

When every dashboard is complete, move on to [07_vanna_setup.md](../setup/07_vanna_setup.md).
