# Modules 07 & 08: Vanna AI — Complete Setup, Training, and Demo Guide

---

## Before You Start

**Your machine:** Mac (macOS)

**Your IDE:** VS Code. All commands run in the VS Code integrated terminal (`Terminal > New Terminal`).

**Your project root:** `/Users/harshpunjabi/Documents/ga4_dbt`
This is the folder containing `dbt_project.yml`. Open this folder in VS Code (`File > Open Folder`), not a subfolder.

**Your service account:** `/Users/harshpunjabi/Documents/ga4_dbt/dbt_service_account.json`
Already exists. Do not touch it.

**Your GCP project ID:** `ga4-analysis-492300`
Already confirmed from your dbt logs.

**Your BigQuery dataset:** `dbt_ga4`
Your mart tables (`fct_purchases`, `fct_sessions`, `fct_product_interactions`) must already be built before running Vanna. Run `dbt run --target prod` first if not done.

**Your existing dbt environment:** `dbt_env` — do not install anything Vanna-related into this.

---

## Why Vanna Gets Its Own Environment

Vanna and dbt **cannot share a virtual environment**. They conflict on `protobuf`:

- dbt requires `protobuf>=6.0,<7.0`
- Vanna's Google dependencies pull in incompatible versions

Mixing them breaks dbt. The fix is a dedicated `vanna_env`.

**The rule for the rest of this project:**

| Task | Environment |
|---|---|
| `dbt run`, `dbt test`, `dbt docs` | `source dbt_env/bin/activate` |
| `python vanna/train_vanna.py` | `source vanna_env/bin/activate` |
| `python vanna/vanna_demo.py` | `source vanna_env/bin/activate` |
| `python vanna/test_connection.py` | `source vanna_env/bin/activate` |

Never run pip install in `dbt_env` for anything Vanna-related. Never run dbt commands inside `vanna_env`.

---

## Final Project File Structure

```
ga4_dbt/                                    ← project root, open this in VS Code
│
├── dbt_project.yml                         [EXISTS]
├── packages.yml                            [EXISTS]
├── dbt_service_account.json                [EXISTS] ← gitignored
├── .env                                    [NEW]    ← gitignored
├── .gitignore                              [EXISTS, UPDATED]
│
├── dbt_env/                                [EXISTS] ← dbt only, do not touch
├── vanna_env/                              [NEW]    ← Vanna only, gitignored
│
├── models/                                 [EXISTS]
│   └── marts/
│       ├── _marts.yml
│       ├── fct_purchases.sql
│       ├── fct_sessions.sql
│       └── fct_product_interactions.sql
│
└── vanna/                                  [NEW FOLDER]
    ├── config.py                           [NEW]
    ├── test_connection.py                  [NEW]
    ├── train_vanna.py                      [NEW]
    ├── vanna_demo.py                       [NEW]
    └── chroma_db/                          [AUTO-CREATED on first training run, gitignored]
```

---

## Part 1: Get a Google Gemini API Key

### Step 1.1 — Create the key

