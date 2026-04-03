# Module 03: dbt Core Setup Guide
## Complete Beginner's Guide

This guide assumes you have never used dbt before. Follow every step in order. Do not skip the verification steps.

---

## What is dbt?

**dbt (data build tool)** is a command-line tool that lets you write SQL `SELECT` statements and dbt handles turning them into tables and views in your data warehouse. It also gives you:

- **Version control** for your SQL (lives in GitHub like code)
- **Testing** to verify your data is correct
- **Documentation** that is always up to date
- **Lineage** so you can see how every table connects

Think of it as "software engineering best practices applied to SQL."

**dbt Core** is the free, open-source version that runs on your local machine. This is what we use in this project.

---

## What You Will Need

Before starting, make sure you have:
- [ ] Python 3.9 or higher installed
- [ ] A Google Cloud account with BigQuery enabled
- [ ] VS Code installed
- [ ] Git installed

**Check your Python version:**
```bash
python --version
# or on some systems:
python3 --version
```
You need 3.9 or higher. If you do not have it, download it from [python.org](https://python.org).

---

## Part 1: Install dbt Core

### Step 1.1 — Create a Virtual Environment

A virtual environment keeps dbt's dependencies separate from the rest of your Python installation. This prevents conflicts.

**Mac / Linux:**
```bash
# Navigate to where you want to store this project
cd ~/Documents
mkdir project4_dbt
cd project4_dbt

# Create the virtual environment
python3 -m venv dbt_env

# Activate it
source dbt_env/bin/activate
```

**Windows (Command Prompt):**
```cmd
cd C:\Users\YourName\Documents
mkdir project4_dbt
cd project4_dbt

python -m venv dbt_env

dbt_env\Scripts\activate
```

**Windows (PowerShell):**
```powershell
cd C:\Users\YourName\Documents
mkdir project4_dbt
cd project4_dbt

python -m venv dbt_env

dbt_env\Scripts\Activate.ps1
```

> **How do you know it worked?** Your terminal prompt will change to show `(dbt_env)` at the start. This means the virtual environment is active.

> **Important:** Every time you open a new terminal window and want to work on this project, you need to activate the virtual environment again using the activate command above.

---

### Step 1.2 — Install dbt with the BigQuery Adapter

dbt needs a specific adapter to connect to each database. We are using BigQuery.

```bash
pip install dbt-bigquery
```

This will install both dbt Core and the BigQuery adapter. It will take 1-2 minutes.

**Verify the installation:**
```bash
dbt --version
```

You should see output like:
```
Core:
  - installed: 1.8.x
  - latest:    1.8.x - Up to date!

Plugins:
  - bigquery: 1.8.x - Up to date!
```

If you see a version number, dbt is installed correctly.

---

## Part 2: Set Up Google Cloud Authentication

dbt needs permission to read and write data in your BigQuery project. We do this using a **service account** — a special Google account that represents an application rather than a person.

### Step 2.1 — Create a Google Cloud Project

If you already have a Google Cloud project from earlier modules, skip to Step 2.2.

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Click the project selector at the top → **New Project**
3. Name it `ga4-analytics-project` (or similar)
4. Click **Create**
5. Wait for the project to be created and select it

### Step 2.2 — Enable the BigQuery API

1. In the left menu, go to **APIs & Services** → **Library**
2. Search for `BigQuery API`
3. Click on it and click **Enable**

### Step 2.3 — Create a Service Account

1. In the left menu, go to **IAM & Admin** → **Service Accounts**
2. Click **+ Create Service Account**
3. Fill in:
   - **Service account name:** `dbt-service-account`
   - **Description:** `Service account for dbt Core`
4. Click **Create and Continue**
5. Under **Grant this service account access to the project**, click **Select a role**
6. Search for `BigQuery` and select **BigQuery Admin**
7. Click **Continue** → **Done**

> **Why BigQuery Admin?** For a learning project this is fine. In a production environment you would use a more restrictive role. BigQuery Admin gives dbt permission to create datasets, tables, and views.

### Step 2.4 — Download the Service Account Key

1. On the Service Accounts page, find the account you just created
2. Click the three dots (⋮) on the right → **Manage keys**
3. Click **Add Key** → **Create new key**
4. Select **JSON** → **Create**
5. A JSON file will download to your computer

**Rename this file** to `dbt_service_account.json` and move it to your project folder (`project4_dbt`).

> **Security note:** This file contains credentials. Never upload it to GitHub. We will set up `.gitignore` to prevent this in Step 3.3.

---

## Part 3: Create the dbt Project

### Step 3.1 — Initialise the Project

With your virtual environment active and inside the `project4_dbt` folder:

```bash
dbt init ga4_ecommerce
```

dbt will ask you a series of questions:

```
Which database adapter would you like to use?
[1] bigquery
Enter a number: 1

Desired authentication method option:
[1] oauth
[2] service_account
Enter a number: 2

project (GCP project id): YOUR_GCP_PROJECT_ID
dataset (the name of your dbt dataset): dbt_ga4
threads: 4
timeout_seconds [300]: 300
location (GCP location): US
```

- For `project`, enter your Google Cloud project ID (find it in the GCP console — it looks like `ga4-analytics-project-123456`)
- For `dataset`, enter `dbt_ga4` — this is where dbt will create your tables in BigQuery
- For `location`, enter `US`

### Step 3.2 — Set the Service Account Key Path

After `dbt init`, dbt creates a `profiles.yml` file in a hidden folder on your system:

- **Mac/Linux:** `~/.dbt/profiles.yml`
- **Windows:** `C:\Users\YourName\.dbt\profiles.yml`

Open this file and update the `keyfile` line to point to your service account JSON:

**Mac/Linux:**
```yaml
ga4_ecommerce:
  target: dev
  outputs:
    dev:
      type: bigquery
      method: service-account
      project: YOUR_GCP_PROJECT_ID
      dataset: dbt_ga4_dev
      keyfile: /Users/YourName/Documents/project4_dbt/dbt_service_account.json
      threads: 4
      timeout_seconds: 300
      location: US
      priority: interactive
    prod:
      type: bigquery
      method: service-account
      project: YOUR_GCP_PROJECT_ID
      dataset: dbt_ga4
      keyfile: /Users/YourName/Documents/project4_dbt/dbt_service_account.json
      threads: 4
      timeout_seconds: 300
      location: US
      priority: interactive
```

**Windows:**
```yaml
ga4_ecommerce:
  target: dev
  outputs:
    dev:
      type: bigquery
      method: service-account
      project: YOUR_GCP_PROJECT_ID
      dataset: dbt_ga4_dev
      keyfile: C:\Users\YourName\Documents\project4_dbt\dbt_service_account.json
      threads: 4
      timeout_seconds: 300
      location: US
      priority: interactive
    prod:
      type: bigquery
      method: service-account
      project: YOUR_GCP_PROJECT_ID
      dataset: dbt_ga4
      keyfile: C:\Users\YourName\Documents\project4_dbt\dbt_service_account.json
      threads: 4
      timeout_seconds: 300
      location: US
      priority: interactive
```

> **Why two targets (dev and prod)?**
> - `dev` writes to `dbt_ga4_dev` — your sandbox for development. It is safe to break things here.
> - `prod` writes to `dbt_ga4` — the "production" dataset that Tableau and Vanna will connect to.
> - During development you always run against `dev`. This protects production from broken models.

### Step 3.3 — Set Up .gitignore

Inside the `ga4_ecommerce` folder that dbt created, open or create a `.gitignore` file and add:

```
# Never commit credentials
*.json
dbt_service_account.json

# dbt generated files
target/
dbt_packages/
logs/

# Python
__pycache__/
*.pyc
dbt_env/
```

---

## Part 4: Configure the dbt Project

### Step 4.1 — Update dbt_project.yml

Navigate into the `ga4_ecommerce` folder. Open `dbt_project.yml` and replace its contents with:

```yaml
name: 'ga4_ecommerce'
version: '1.0.0'
config-version: 2

profile: 'ga4_ecommerce'

model-paths: ["models"]
analysis-paths: ["analyses"]
test-paths: ["tests"]
seed-paths: ["seeds"]
macro-paths: ["macros"]
snapshot-paths: ["snapshots"]

target-path: "target"
clean-targets:
  - "target"
  - "dbt_packages"

models:
  ga4_ecommerce:
    staging:
      +materialized: view
      +schema: staging
    intermediate:
      +materialized: view
      +schema: intermediate
    marts:
      +materialized: table
      +schema: marts
```

**What this does:**
- `staging` models will be created as **views** (no storage cost, rebuilt on every query)
- `intermediate` models will be created as **views**
- `mart` models will be created as **tables** (stored, fast to query)
- Each layer gets its own schema prefix in BigQuery

### Step 4.2 — Create the Folder Structure

Inside the `models` folder, create three subfolders:

```bash
cd ga4_ecommerce
mkdir -p models/staging
mkdir -p models/intermediate
mkdir -p models/marts
mkdir -p macros
mkdir -p seeds
mkdir -p tests
```

Delete the example models that dbt created:
```bash
rm -rf models/example
```

---

## Part 5: Verify Everything Works

### Step 5.1 — Run dbt Debug

This command checks that dbt can connect to BigQuery successfully.

```bash
# Make sure you are inside the ga4_ecommerce folder
cd ga4_ecommerce

dbt debug
```

**What you want to see:**
```
All checks passed!
```

**Common errors and fixes:**

| Error | Fix |
|---|---|
| `Could not find profile named 'ga4_ecommerce'` | Check that the profile name in `dbt_project.yml` matches `profiles.yml` exactly |
| `File not found: dbt_service_account.json` | Check the `keyfile` path in `profiles.yml` — it must be the full absolute path |
| `403 Access Denied` | Check that your service account has BigQuery Admin role |
| `Project not found` | Check your GCP project ID — copy it exactly from the GCP console |

### Step 5.2 — Create a Test Model

Create a simple test model to verify dbt can write to BigQuery.

Create the file `models/staging/stg_test.sql`:
```sql
SELECT
  'dbt is working' AS message,
  CURRENT_TIMESTAMP() AS run_at
```

Run it:
```bash
dbt run --select stg_test
```

You should see:
```
1 of 1 START sql view model dbt_ga4_dev_staging.stg_test .................. [RUN]
1 of 1 OK created sql view model dbt_ga4_dev_staging.stg_test ............. [OK]
```

Go to BigQuery and check that a dataset called `dbt_ga4_dev_staging` was created with a view called `stg_test`. If it is there, dbt is working correctly.

Delete the test file before moving on:
```bash
rm models/staging/stg_test.sql
```

---

## Part 6: VS Code Setup (Recommended)

### Step 6.1 — Install the dbt Power User Extension

1. Open VS Code
2. Click the Extensions icon (or press `Ctrl+Shift+X`)
3. Search for `dbt Power User`
4. Install it

This extension gives you:
- SQL syntax highlighting for dbt models
- Auto-complete for dbt functions
- Preview of compiled SQL
- Model lineage view

### Step 6.2 — Open the Project in VS Code

```bash
code .
```

Or open VS Code and use **File → Open Folder** to open the `ga4_ecommerce` folder.

---

## Setup Complete

You now have:
- [ ] dbt Core installed with the BigQuery adapter
- [ ] A Google Cloud service account with BigQuery access
- [ ] A dbt project initialised with dev and prod targets
- [ ] The correct folder structure for staging, intermediate, and marts
- [ ] Verified that dbt can connect and write to BigQuery

**Your project folder should look like this:**
```
ga4_ecommerce/
├── .gitignore
├── dbt_project.yml
├── models/
│   ├── staging/      (empty for now)
│   ├── intermediate/ (empty for now)
│   └── marts/        (empty for now)
├── macros/
├── seeds/
└── tests/
```

When you are ready, move on to [../docs/04_dbt_staging.md](../docs/04_dbt_staging.md).
