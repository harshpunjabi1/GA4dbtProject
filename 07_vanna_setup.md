# Module 07: Vanna AI Setup Guide
## Complete Beginner's Guide

---

## What is Vanna AI?

Vanna is an open-source Python library that lets users ask questions about your database in plain English and get accurate SQL back — and the results.

It works using **RAG (Retrieval-Augmented Generation)**:
1. You train it with your table schemas, documentation, and example SQL queries
2. When a user asks a question, Vanna finds the most relevant training examples
3. It sends those examples + the question to an LLM
4. The LLM generates accurate SQL based on the context
5. Vanna runs the SQL against your database and returns the result

The quality of Vanna's SQL is directly tied to the quality of your training data. This is why the documentation in your `schema.yml` files matters.

---

## What You Will Need

- [ ] Python virtual environment (the same one used for dbt, or a new one)
- [ ] Google AI Studio API key (free)
- [ ] BigQuery service account JSON (same one used for dbt)
- [ ] Your dbt mart tables built in BigQuery (prod dataset)

---

## Part 1: Get a Google Gemini API Key

### Step 1.1 — Go to Google AI Studio

1. Go to [aistudio.google.com](https://aistudio.google.com)
2. Sign in with your Google account (the same one used for BigQuery)
3. Click **Get API key** in the left menu
4. Click **Create API key**
5. Select your existing Google Cloud project
6. Copy the API key and save it somewhere safe

> **This key is free.** The free tier gives you up to 250 requests per day on Gemini 2.5 Flash — more than enough for this project and student use.

> **Security:** Treat this key like a password. Never paste it directly into code files that go to GitHub. We will use environment variables.

---

## Part 2: Set Up the Python Environment

### Step 2.1 — Activate Your Virtual Environment

**Mac/Linux:**
```bash
cd ~/Documents/project4_dbt
source dbt_env/bin/activate
```

**Windows:**
```cmd
cd C:\Users\YourName\Documents\project4_dbt
dbt_env\Scripts\activate
```

### Step 2.2 — Install Vanna and Dependencies

```bash
pip install vanna
pip install vanna[google]
pip install chromadb
pip install google-cloud-bigquery
pip install db-dtypes
pip install pandas
```

This will take 2-3 minutes.

**Verify the installation:**
```python
python -c "import vanna; print('Vanna installed successfully')"
```

---

## Part 3: Set Up Environment Variables

We store the API key and project details as environment variables so they never appear in code.

### Step 3.1 — Create a .env File

In your project root folder, create a file called `.env`:

```
GEMINI_API_KEY=your_api_key_here
GCP_PROJECT_ID=your_gcp_project_id
BQ_DATASET=dbt_ga4
BQ_KEYFILE_PATH=/full/path/to/dbt_service_account.json
```

Replace each value with your actual credentials.

### Step 3.2 — Add .env to .gitignore

Make sure `.env` is in your `.gitignore` file:

```
.env
*.json
dbt_service_account.json
```

### Step 3.3 — Install python-dotenv

```bash
pip install python-dotenv
```

---

## Part 4: Create the Vanna Configuration

Create the folder `vanna/` in your project root. Create the file `vanna/config.py`:

```python
import os
from dotenv import load_dotenv
from vanna.google import GoogleGeminiChat
from vanna.chromadb import ChromaDB_VectorStore

load_dotenv()

class GA4Vanna(ChromaDB_VectorStore, GoogleGeminiChat):
    """
    Custom Vanna class combining:
    - ChromaDB as the local vector store (stores training data)
    - Google Gemini as the LLM (generates SQL)
    """
    pass


def get_vanna_instance():
    """
    Initialise and return a configured Vanna instance.
    Call this at the start of your training and query scripts.
    """
    vn = GA4Vanna(config={
        "api_key": os.getenv("GEMINI_API_KEY"),
        "model": "gemini-2.5-flash",
        "chroma_path": "./vanna/chroma_db"  # local vector store location
    })

    vn.connect_to_bigquery(
        project_id=os.getenv("GCP_PROJECT_ID"),
        dataset_id=os.getenv("BQ_DATASET"),
        credentials_file=os.getenv("BQ_KEYFILE_PATH")
    )

    return vn
```

---

## Part 5: Verify the Connection

Create `vanna/test_connection.py`:

```python
from config import get_vanna_instance

print("Connecting to BigQuery via Vanna...")
vn = get_vanna_instance()

# Try a simple query to verify the connection works
result = vn.run_sql("SELECT COUNT(*) AS row_count FROM fct_purchases")
print(f"Connection successful! fct_purchases has {result['row_count'][0]} rows.")
```

Run it from the `vanna/` directory:
```bash
cd vanna
python test_connection.py
```

You should see a row count printed. If you get an error, check:
- Your `.env` file has the correct values
- The `dbt_ga4` dataset exists in BigQuery (run `dbt run --target prod` if not)
- The service account JSON path is correct and absolute

---

## Understanding ChromaDB

ChromaDB is a local vector database that Vanna uses to store your training data. When you train Vanna with DDL, documentation, and SQL examples, ChromaDB stores them as vector embeddings.

When a user asks a question, Vanna:
1. Converts the question to a vector embedding
2. Searches ChromaDB for the most similar training examples
3. Passes those examples to Gemini as context

The ChromaDB files are stored in `./vanna/chroma_db/` on your local machine. This folder is **not uploaded to GitHub** (add it to `.gitignore`). Each student will build their own local ChromaDB by running the training script.

Add to `.gitignore`:
```
vanna/chroma_db/
```

---

## Setup Complete

You now have:
- [ ] Google Gemini API key from AI Studio
- [ ] Vanna and dependencies installed
- [ ] `.env` file with credentials (not in GitHub)
- [ ] `vanna/config.py` with the Vanna class configuration
- [ ] Verified connection to BigQuery

When you are ready, move on to [../docs/08_vanna_training.md](../docs/08_vanna_training.md).
