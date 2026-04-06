# Modules 07 & 08: Vanna AI — Complete Setup, Training, and Demo Guide

---

## Before You Start: Read This Carefully

This guide assumes the following about your environment. If anything does not match, stop and fix it before continuing.

**Your machine:** Mac (macOS)

**Your IDE:** VS Code. All commands in this guide are run in the **VS Code integrated terminal** (`Terminal > New Terminal`). Do not use Jupyter Notebook for this module. Do not use the macOS Terminal app separately — use the terminal inside VS Code so your working directory is always the project root.

**Your project root:** `/Users/harshpunjabi/Documents/ga4_dbt`
This is the folder that contains `dbt_project.yml`. Every command in this guide is run from this folder unless explicitly stated otherwise. When you open VS Code, open this folder (`File > Open Folder`), not a subfolder.

**Your service account:** `/Users/harshpunjabi/Documents/ga4_dbt/dbt_service_account.json`
This file already exists. You do not need to create it.

**Your BigQuery dataset:** `dbt_ga4`
Your dbt mart tables (`fct_purchases`, `fct_sessions`, `fct_product_interactions`) must already be built in BigQuery before you run Vanna. If you have not run `dbt run --target prod` yet, do that first.

**Your virtual environment:** `dbt_env` — already created inside your project root from earlier modules.

---

## What Vanna AI Does

Vanna lets a user ask a plain-English question and get back a SQL query — and the query result. It works using RAG (Retrieval-Augmented Generation):

1. You train it once with your table schemas, plain-English descriptions, and example SQL queries
2. When a question is asked, Vanna converts it to a vector embedding and finds the most similar training examples in ChromaDB
3. Those examples + the question are sent to Google Gemini
4. Gemini generates SQL that fits your schema
5. Vanna runs the SQL against BigQuery and returns a pandas DataFrame

**ChromaDB** is a local vector database that lives on your machine inside `vanna/chroma_db/`. It is not a cloud service. It is not uploaded to GitHub. Every student builds their own ChromaDB by running the training script.

**Google Gemini** is the LLM. You access it via a free API key from Google AI Studio. Your data is sent to Google's API — do not train Vanna with anything confidential.

---

## Final Project File Structure

After completing both modules, your project will look like this. Files marked `[NEW]` are created in this guide. Files marked `[EXISTS]` already exist from earlier modules.

```
ga4_dbt/                                ← project root, open this in VS Code
│
├── dbt_project.yml                     [EXISTS]
├── packages.yml                        [EXISTS]
├── dbt_service_account.json            [EXISTS] ← gitignored, never commit this
├── .env                                [NEW]    ← gitignored, never commit this
├── .gitignore                          [EXISTS, UPDATED]
│
├── dbt_env/                            [EXISTS] ← your Python virtual environment
│
├── models/                             [EXISTS]
│   └── marts/
│       ├── _marts.yml
│       ├── fct_purchases.sql
│       ├── fct_sessions.sql
│       └── fct_product_interactions.sql
│
└── vanna/                              [NEW FOLDER]
    ├── config.py                       [NEW]
    ├── test_connection.py              [NEW]
    ├── train_vanna.py                  [NEW]
    ├── vanna_demo.py                   [NEW]
    └── chroma_db/                      [AUTO-CREATED when training runs, gitignored]
```

---

## Part 1: Get a Google Gemini API Key

### Step 1.1 — Create the key