1. Go to [https://aistudio.google.com](https://aistudio.google.com)
2. Sign in with the same Google account that owns your BigQuery project
3. In the left sidebar click **Get API key**
4. Click **Create API key**
5. Select your existing Google Cloud project from the dropdown (`ga4-analysis-492300`)
6. Click **Create API key in existing project**
7. Copy the key — it starts with `AIzaSy...`
8. Save it in your password manager or notes. You cannot retrieve it again but can always create a new one.

Billing does not need to be enabled. The free tier is sufficient for this project.

---

## Part 2: Set Up the Python Environment

### Step 2.1 — Open the project in VS Code and verify location

Open VS Code. `File > Open Folder` → select `/Users/harshpunjabi/Documents/ga4_dbt`.

Open the integrated terminal (`Terminal > New Terminal`) and verify:

```bash
pwd
```

Expected:
```
/Users/harshpunjabi/Documents/ga4_dbt
```

---

### Step 2.2 — Deactivate dbt_env if it is active

If your prompt shows `(dbt_env)`:

```bash
deactivate
```

Your prompt should now show `(base)` or nothing. Do not proceed until `dbt_env` is deactivated.

---

### Step 2.3 — Create the Vanna virtual environment

```bash
python3 -m venv vanna_env
source vanna_env/bin/activate
```

Your prompt must now show `(vanna_env)`. If it still shows `(dbt_env)`, run `deactivate` first.

---

### Step 2.4 — Install all Vanna dependencies

Run this single command and wait for it to complete:

```bash
pip install "vanna[chromadb,google]<2.0" google-cloud-bigquery db-dtypes python-dotenv
```

This installs:
- `vanna<2.0` — the 0.x version this module is written for. Vanna 2.0 is a full rewrite with a different API and will not work with this code.
- `vanna[chromadb]` — local vector store
- `vanna[google]` — Google Gemini LLM support
- `google-cloud-bigquery` — BigQuery connector
- `db-dtypes` — required for BigQuery pandas type handling
- `python-dotenv` — reads your `.env` file

Takes approximately 3–5 minutes.

---

### Step 2.5 — Verify the installation

```bash
python -c "
from vanna.google import GoogleGeminiChat
from vanna.chromadb import ChromaDB_VectorStore
from google.cloud import bigquery
import dotenv
print('All imports OK')
"
```

Expected output:
```
All imports OK
```

If you see `ModuleNotFoundError`, paste the error before continuing.

---

### Step 2.6 — Add vanna_env to .gitignore

```bash
echo "vanna_env/" >> .gitignore
```

---

## Part 3: Configure Environment Variables

### Step 3.1 — Create the .env file

In VS Code Explorer, right-click the project root (`ga4_dbt`) and select **New File**. Name it `.env`.

Paste this content exactly:

```
GEMINI_API_KEY=paste_your_key_here
GCP_PROJECT_ID=ga4-analysis-492300
BQ_KEYFILE_PATH=/Users/harshpunjabi/Documents/ga4_dbt/dbt_service_account.json
```

- `GEMINI_API_KEY` — paste the key from Step 1.1
- `GCP_PROJECT_ID` — already filled in for you from your dbt logs
- `BQ_KEYFILE_PATH` — already correct, leave it exactly as shown

Save with `Cmd+S`.

**This is the only file where you paste anything manually. The Python scripts read from here automatically.**

---

### Step 3.2 — Update .gitignore

Open `.gitignore` and confirm these lines are present. Add any that are missing:

```
# Credentials
.env
dbt_service_account.json

# Vanna
vanna_env/
vanna/chroma_db/
```

Do not add `*.json` — that would block `packages.json`.

---

## Part 4: Create the Vanna Folder and Files

In VS Code Explorer, right-click the project root and select **New Folder**. Name it `vanna`.

All four files below go inside `vanna/`. Right-click the `vanna` folder, select **New File** for each one.

---

### Step 4.1 — vanna/config.py

```python
import os
import sys
sys.path.append(os.path.dirname(os.path.abspath(__file__)))

from dotenv import load_dotenv
from vanna.google import GoogleGeminiChat
from vanna.chromadb import ChromaDB_VectorStore

# Navigate one level up from vanna/ to find .env in the project root
load_dotenv(os.path.join(os.path.dirname(os.path.abspath(__file__)), '..', '.env'))


class GA4Vanna(ChromaDB_VectorStore, GoogleGeminiChat):
    """
    Custom Vanna class combining:
    - ChromaDB_VectorStore: local vector database for training data
    - GoogleGeminiChat: Gemini LLM for SQL generation
    """
    def __init__(self, config=None):
        ChromaDB_VectorStore.__init__(self, config=config)
        GoogleGeminiChat.__init__(self, config=config)


def get_vanna_instance():
    """
    Initialise and return a configured Vanna instance.
    Called at the top of every other Vanna script.
    """
    # Absolute path to chroma_db folder inside vanna/
    chroma_path = os.path.join(os.path.dirname(os.path.abspath(__file__)), 'chroma_db')

    vn = GA4Vanna(config={
        "api_key": os.getenv("GEMINI_API_KEY"),
        "model":   "gemini-1.5-flash",   # stable model available on free tier
        "path":    chroma_path            # ChromaDB key is "path" not "chroma_path"
    })

    # cred_file_path is the correct parameter name — not credentials_file, not dataset_id
    vn.connect_to_bigquery(
        project_id=os.getenv("GCP_PROJECT_ID"),
        cred_file_path=os.getenv("BQ_KEYFILE_PATH")
    )

    return vn
```

---

### Step 4.2 — vanna/test_connection.py

```python
import os
import sys
sys.path.append(os.path.dirname(os.path.abspath(__file__)))

from config import get_vanna_instance

print("Connecting to BigQuery via Vanna...")
vn = get_vanna_instance()

result = vn.run_sql("SELECT COUNT(*) AS row_count FROM dbt_ga4.fct_purchases")
print(f"Connection successful. fct_purchases has {result['row_count'][0]:,} rows.")
```

---

### Step 4.3 — vanna/train_vanna.py

```python
"""
Vanna AI Training Script — ShopStream GA4 Project

Trains Vanna with:
  1. DDL for all three mart tables
  2. Plain-English documentation for each table
  3. 10 question/SQL pairs covering all three business questions

Run once after dbt mart tables are built in BigQuery.
Re-run only if you add tables or change documentation — delete vanna/chroma_db/ first.

How to run:
  source vanna_env/bin/activate
  python vanna/train_vanna.py
"""

import os
import sys
sys.path.append(os.path.dirname(os.path.abspath(__file__)))

from config import get_vanna_instance

print("Initialising Vanna...")
vn = get_vanna_instance()


# =============================================================================
# PART 1: DDL TRAINING
# Tells Vanna the exact column names, data types, and table structure.
# All table references use fully qualified names: dbt_ga4.table_name
# =============================================================================

print("\n[1/3] Training on DDL...")

vn.train(ddl="""
CREATE TABLE dbt_ga4.fct_purchases (
    purchase_id             STRING NOT NULL,
    transaction_id          STRING,
    user_pseudo_id          STRING NOT NULL,
    session_id              STRING,
    purchase_date           DATE,
    purchased_at            TIMESTAMP,
    revenue                 FLOAT64,
    revenue_usd             FLOAT64,
    item_count              INT64,
    total_quantity          INT64,
    device_category         STRING,
    device_operating_system STRING,
    geo_country             STRING,
    traffic_source          STRING,
    traffic_medium          STRING,
    traffic_campaign        STRING
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
    interacted_at       TIMESTAMP,
    device_category     STRING,
    device_operating_system STRING,
    geo_country         STRING,
    traffic_source      STRING,
    traffic_medium      STRING,
    traffic_campaign    STRING
)
PARTITION BY interaction_date
CLUSTER BY event_name, item_category;
""")

vn.train(ddl="""
CREATE TABLE dbt_ga4.fct_sessions (
    session_id               STRING NOT NULL,
    user_pseudo_id           STRING,
    session_date             DATE,
    session_start_at         TIMESTAMP,
    session_end_at           TIMESTAMP,
    session_duration_seconds INT64,
    session_number           INT64,
    session_engaged          INT64,
    total_engagement_ms      INT64,
    event_count              INT64,
    page_view_count          INT64,
    product_view_count       INT64,
    add_to_cart_count        INT64,
    checkout_count           INT64,
    purchase_count           INT64,
    converted                INT64,
    device_category          STRING,
    device_operating_system  STRING,
    device_browser           STRING,
    geo_country              STRING,
    session_source           STRING,
    session_medium           STRING,
    session_campaign         STRING,
    traffic_source           STRING,
    traffic_medium           STRING,
    traffic_campaign         STRING
)
PARTITION BY session_date
CLUSTER BY session_source, device_category;
""")

print("    DDL done (3 tables).")


# =============================================================================
# PART 2: DOCUMENTATION TRAINING
# Tells Vanna what the tables mean in business terms.
# This is what lets Vanna pick the right table for each question type.
# =============================================================================

print("\n[2/3] Training on documentation...")

vn.train(documentation="""
Table: dbt_ga4.fct_purchases
One row per completed purchase transaction on the Google Merchandise Store.
Primary table for revenue analysis.
Use for: total revenue, average order value, transaction counts,
revenue by traffic channel, revenue by device, revenue by country.
The revenue column is total transaction value in native currency.
revenue_usd is total transaction value in USD.
Join to dbt_ga4.fct_product_interactions on session_id to see item-level detail.
Always use the fully qualified table name dbt_ga4.fct_purchases in SQL.
""")

vn.train(documentation="""
Table: dbt_ga4.fct_product_interactions
One row per product-level event. Covers four event types:
view_item, add_to_cart, begin_checkout, purchase.
Use for: purchase funnel analysis, product view counts, add-to-cart rates,
product conversion rates, category performance.
funnel_step orders events: 1=view_item, 2=add_to_cart, 3=begin_checkout, 4=purchase.
revenue is only populated for purchase events (funnel_step=4).
To calculate conversion rate: SAFE_DIVIDE(sessions at step 4, sessions at step 1).
Always use the fully qualified table name dbt_ga4.fct_product_interactions in SQL.
""")

vn.train(documentation="""
Table: dbt_ga4.fct_sessions
One row per user session. Sessions are reconstructed from GA4 events.
Use for: session counts, conversion rates by channel, engagement metrics,
traffic source analysis, device breakdowns.
converted = 1 if the session resulted in a purchase, 0 if not.
Session conversion rate = SUM(converted) / COUNT(session_id).
session_source and session_medium are session-level attribution (use for channel performance).
traffic_source and traffic_medium are user-level first-touch attribution (use for acquisition analysis).
traffic_medium='(none)' means direct traffic.
Always use the fully qualified table name dbt_ga4.fct_sessions in SQL.
""")

print("    Documentation done (3 tables).")


# =============================================================================
# PART 3: QUESTION/SQL PAIR TRAINING
# Shows Vanna exactly how business questions map to correct SQL for this schema.
# =============================================================================

print("\n[3/3] Training on question/SQL pairs...")

# Business Question 1: Acquisition

vn.train(
    question="What is the total revenue by traffic source?",
    sql="""
    SELECT
        traffic_source,
        traffic_medium,
        ROUND(SUM(revenue), 2)         AS total_revenue,
        COUNT(DISTINCT user_pseudo_id) AS unique_purchasers,
        COUNT(*)                       AS total_transactions
    FROM dbt_ga4.fct_purchases
    GROUP BY 1, 2
    ORDER BY total_revenue DESC
    """
)

vn.train(
    question="What is the session conversion rate by traffic medium?",
    sql="""
    SELECT
        session_medium,
        COUNT(*)                            AS total_sessions,
        SUM(converted)                      AS converted_sessions,
        ROUND(SUM(converted) / COUNT(*), 4) AS conversion_rate
    FROM dbt_ga4.fct_sessions
    WHERE session_medium IS NOT NULL
    GROUP BY session_medium
    ORDER BY conversion_rate DESC
    """
)

vn.train(
    question="Which traffic sources drive the most sessions?",
    sql="""
    SELECT
        session_source,
        session_medium,
        COUNT(DISTINCT session_id)     AS sessions,
        COUNT(DISTINCT user_pseudo_id) AS unique_users,
        SUM(converted)                 AS conversions
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
        DATE_TRUNC(purchase_date, WEEK) AS week,
        ROUND(SUM(revenue), 2)          AS weekly_revenue,
        COUNT(DISTINCT user_pseudo_id)  AS unique_purchasers,
        COUNT(*)                        AS transactions,
        ROUND(AVG(revenue), 2)          AS avg_order_value
    FROM dbt_ga4.fct_purchases
    GROUP BY week
    ORDER BY week
    """
)

# Business Question 2: Funnel

vn.train(
    question="How many users reach each step of the purchase funnel?",
    sql="""
    SELECT
        funnel_step,
        event_name                     AS funnel_step_name,
        COUNT(DISTINCT user_pseudo_id) AS unique_users,
        COUNT(DISTINCT session_id)     AS unique_sessions
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
            SAFE_DIVIDE(previous_step_sessions - sessions_at_step, previous_step_sessions),
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

# Business Question 3: Products

vn.train(
    question="What are the top 10 products by revenue?",
    sql="""
    SELECT
        item_name,
        item_category,
        ROUND(SUM(revenue), 2)     AS total_revenue,
        SUM(quantity)              AS units_sold,
        COUNT(DISTINCT session_id) AS purchase_sessions
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
        ROUND(SAFE_DIVIDE(purchases, views), 4) AS view_to_purchase_rate,
        ROUND(SAFE_DIVIDE(adds, views), 4)      AS view_to_cart_rate
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
        ROUND(SUM(revenue), 2)     AS total_revenue,
        COUNT(DISTINCT session_id) AS transactions,
        SUM(quantity)              AS units_sold,
        ROUND(AVG(price), 2)       AS avg_price
    FROM dbt_ga4.fct_product_interactions
    WHERE event_name = 'purchase'
      AND item_category IS NOT NULL
    GROUP BY item_category
    ORDER BY total_revenue DESC
    """
)

print("    Question/SQL pairs done (10 pairs).")

print("""
==============================
Training complete.
  - 3 DDL definitions
  - 3 documentation blocks
  - 10 question/SQL pairs

ChromaDB written to: vanna/chroma_db/

Next step:
  python vanna/vanna_demo.py
==============================
""")
```

---

### Step 4.4 — vanna/vanna_demo.py

```python
"""
Vanna AI Demo — ShopStream GA4 Project

Runs 5 pre-defined questions then lets you enter your own.
Prints generated SQL and the result table for each question.

How to run:
  source vanna_env/bin/activate
  python vanna/vanna_demo.py
"""

import os
import sys
sys.path.append(os.path.dirname(os.path.abspath(__file__)))

from config import get_vanna_instance
import pandas as pd

pd.set_option('display.max_columns', None)
pd.set_option('display.width', 120)
pd.set_option('display.max_rows', 30)

print("Starting Vanna GA4 Demo...")
vn = get_vanna_instance()


def ask(question: str):
    print(f"\n{'='*60}")
    print(f"QUESTION: {question}")
    print('='*60)
    sql = vn.generate_sql(question)
    print(f"\nGenerated SQL:\n{sql}")
    try:
        result = vn.run_sql(sql)
        print(f"\nResult ({len(result)} rows):")
        print(result.to_string(index=False))
    except Exception as e:
        print(f"\nSQL execution failed: {e}")
        print("Check the generated SQL above for errors.")


ask("What is the total revenue by traffic source?")
ask("What is the session conversion rate by traffic medium?")
ask("How many users reach each step of the purchase funnel?")
ask("What are the top 10 products by revenue?")
ask("Which products have the highest view count but lowest purchase rate?")

print("\n" + "="*60)
print("Enter your own question (or press Enter to exit):")
user_question = input("> ").strip()
if user_question:
    ask(user_question)
```

---

## Part 5: Run the Scripts

Make sure `vanna_env` is active before every command here:

```bash
source vanna_env/bin/activate
```

---

### Step 5.1 — Test the BigQuery connection

```bash
python vanna/test_connection.py
```

Expected output:
```
Connecting to BigQuery via Vanna...
Connection successful. fct_purchases has X,XXX rows.
```

**Do not proceed to training if this fails.** Common errors:

| Error | Fix |
|---|---|
| `DefaultCredentialsError` | Check `BQ_KEYFILE_PATH` in `.env` points to the correct file |
| `NotFound: 404` | `dbt_ga4` dataset or `fct_purchases` table does not exist — run `dbt run --target prod` first |
| `Forbidden: 403` | Service account needs BigQuery Data Viewer + BigQuery Job User roles |
| `KeyError: GEMINI_API_KEY` | `.env` file not found — check it is in the project root named `.env` not `.env.txt` |
| `ModuleNotFoundError` | Wrong environment active — run `source vanna_env/bin/activate` |

---

### Step 5.2 — Run training

```bash
python vanna/train_vanna.py
```

Expected output:
```
Initialising Vanna...

[1/3] Training on DDL...
    DDL done (3 tables).

[2/3] Training on documentation...
    Documentation done (3 tables).

[3/3] Training on question/SQL pairs...
    Question/SQL pairs done (10 pairs).

==============================
Training complete.
...
==============================
```

Takes 30–90 seconds. Makes API calls to Gemini for each training item.

After training, `vanna/chroma_db/` will appear containing `chroma.sqlite3`. This is your local vector store.

**Run training once only.** To retrain from scratch: delete `vanna/chroma_db/` then re-run `train_vanna.py`.

---

### Step 5.3 — Run the demo

```bash
python vanna/vanna_demo.py
```

You will see generated SQL and result tables for each question, then a prompt to enter your own.

---

## Part 6: The Teaching Moment

This experiment shows why documentation quality matters.

**Run 1:** Comment out all three `vn.train(documentation=...)` blocks in `train_vanna.py`. Delete `vanna/chroma_db/`. Run training. Ask:
```
What is the conversion rate by channel?
```
The SQL will likely be wrong or query the wrong table.

**Run 2:** Uncomment the documentation blocks. Delete `vanna/chroma_db/`. Retrain. Ask the same question.

Now it uses `dbt_ga4.fct_sessions` with `SUM(converted) / COUNT(session_id)` — because the documentation told it what `converted` means and how conversion rate is calculated.

This is the point: Vanna's SQL accuracy is a direct output of your `_marts.yml` description quality.

---

## Checklist

- [ ] `vanna_env` created and dependencies installed without errors
- [ ] `vanna_env/` added to `.gitignore`
- [ ] `.env` created in project root with `GEMINI_API_KEY` filled in
- [ ] `vanna/chroma_db/` added to `.gitignore`
- [ ] All four files created: `config.py`, `test_connection.py`, `train_vanna.py`, `vanna_demo.py`
- [ ] `test_connection.py` returned a row count
- [ ] `train_vanna.py` completed without errors
- [ ] `vanna/chroma_db/` folder appeared after training
- [ ] `vanna_demo.py` returned results for all five questions
- [ ] Tried at least one custom question
