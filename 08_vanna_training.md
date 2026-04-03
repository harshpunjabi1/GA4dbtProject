# Module 08: Training Vanna and Running the Demo

---

## The Training Strategy

Vanna learns from three types of training data:

| Type | What It Is | Why It Matters |
|---|---|---|
| **DDL** | The `CREATE TABLE` definition of your mart | Tells Vanna the exact column names, types, and table structure |
| **Documentation** | Plain English descriptions of tables and columns | Helps Vanna understand business meaning, not just technical structure |
| **SQL examples** | Real question + correct SQL pairs | Shows Vanna how to translate specific types of questions |

The most important lesson: **documentation quality directly determines SQL quality.** The descriptions you wrote in `_marts.yml` are your documentation training data.

---

## The Training Script

Create `vanna/train_vanna.py`:

```python
"""
Vanna AI Training Script for GA4 Ecommerce Project

This script trains Vanna with:
1. DDL from your BigQuery mart tables
2. Documentation from your dbt schema.yml descriptions
3. Curated question/SQL pairs covering the 3 business questions

Run this script once after your dbt mart tables are built.
Re-run it if you add new tables or update documentation.
"""

import sys
import os
sys.path.append(os.path.dirname(os.path.abspath(__file__)))

from config import get_vanna_instance

print("Initialising Vanna...")
vn = get_vanna_instance()


# =============================================================================
# PART 1: DDL TRAINING
# Train Vanna with the structure of each mart table.
# =============================================================================

print("\nTraining on DDL...")

vn.train(ddl="""
CREATE TABLE dbt_ga4.fct_purchases (
    purchase_id         STRING NOT NULL,
    transaction_id      STRING,
    user_pseudo_id      STRING NOT NULL,
    session_id          STRING,
    purchase_date       DATE,
    purchased_at        TIMESTAMP,
    revenue             FLOAT64,
    item_count          INT64,
    total_quantity      INT64,
    device_category     STRING,
    device_operating_system STRING,
    geo_country         STRING,
    traffic_source      STRING,
    traffic_medium      STRING,
    traffic_campaign    STRING
)
PARTITION BY purchase_date
CLUSTER BY traffic_source, device_category;
""")

vn.train(ddl="""
CREATE TABLE dbt_ga4.fct_product_interactions (
    interaction_id      STRING NOT NULL,
    event_name          STRING,
    funnel_step         INT64,
    item_id             STRING,
    item_name           STRING,
    item_brand          STRING,
    item_category       STRING,
    item_category_2     STRING,
    item_variant        STRING,
    price               FLOAT64,
    quantity            INT64,
    revenue             FLOAT64,
    user_pseudo_id      STRING,
    session_id          STRING,
    interaction_date    DATE,
    device_category     STRING,
    traffic_source      STRING,
    traffic_medium      STRING
)
PARTITION BY interaction_date
CLUSTER BY event_name, item_category;
""")

vn.train(ddl="""
CREATE TABLE dbt_ga4.fct_sessions (
    session_id              STRING NOT NULL,
    user_pseudo_id          STRING,
    session_date            DATE,
    session_start_at        TIMESTAMP,
    session_end_at          TIMESTAMP,
    session_duration_seconds INT64,
    session_number          INT64,
    session_engaged         INT64,
    event_count             INT64,
    page_view_count         INT64,
    product_view_count      INT64,
    add_to_cart_count       INT64,
    checkout_count          INT64,
    purchase_count          INT64,
    converted               INT64,
    device_category         STRING,
    device_operating_system STRING,
    device_browser          STRING,
    geo_country             STRING,
    traffic_source          STRING,
    traffic_medium          STRING,
    traffic_campaign        STRING,
    total_engagement_ms     INT64
)
PARTITION BY session_date
CLUSTER BY traffic_source, device_category;
""")


# =============================================================================
# PART 2: DOCUMENTATION TRAINING
# Train Vanna with business context for each table and column.
# These descriptions come directly from the schema.yml files.
# =============================================================================

print("Training on documentation...")

vn.train(documentation="""
Table: fct_purchases
One row per completed purchase transaction on the Google Merchandise Store.
This is the primary table for revenue analysis.
Use this table to answer questions about: total revenue, average order value,
number of transactions, revenue by channel, revenue by device, revenue by product category.
The revenue column contains the total transaction value in USD.
Join to fct_product_interactions on session_id to see which products were in each purchase.
""")

vn.train(documentation="""
Table: fct_product_interactions
One row per product-level event. Contains view_item, add_to_cart, begin_checkout,
and purchase events where a product is involved.
Use this table for: funnel drop-off analysis, product view counts, add-to-cart rates,
product conversion rates, and category-level performance.
The funnel_step column orders events: 1=view_item, 2=add_to_cart, 3=begin_checkout, 4=purchase.
Revenue is only populated for purchase events (funnel_step = 4).
To calculate conversion rate: count distinct sessions at step 4 / count distinct sessions at step 1.
""")

vn.train(documentation="""
Table: fct_sessions
One row per user session. A session is a group of events from one user.
Use this table for: session counts, conversion rates by channel, engagement metrics,
traffic source analysis, and device breakdowns.
The converted column is 1 if the session resulted in a purchase, 0 if not.
Conversion rate = SUM(converted) / COUNT(session_id).
traffic_source and traffic_medium describe where the session came from
e.g. source='google', medium='organic' means organic Google search.
""")


# =============================================================================
# PART 3: QUESTION/SQL PAIR TRAINING
# Train Vanna with real business questions and correct SQL.
# These are curated to cover the 3 business questions from Module 00.
# =============================================================================

print("Training on question/SQL pairs...")

# --- Business Question 1: Acquisition ---

vn.train(
    question="What is the total revenue by traffic source?",
    sql="""
    SELECT
        traffic_source,
        traffic_medium,
        ROUND(SUM(revenue), 2)          AS total_revenue,
        COUNT(DISTINCT user_pseudo_id)  AS unique_purchasers,
        COUNT(*)                        AS total_transactions
    FROM dbt_ga4.fct_purchases
    GROUP BY 1, 2
    ORDER BY total_revenue DESC
    """
)

vn.train(
    question="What is the session conversion rate by traffic medium?",
    sql="""
    SELECT
        traffic_medium,
        COUNT(*)                            AS total_sessions,
        SUM(converted)                      AS converted_sessions,
        ROUND(SUM(converted) / COUNT(*), 4) AS conversion_rate
    FROM dbt_ga4.fct_sessions
    WHERE traffic_medium IS NOT NULL
    GROUP BY traffic_medium
    ORDER BY conversion_rate DESC
    """
)

vn.train(
    question="Which traffic sources drive the most sessions?",
    sql="""
    SELECT
        traffic_source,
        traffic_medium,
        COUNT(DISTINCT session_id)          AS sessions,
        COUNT(DISTINCT user_pseudo_id)      AS unique_users,
        SUM(converted)                      AS conversions
    FROM dbt_ga4.fct_sessions
    GROUP BY 1, 2
    ORDER BY sessions DESC
    LIMIT 20
    """
)

vn.train(
    question="What is the weekly revenue trend?",
    sql="""
    SELECT
        DATE_TRUNC(purchase_date, WEEK)     AS week,
        ROUND(SUM(revenue), 2)              AS weekly_revenue,
        COUNT(DISTINCT user_pseudo_id)      AS unique_purchasers,
        COUNT(*)                            AS transactions,
        ROUND(AVG(revenue), 2)              AS avg_order_value
    FROM dbt_ga4.fct_purchases
    GROUP BY week
    ORDER BY week
    """
)

# --- Business Question 2: Funnel ---

vn.train(
    question="How many users reach each step of the purchase funnel?",
    sql="""
    SELECT
        funnel_step,
        event_name                          AS funnel_step_name,
        COUNT(DISTINCT user_pseudo_id)      AS unique_users,
        COUNT(DISTINCT session_id)          AS unique_sessions
    FROM dbt_ga4.fct_product_interactions
    GROUP BY 1, 2
    ORDER BY funnel_step
    """
)

vn.train(
    question="What is the funnel drop-off rate between each step?",
    sql="""
    WITH funnel_counts AS (
        SELECT
            funnel_step,
            event_name,
            COUNT(DISTINCT session_id) AS sessions_at_step
        FROM dbt_ga4.fct_product_interactions
        GROUP BY 1, 2
    ),
    with_previous AS (
        SELECT
            funnel_step,
            event_name,
            sessions_at_step,
            LAG(sessions_at_step) OVER (ORDER BY funnel_step) AS previous_step_sessions
        FROM funnel_counts
    )
    SELECT
        funnel_step,
        event_name,
        sessions_at_step,
        previous_step_sessions,
        ROUND(
            (previous_step_sessions - sessions_at_step) / previous_step_sessions,
            4
        ) AS drop_off_rate
    FROM with_previous
    ORDER BY funnel_step
    """
)

vn.train(
    question="How does funnel conversion differ between desktop and mobile?",
    sql="""
    SELECT
        device_category,
        funnel_step,
        event_name,
        COUNT(DISTINCT session_id) AS sessions
    FROM dbt_ga4.fct_product_interactions
    GROUP BY 1, 2, 3
    ORDER BY device_category, funnel_step
    """
)

# --- Business Question 3: Products ---

vn.train(
    question="What are the top 10 products by revenue?",
    sql="""
    SELECT
        item_name,
        item_category,
        ROUND(SUM(revenue), 2)              AS total_revenue,
        SUM(quantity)                       AS units_sold,
        COUNT(DISTINCT session_id)          AS purchase_sessions
    FROM dbt_ga4.fct_product_interactions
    WHERE event_name = 'purchase'
    GROUP BY 1, 2
    ORDER BY total_revenue DESC
    LIMIT 10
    """
)

vn.train(
    question="Which products have the highest view count but lowest purchase rate?",
    sql="""
    WITH product_funnel AS (
        SELECT
            item_id,
            item_name,
            item_category,
            COUNT(DISTINCT CASE WHEN event_name = 'view_item'   THEN session_id END) AS views,
            COUNT(DISTINCT CASE WHEN event_name = 'add_to_cart' THEN session_id END) AS adds,
            COUNT(DISTINCT CASE WHEN event_name = 'purchase'    THEN session_id END) AS purchases
        FROM dbt_ga4.fct_product_interactions
        GROUP BY 1, 2, 3
    )
    SELECT
        item_name,
        item_category,
        views,
        adds,
        purchases,
        ROUND(SAFE_DIVIDE(purchases, views), 4)  AS view_to_purchase_rate,
        ROUND(SAFE_DIVIDE(adds, views), 4)        AS view_to_cart_rate
    FROM product_funnel
    WHERE views > 50
    ORDER BY views DESC, view_to_purchase_rate ASC
    LIMIT 20
    """
)

vn.train(
    question="What is the revenue and transaction count by product category?",
    sql="""
    SELECT
        item_category,
        ROUND(SUM(revenue), 2)              AS total_revenue,
        COUNT(DISTINCT session_id)          AS transactions,
        SUM(quantity)                       AS units_sold,
        ROUND(AVG(price), 2)                AS avg_price
    FROM dbt_ga4.fct_product_interactions
    WHERE event_name = 'purchase'
      AND item_category IS NOT NULL
    GROUP BY item_category
    ORDER BY total_revenue DESC
    """
)

print("\nTraining complete!")
print("Vanna has been trained on:")
print("  - 3 DDL definitions")
print("  - 3 documentation blocks")
print("  - 10 question/SQL pairs")
print("\nYou can now run vanna_demo.py to ask questions.")
```