1. Go to [https://aistudio.google.com](https://aistudio.google.com) in your browser
2. Sign in with the same Google account that owns your BigQuery project
3. In the left sidebar, click **Get API key**
4. Click **Create API key**
5. Select your existing Google Cloud project from the dropdown
6. Click **Create API key in existing project**
7. Copy the key — it looks like `AIzaSy...`
8. Save it somewhere safe (password manager, notes). You cannot retrieve it again after closing this screen, but you can always create a new one.

You do not need to enable billing for Gemini API access. The free tier is sufficient for this project.

---

## Part 2: Set Up the Python Environment

### Step 2.1 — Open your project in VS Code

Open VS Code. Go to `File > Open Folder`. Select `/Users/harshpunjabi/Documents/ga4_dbt`. Click **Open**.

Open the integrated terminal: `Terminal > New Terminal`. You should see a prompt ending in `ga4_dbt`.

Verify your working directory:
```bash
pwd
```
Expected output:
```
/Users/harshpunjabi/Documents/ga4_dbt
```

If it shows anything else, stop and navigate there:
```bash
cd /Users/harshpunjabi/Documents/ga4_dbt
```

### Step 2.2 — Activate the virtual environment

```bash
source dbt_env/bin/activate
```

Your terminal prompt should now show `(dbt_env)` at the start. Every command from this point forward assumes the virtual environment is active. If you close VS Code and reopen it, you need to run this activation command again.

### Step 2.3 — Install Vanna and its dependencies

Run these commands one at a time. Wait for each to finish before running the next.

```bash
pip install vanna
```
```bash
pip install "vanna[google]"
```
```bash
pip install chromadb
```
```bash
pip install google-cloud-bigquery
```
```bash
pip install db-dtypes
```
```bash
pip install pandas
```
```bash
pip install python-dotenv
```

The quotes around `"vanna[google]"` are required on Mac. Without them the shell misinterprets the brackets and the install fails silently.

Total install time is approximately 3–5 minutes depending on your connection.

### Step 2.4 — Verify the installs

```bash
python -c "import vanna; import chromadb; import google.cloud.bigquery; print('All dependencies installed successfully')"
```

Expected output:
```
All dependencies installed successfully
```

If you get a `ModuleNotFoundError`, re-run the install step for the missing package.

---

## Part 3: Configure Environment Variables

### Step 3.1 — Create the .env file

In VS Code, go to the Explorer panel on the left. Right-click on the `ga4_dbt` folder (the root) and select **New File**. Name it `.env`.

Add the following content exactly, replacing the two placeholder values:

```
GEMINI_API_KEY=your_gemini_api_key_here
GCP_PROJECT_ID=your_gcp_project_id_here
BQ_KEYFILE_PATH=/Users/harshpunjabi/Documents/ga4_dbt/dbt_service_account.json
```

- `GEMINI_API_KEY` — the key you copied from Google AI Studio in Step 1.1
- `GCP_PROJECT_ID` — your Google Cloud project ID. Find this in the [Google Cloud Console](https://console.cloud.google.com) in the top navigation bar. It looks like `my-project-123456`
- `BQ_KEYFILE_PATH` — leave this exactly as shown. This path is hardcoded to your machine

Save the file with `Cmd+S`.

> **What is the GCP_PROJECT_ID?** It is not your project name. It is the project ID shown in the Google Cloud Console header. It usually contains hyphens and numbers.

### Step 3.2 — Update .gitignore

Open `.gitignore` in VS Code. Add these lines if they are not already there:

```
# Credentials and secrets
.env
dbt_service_account.json

# Vanna local vector store
vanna/chroma_db/
```

Do not add `*.json` as a blanket rule — that would block `packages.json` and other non-sensitive files.

---

## Part 4: Create the Vanna Folder and Files

### Step 4.1 — Create the vanna/ folder

In VS Code Explorer, right-click the project root and select **New Folder**. Name it `vanna`.

You will create four files inside this folder. All four files below go inside `vanna/`, not in the project root.

---

### Step 4.2 — Create vanna/config.py

In VS Code Explorer, right-click the `vanna` folder and select **New File**. Name it `config.py`.

Paste this content exactly:

```python
import os
import sys
sys.path.append(os.path.dirname(os.path.abspath(__file__)))

from dotenv import load_dotenv
from vanna.google import GoogleGeminiChat
from vanna.chromadb import ChromaDB_VectorStore

# Load environment variables from .env in the project root
load_dotenv(os.path.join(os.path.dirname(os.path.abspath(__file__)), '..', '.env'))


class GA4Vanna(ChromaDB_VectorStore, GoogleGeminiChat):
    """
    Custom Vanna class that combines:
    - ChromaDB_VectorStore: local vector database for storing training data
    - GoogleGeminiChat: Gemini LLM for generating SQL from retrieved examples
    """
    def __init__(self, config=None):
        ChromaDB_VectorStore.__init__(self, config=config)
        GoogleGeminiChat.__init__(self, config=config)


def get_vanna_instance():
    """
    Initialise and return a configured Vanna instance connected to BigQuery.
    Call this at the start of every script that uses Vanna.
    """
    chroma_path = os.path.join(os.path.dirname(os.path.abspath(__file__)), 'chroma_db')

    vn = GA4Vanna(config={
        "api_key": os.getenv("GEMINI_API_KEY"),
        "model":   "gemini-2.0-flash",
        "path":    chroma_path       # ChromaDB key is "path", not "chroma_path"
    })

    # connect_to_bigquery parameters:
    # - project_id: your GCP project ID (not the dataset name)
    # - cred_file_path: absolute path to service account JSON
    # - No dataset_id parameter exists. Dataset is specified in SQL as dbt_ga4.table_name
    vn.connect_to_bigquery(
        project_id=os.getenv("GCP_PROJECT_ID"),
        cred_file_path=os.getenv("BQ_KEYFILE_PATH")
    )

    return vn
```

Save the file.

**Why `load_dotenv` points one level up:** `config.py` lives inside `vanna/`. The `.env` file lives in the project root, one level above. The `os.path.join(..., '..', '.env')` navigates up to find it reliably regardless of where the script is called from.

**Why `"path"` not `"chroma_path"`:** This is confirmed from the ChromaDB_VectorStore source code. The config dict key is `"path"`. Using `"chroma_path"` silently falls back to the current working directory and your training data ends up scattered.

**Why no `dataset_id` in `connect_to_bigquery`:** That parameter does not exist in Vanna's BigQuery connector. Dataset is handled by using fully qualified table names in every SQL query (`dbt_ga4.fct_purchases`, not just `fct_purchases`). All SQL in the training and demo scripts already does this.

---

### Step 4.3 — Create vanna/test_connection.py

Right-click the `vanna` folder, select **New File**, name it `test_connection.py`.

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

Save the file.

---

### Step 4.4 — Create vanna/train_vanna.py

Right-click the `vanna` folder, select **New File**, name it `train_vanna.py`.

```python
"""
Vanna AI Training Script — ShopStream GA4 Project

Trains Vanna with:
  1. DDL for all three mart tables
  2. Plain-English documentation for each table
  3. 10 question/SQL pairs covering all three business questions

Run once after your dbt mart tables are built in BigQuery.
Re-run if you add tables or update descriptions.

Run from the project root (ga4_dbt/):
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
# DDL uses fully qualified table names: dbt_ga4.table_name
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
    device_category     STRING,
    traffic_source      STRING,
    traffic_medium      STRING
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
    traffic_source           STRING,
    traffic_medium           STRING,
    traffic_campaign         STRING,
    total_engagement_ms      INT64
)
PARTITION BY session_date
CLUSTER BY traffic_source, device_category;
""")

print("    DDL training done (3 tables).")


# =============================================================================
# PART 2: DOCUMENTATION TRAINING
# Tells Vanna what the tables mean in business terms, not just technically.
# This is what allows Vanna to pick the right table for each question.
# The "Always use fully qualified name" line steers Gemini to write correct SQL.
# =============================================================================

print("\n[2/3] Training on documentation...")

vn.train(documentation="""
Table: dbt_ga4.fct_purchases
One row per completed purchase transaction on the Google Merchandise Store.
This is the primary table for revenue analysis.
Use this table for: total revenue, average order value (AOV), transaction counts,
revenue by traffic channel, revenue by device type, revenue by country.
The revenue column is the total transaction value in USD.
Join to dbt_ga4.fct_product_interactions on session_id to see the products in each purchase.
Always use the fully qualified table name dbt_ga4.fct_purchases in SQL queries.
""")

vn.train(documentation="""
Table: dbt_ga4.fct_product_interactions
One row per product-level event. Covers four event types: view_item, add_to_cart,
begin_checkout, and purchase — wherever a product is involved.
Use this table for: purchase funnel analysis, product view counts, add-to-cart rates,
product-level conversion rates, and category performance.
The funnel_step column orders events numerically: 1=view_item, 2=add_to_cart,
3=begin_checkout, 4=purchase.
Revenue is only populated for purchase events (funnel_step = 4 or event_name = 'purchase').
To calculate overall conversion rate: SAFE_DIVIDE(sessions at step 4, sessions at step 1).
Always use the fully qualified table name dbt_ga4.fct_product_interactions in SQL queries.
""")

vn.train(documentation="""
Table: dbt_ga4.fct_sessions
One row per user session. A session is a grouped sequence of events from one user visit.
Use this table for: total session counts, session-level conversion rates by channel,
engagement metrics, traffic source breakdown, and device category splits.
The converted column is 1 if the session ended in a purchase, 0 if not.
Session conversion rate = SUM(converted) / COUNT(session_id).
traffic_source and traffic_medium describe the acquisition channel.
Example: traffic_source='google' and traffic_medium='organic' means organic Google search.
traffic_medium='(none)' means direct traffic.
Always use the fully qualified table name dbt_ga4.fct_sessions in SQL queries.
""")

print("    Documentation training done (3 tables).")


# =============================================================================
# PART 3: QUESTION/SQL PAIR TRAINING
# Shows Vanna how real business questions map to correct SQL for this schema.
# Covers all 3 business questions from Module 00.
# =============================================================================

print("\n[3/3] Training on question/SQL pairs...")

# --- Business Question 1: Acquisition ---

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

# --- Business Question 2: Funnel ---

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

# --- Business Question 3: Products ---

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

print("    Question/SQL training done (10 pairs).")

print("""
==============================
Training complete.
Vanna has been trained on:
  - 3 DDL definitions
  - 3 documentation blocks
  - 10 question/SQL pairs

ChromaDB data written to: vanna/chroma_db/

Next step: run the demo.
    python vanna/vanna_demo.py
==============================
""")
```

Save the file.

---

### Step 4.5 — Create vanna/vanna_demo.py

Right-click the `vanna` folder, select **New File**, name it `vanna_demo.py`.

```python
"""
Vanna AI Demo Script — ShopStream GA4 Project

Asks five pre-defined questions then prompts you to enter your own.
Prints the generated SQL and the query result for each question.

Run from the project root (ga4_dbt/):
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
    """Ask Vanna a question. Prints generated SQL then the result DataFrame."""
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


# Pre-defined demo questions — one per business question category
ask("What is the total revenue by traffic source?")
ask("What is the session conversion rate by traffic medium?")
ask("How many users reach each step of the purchase funnel?")
ask("What are the top 10 products by revenue?")
ask("Which products have the highest view count but lowest purchase rate?")

# Interactive prompt
print("\n" + "="*60)
print("Enter your own question (or press Enter to exit):")
user_question = input("> ").strip()
if user_question:
    ask(user_question)
```

Save the file.

---

## Part 5: Run the Scripts

All commands below are run in the VS Code integrated terminal from the project root (`ga4_dbt/`). The virtual environment must be active — you should see `(dbt_env)` in your prompt.

### Step 5.1 — Test the BigQuery connection

```bash
python vanna/test_connection.py
```

Expected output:
```
Connecting to BigQuery via Vanna...
Connection successful. fct_purchases has 4,305 rows.
```

The row count will match whatever is in your `fct_purchases` table.

**If you get an error here, do not proceed to training.** Fix the connection first.

Common errors and fixes:

| Error | Fix |
|---|---|
| `google.auth.exceptions.DefaultCredentialsError` | Check that `BQ_KEYFILE_PATH` in `.env` is correct and the file exists at that path |
| `google.api_core.exceptions.NotFound: 404` | The `dbt_ga4` dataset or `fct_purchases` table does not exist. Run `dbt run --target prod` first |
| `google.api_core.exceptions.Forbidden: 403` | The service account does not have BigQuery permissions. It needs BigQuery Data Viewer + BigQuery Job User |
| `KeyError: 'GEMINI_API_KEY'` | The `.env` file is not being found. Check the file exists in the project root and is named `.env` (not `.env.txt`) |
| `ModuleNotFoundError` | The virtual environment is not active. Run `source dbt_env/bin/activate` |

---

### Step 5.2 — Run the training script

```bash
python vanna/train_vanna.py
```

This will print progress as it trains each section:

```
Initialising Vanna...

[1/3] Training on DDL...
    DDL training done (3 tables).

[2/3] Training on documentation...
    Documentation training done (3 tables).

[3/3] Training on question/SQL pairs...
    Question/SQL training done (10 pairs).

==============================
Training complete.
...
==============================
```

Training makes API calls to Google Gemini to generate embeddings. It will take approximately 30–60 seconds.

After training, a new folder `vanna/chroma_db/` will appear in your project. This is your local vector store. It contains a file called `chroma.sqlite3`. Do not delete it — re-running training will add duplicate entries to it, not replace them cleanly.

**You only need to run training once.** If you want to retrain from scratch (for example, after changing documentation), delete the `vanna/chroma_db/` folder first, then re-run `train_vanna.py`.

---

### Step 5.3 — Run the demo

```bash
python vanna/vanna_demo.py
```

For each question you will see the generated SQL followed by the result table. After the five pre-defined questions, you will be prompted to enter your own question.

Example output for one question:

```
============================================================
QUESTION: What is the total revenue by traffic source?
============================================================

Generated SQL:
SELECT
    traffic_source,
    traffic_medium,
    ROUND(SUM(revenue), 2) AS total_revenue,
    ...
FROM dbt_ga4.fct_purchases
GROUP BY 1, 2
ORDER BY total_revenue DESC

Result (12 rows):
 traffic_source traffic_medium  total_revenue  unique_purchasers  total_transactions
         google        organic       24312.45                892                1043
          (direct)        (none)     18740.21                712                 834
...
```

---

## Part 6: The Teaching Moment — Why Documentation Quality Matters

This experiment demonstrates the core lesson of this module.

**Step 1:** Before running training, comment out the three `vn.train(documentation=...)` blocks in `train_vanna.py`. Delete `vanna/chroma_db/`. Run training without documentation. Then run the demo and ask:

```
What is the conversion rate by channel?
```

Note the SQL generated — it will likely be wrong or generic.

**Step 2:** Uncomment the documentation blocks. Delete `vanna/chroma_db/`. Run training again with full documentation. Ask the same question.

The SQL should now correctly use `dbt_ga4.fct_sessions` with `SUM(converted) / COUNT(session_id)` — because the documentation explicitly explained what `converted` means and how conversion rate is calculated.

This is the point: Vanna's SQL quality is a direct output of the quality of your `_marts.yml` descriptions.

---

## Checklist

Work through this before declaring the module complete:

- [ ] `.env` file created in project root with all three values filled in
- [ ] `vanna/chroma_db/` added to `.gitignore`
- [ ] `dbt_service_account.json` in `.gitignore`
- [ ] All four files created: `config.py`, `test_connection.py`, `train_vanna.py`, `vanna_demo.py`
- [ ] `test_connection.py` returned a row count successfully
- [ ] `train_vanna.py` completed without errors
- [ ] `vanna/chroma_db/` folder appeared after training
- [ ] `vanna_demo.py` returned results for all five demo questions
- [ ] Tried at least one custom question of your own
