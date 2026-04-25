# Module 03: dbt Core Setup Guide
## Complete Beginner's Guide

This guide assumes you have never used dbt before. Follow every step in order. Do not skip the verification steps.

Every step is labelled clearly:
- 🖥️ **TERMINAL** — type this into your terminal
- 📝 **VS CODE** — do this in VS Code
- 🌐 **BROWSER** — do this in your web browser

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

---

## Part 1: Install dbt Core

### Step 1.1 — Check your Python version

🖥️ **TERMINAL**
```bash
python --version
# or on some systems:
python3 --version
```

You need 3.9 or higher. If you do not have it, download it from [python.org](https://python.org).

---

### Step 1.2 — Create a project folder and virtual environment

🖥️ **TERMINAL**

**Mac / Linux:**
```bash
cd ~/Documents
mkdir project4_dbt
cd project4_dbt

python3 -m venv dbt_env
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

> **How do you know it worked?** Your terminal prompt will change to show `(dbt_env)` at the start.

> **Important:** Every time you open a new terminal window to work on this project, you must activate the virtual environment again before doing anything else.

---

### Step 1.3 — Install dbt with the BigQuery adapter

🖥️ **TERMINAL**
```bash
pip install dbt-bigquery
```

This installs both dbt Core and the BigQuery adapter. It takes 1-2 minutes.

**Verify the installation:**

🖥️ **TERMINAL**
```bash
dbt --version
```

You should see something like:
```
Core:
  - installed: 1.8.x
  - latest:    1.8.x - Up to date!

Plugins:
  - bigquery: 1.8.x - Up to date!
```

---

## Part 2: Set Up Google Cloud Authentication

### Step 2.1 — Create a Google Cloud Project

🌐 **BROWSER** — [console.cloud.google.com](https://console.cloud.google.com)

If you already have a GCP project from a previous module, skip to Step 2.2.

1. Click the project selector at the top → **New Project**
2. Name it `ga4-analytics-project`
3. Click **Create** and wait for it to finish
4. Select the new project from the project selector

---

### Step 2.2 — Enable the BigQuery API

🌐 **BROWSER** — still in the GCP console

1. Left menu → **APIs & Services** → **Library**
2. Search for `BigQuery API`
3. Click it → click **Enable**

---

### Step 2.3 — Create a Service Account

🌐 **BROWSER** — still in the GCP console

1. Left menu → **IAM & Admin** → **Service Accounts**
2. Click **+ Create Service Account**
3. Fill in:
   - **Service account name:** `dbt-service-account`
   - **Description:** `Service account for dbt Core`
4. Click **Create and Continue**
5. Click **Select a role** → search for `BigQuery` → select **BigQuery Admin**
6. Click **Continue** → **Done**

---

### Step 2.4 — Download the Service Account Key

🌐 **BROWSER** — still in the GCP console

1. On the Service Accounts page, find `dbt-service-account`
2. Click the three dots (⋮) on the right → **Manage keys**
3. Click **Add Key** → **Create new key**
4. Select **JSON** → **Create**
5. A JSON file downloads to your computer

**Rename it** to `dbt_service_account.json` and move it into the `project4_dbt` folder you created in Step 1.2.

> **Security note:** This file contains credentials. Never upload it to GitHub. We block it in .gitignore in Step 3.3.

---

## Part 3: Create the dbt Project

### Step 3.1 — Run dbt init

🖥️ **TERMINAL**

Make sure you are still inside `project4_dbt` with `(dbt_env)` active, then run:

```bash
dbt init ga4_ecommerce
```

dbt will ask you a series of questions. Answer them exactly as shown:

```
Which database adapter would you like to use?
[1] bigquery
Enter a number: 1

Desired authentication method option:
[1] oauth
[2] service_account
Enter a number: 2

project (GCP project id): YOUR_GCP_PROJECT_ID
dataset (the name of your dbt dataset): dbt_ga4_dev
threads [1]: 4
job_execution_timeout_seconds [300]: 300
job_retries [1]: 1

Desired location option:
[1] US
[2] EU
Enter a number: 1

keyfile (/path/to/keyfile.json): /Users/YourName/Documents/project4_dbt/dbt_service_account.json
```

Notes:
- **project:** your GCP project ID — find it in the GCP console top bar. It looks like `ga4-analytics-project-123456`.
- **dataset:** enter `dbt_ga4_dev`. This is your development sandbox.
- **keyfile:** enter the full absolute path to `dbt_service_account.json`. On Mac it starts with `/Users/YourName/...`, on Windows it starts with `C:\Users\YourName\...`.

When this finishes, two things have happened:
1. A `ga4_ecommerce` project folder was created inside `project4_dbt`
2. A `profiles.yml` file was written to `~/.dbt/profiles.yml` (Mac/Linux) or `C:\Users\YourName\.dbt\profiles.yml` (Windows)

---

### Step 3.2 — Add the Production Target to profiles.yml

`dbt init` only created a `dev` target in `profiles.yml`. We need to manually add a `prod` target.

📝 **VS CODE** — open the file:
- Mac/Linux: `~/.dbt/profiles.yml`
- Windows: `C:\Users\YourName\.dbt\profiles.yml`

Replace the entire contents of the file with the following. Substitute your actual GCP project ID and your actual keyfile path.

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
      job_execution_timeout_seconds: 300
      location: US
      priority: interactive
    prod:
      type: bigquery
      method: service-account
      project: YOUR_GCP_PROJECT_ID
      dataset: dbt_ga4
      keyfile: /Users/YourName/Documents/project4_dbt/dbt_service_account.json
      threads: 4
      job_execution_timeout_seconds: 300
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
      job_execution_timeout_seconds: 300
      location: US
      priority: interactive
    prod:
      type: bigquery
      method: service-account
      project: YOUR_GCP_PROJECT_ID
      dataset: dbt_ga4
      keyfile: C:\Users\YourName\Documents\project4_dbt\dbt_service_account.json
      threads: 4
      job_execution_timeout_seconds: 300
      location: US
      priority: interactive
```

Save and close the file.

> **Why two targets?**
> - `dev` writes to `dbt_ga4_dev` — your sandbox. Safe to break things here.
> - `prod` writes to `dbt_ga4` — the clean dataset that Looker Studio and Vanna will connect to.
> - `target: dev` at the top means every `dbt run` defaults to dev unless you explicitly pass `--target prod`. Your production data is always protected.

---

### Step 3.3 — Update .gitignore

📝 **VS CODE** — open `ga4_ecommerce/.gitignore`

dbt already created this file. Open it and make sure it contains the following. Add any lines that are missing:

```
# Credentials — never commit these
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

Save the file.

---

## Part 4: Configure the dbt Project

### Step 4.1 — Update dbt_project.yml

📝 **VS CODE** — open `ga4_ecommerce/dbt_project.yml`

Replace the entire contents with:

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

Save the file.

**What this does:**
- `staging` models → created as **views** (no storage cost, always fresh)
- `intermediate` models → created as **views**
- `marts` models → created as **tables** (stored, fast to query)
- Each layer gets its own schema in BigQuery

---

### Step 4.2 — Create the Model Folders and Remove Example Models

🖥️ **TERMINAL**

`dbt init` already created `macros`, `seeds`, and `tests`. You only need to create the three subfolders inside `models`, and delete the example models dbt generated.

```bash
# Navigate into the dbt project folder
cd ga4_ecommerce

# Create the three model layer folders
mkdir -p models/staging
mkdir -p models/intermediate
mkdir -p models/marts

# Delete the example models dbt generated
rm -rf models/example
```

---

## Part 5: Verify Everything Works

### Step 5.1 — Run dbt debug

🖥️ **TERMINAL**

Make sure you are inside the `ga4_ecommerce` folder, then run:

```bash
dbt debug
```

**What you want to see:**
```
All checks passed!
```

**Common errors and fixes:**

| Error | Fix |
|---|---|
| `Could not find profile named 'ga4_ecommerce'` | The profile name in `dbt_project.yml` must exactly match the top-level key in `profiles.yml` |
| `FileNotFoundError: dbt_service_account.json` | The `keyfile` path in `profiles.yml` must be the full absolute path to the file |
| `403 Access Denied` | Your service account does not have BigQuery Admin — go back to Step 2.3 |
| `Project not found` | Copy your GCP project ID exactly from the GCP console — it includes the number suffix |

---

### Step 5.2 — Create a Test Model

📝 **VS CODE** — create a new file at `models/staging/stg_test.sql`

Paste this content into it:

```sql
SELECT
  'dbt is working' AS message,
  CURRENT_TIMESTAMP() AS run_at
```

Save the file.

---

### Step 5.3 — Run the Test Model

🖥️ **TERMINAL**

```bash
dbt run --select stg_test
```

You should see:
```
1 of 1 START sql view model dbt_ga4_dev_staging.stg_test .................. [RUN]
1 of 1 OK created sql view model dbt_ga4_dev_staging.stg_test ............. [OK]
```

> **Why does it say `dbt_ga4_dev_staging` and not `dbt_ga4_dev`?** dbt combines your dataset name (`dbt_ga4_dev`) with the schema you set in `dbt_project.yml` (`staging`) to create `dbt_ga4_dev_staging`. This is correct and expected.

---

### Step 5.4 — Verify in BigQuery

🌐 **BROWSER** — [console.cloud.google.com/bigquery](https://console.cloud.google.com/bigquery)

In the left panel, expand your project and confirm:
- A dataset called `dbt_ga4_dev_staging` exists
- Inside it, there is a view called `stg_test`

If it is there, dbt is fully working.

---

### Step 5.5 — Delete the Test Model

📝 **VS CODE** — right-click `models/staging/stg_test.sql` → **Delete**

---

## Part 6: VS Code Setup

### Step 6.1 — Install the dbt Power User Extension

📝 **VS CODE**

1. Click the Extensions icon in the left sidebar (or press `Ctrl+Shift+X`)
2. Search for `dbt Power User`
3. Click **Install**

This extension gives you:
- SQL syntax highlighting for dbt models
- Auto-complete for `ref()` and `source()` functions
- Preview of compiled SQL before you run it
- A visual lineage graph showing how your models connect

---

### Step 6.2 — Open the Project Folder in VS Code

🖥️ **TERMINAL**

```bash
# Run this from inside the ga4_ecommerce folder
code .
```

This opens the entire `ga4_ecommerce` folder as a VS Code workspace. From this point on, all file editing happens here.

---

## Setup Complete

You now have:
- [ ] dbt Core installed with the BigQuery adapter
- [ ] A Google Cloud service account with BigQuery access
- [ ] A dbt project with dev and prod targets configured in `profiles.yml`
- [ ] `dbt_project.yml` configured with staging, intermediate, and marts layers
- [ ] Verified that dbt can connect to and write to BigQuery

**Your folder structure should look like this:**
```
project4_dbt/
├── dbt_service_account.json    ← stays here, never committed to Git
├── dbt_env/                    ← your virtual environment
└── ga4_ecommerce/
    ├── .gitignore
    ├── dbt_project.yml
    ├── models/
    │   ├── staging/            (empty)
    │   ├── intermediate/       (empty)
    │   └── marts/              (empty)
    ├── macros/
    ├── seeds/
    └── tests/
```

When you are ready, move on to [../docs/04_dbt_staging.md](../docs/04_dbt_staging.md).
