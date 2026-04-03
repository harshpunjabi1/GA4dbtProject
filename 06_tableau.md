# Module 06: Tableau Dashboards

## Setup: Connect Tableau to BigQuery

1. Download and install **Tableau Public** from [public.tableau.com](https://public.tableau.com)
2. Open Tableau Public
3. Under **Connect** → **To a Server** → **Google BigQuery**
4. Sign in with your Google account
5. Select your **Project** → **Dataset:** `dbt_ga4` (the prod dataset)
6. You will see your three mart tables: `fct_purchases`, `fct_product_interactions`, `fct_sessions`

> **Important:** Connect to the `dbt_ga4` prod dataset, not `dbt_ga4_dev`. Make sure you have run `dbt run --target prod` first.

---

## Dashboard 1: Acquisition Performance

**Business question:** Which channels bring us users and which convert?

### Data source: `fct_sessions`

### Charts to build:

**Chart 1 — Sessions by channel over time**
- Rows: `session_date` (set to Week)
- Columns: `COUNT(session_id)`
- Color: `traffic_medium`
- Chart type: Line

**Chart 2 — Conversion rate by channel**
- Create a calculated field: `Conversion Rate = SUM(converted) / COUNT(session_id)`
- Rows: `traffic_source`
- Columns: `Conversion Rate`
- Sort: descending
- Chart type: Bar

**Chart 3 — Sessions vs Conversions (scatter)**
- X axis: `COUNT(session_id)` (volume)
- Y axis: `Conversion Rate`
- Mark: `traffic_source`
- This reveals high-volume/low-conversion vs low-volume/high-conversion channels

### Dashboard title (use this exact wording):
> "Which channels bring us users — and which convert them to purchase?"

---

## Dashboard 2: Purchase Funnel

**Business question:** Where do users drop off in the purchase journey?

### Data source: `fct_product_interactions`

### Charts to build:

**Chart 1 — Funnel chart**
- Create a calculated field for each step count:
  - `Step 1 - Views = COUNTD(IF [event_name]='view_item' THEN [user_pseudo_id] END)`
  - `Step 2 - Cart = COUNTD(IF [event_name]='add_to_cart' THEN [user_pseudo_id] END)`
  - `Step 3 - Checkout = COUNTD(IF [event_name]='begin_checkout' THEN [user_pseudo_id] END)`
  - `Step 4 - Purchase = COUNTD(IF [event_name]='purchase' THEN [user_pseudo_id] END)`
- Arrange as a horizontal bar chart ordered by funnel step
- Add drop-off percentage as a label between bars

**Chart 2 — Funnel by device**
- Same data, split by `device_category`
- Shows whether mobile has worse drop-off than desktop

**Chart 3 — Conversion rate trend (weekly)**
- X: `interaction_date` (week)
- Y: `Step 4 / Step 1` (purchase / view ratio)
- Line chart

### Dashboard title:
> "At which step do we lose the most users — and does device type matter?"

---

## Dashboard 3: Product Performance

**Business question:** Which products have high interest but low conversion?

### Data source: `fct_product_interactions`

### Charts to build:

**Chart 1 — Top products by revenue**
- Rows: `item_name` (top 15)
- Columns: `SUM(revenue)`
- Filter: `event_name = 'purchase'`
- Chart type: Horizontal bar

**Chart 2 — View-to-purchase ratio by product**
- Create calculated fields:
  - `Views = COUNTD(IF [event_name]='view_item' THEN [session_id] END)`
  - `Purchases = COUNTD(IF [event_name]='purchase' THEN [session_id] END)`
  - `View to Purchase Rate = [Purchases] / [Views]`
- Scatter: X = Views (volume of interest), Y = View to Purchase Rate (conversion)
- Marks labelled by `item_name`
- Quadrant annotation: high views + low conversion = opportunity

**Chart 3 — Category breakdown**
- Rows: `item_category`
- Columns: Views, Adds to Cart, Purchases (side-by-side bars)
- Shows category-level funnel

### Dashboard title:
> "Which products are generating revenue — and which have interest but not conversion?"

---

## Publishing

1. File → Save to Tableau Public
2. Sign in to your Tableau Public account (create one free at public.tableau.com)
3. The workbook will be published and given a public URL
4. Copy this URL — you will add it to your GitHub README

> **Note:** Tableau Public workbooks are visible to anyone. Since the GA4 data is already a public dataset, this is fine.

---

## What You Have Built

- [ ] Tableau connected to BigQuery prod marts
- [ ] Dashboard 1: Acquisition Performance
- [ ] Dashboard 2: Purchase Funnel
- [ ] Dashboard 3: Product Performance
- [ ] All three dashboards published to Tableau Public
- [ ] Tableau Public URL saved for your README

When you are ready, move on to [../setup/07_vanna_setup.md](../setup/07_vanna_setup.md).