---

## The Demo Script

Create `vanna/vanna_demo.py`:

```python
"""
Vanna AI Demo Script

Ask natural language questions about the GA4 ecommerce data.
Vanna will generate SQL, run it against BigQuery, and return results.
"""

import sys
import os
sys.path.append(os.path.dirname(os.path.abspath(__file__)))

from config import get_vanna_instance
import pandas as pd

pd.set_option('display.max_columns', None)
pd.set_option('display.width', 120)

print("Starting Vanna GA4 Demo...")
vn = get_vanna_instance()

def ask(question: str):
    """Ask Vanna a question and print the SQL and result."""
    print(f"\n{'='*60}")
    print(f"Question: {question}")
    print('='*60)

    sql = vn.generate_sql(question)
    print(f"\nGenerated SQL:\n{sql}")

    try:
        result = vn.run_sql(sql)
        print(f"\nResult:")
        print(result.to_string(index=False))
    except Exception as e:
        print(f"\nError running SQL: {e}")


# --- Run the demo questions ---

ask("What is the total revenue by traffic source?")

ask("What is the session conversion rate by traffic medium?")

ask("How many users reach each step of the purchase funnel?")

ask("What are the top 10 products by revenue?")

ask("Which products have the highest view count but lowest purchase rate?")

# --- Try your own question ---
print("\n" + "="*60)
print("Try your own question:")
user_question = input("> ")
if user_question:
    ask(user_question)
```

---

## Running the Training and Demo

```bash
# Navigate to the vanna directory
cd vanna

# Run training (do this once, or when you update documentation)
python train_vanna.py

# Run the demo
python vanna_demo.py
```

---

## The Quality Demonstration

This is the most important teaching moment in the module. Run this experiment:

**Before training documentation:**
Ask: `"What is the conversion rate by channel?"`
Note the SQL Vanna generates — it may be incorrect or generic.

**After training documentation:**
The same question should produce correct SQL using `fct_sessions` with the right `SUM(converted) / COUNT(*)` formula.

The difference is the documentation training block that explained:
> "The converted column is 1 if the session resulted in a purchase. Conversion rate = SUM(converted) / COUNT(session_id)."

This is why every description in `_marts.yml` matters.

---

## What You Have Built

- [ ] `vanna/config.py` — Vanna class configuration
- [ ] `vanna/train_vanna.py` — full training script with DDL, docs, and SQL pairs
- [ ] `vanna/vanna_demo.py` — interactive demo script
- [ ] Vanna trained and returning correct SQL for all 10 example questions
- [ ] Demonstrated the documentation quality → SQL quality connection

When you are ready, move on to [09_portfolio.md](09_portfolio.md).
