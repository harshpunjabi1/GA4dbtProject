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

This installs both dbt Core and the BigQuery adapter. It will take 1-2 minutes.

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

> **Why BigQuery Admin?** For a learning project this is fine. In a production environment you would use a more restrictive role.

### Step 2.4 — Download the Service Account Key

1. On the Service Accounts page, find the account you just created
2. Click the three dots (⋮) on the right → **Manage keys**
3. Click **Add Key** → **Create new key**
4. Select **JSON** → **Create**
5. A JSON file will download to your computer

**Rename this file** to `dbt_service_account.json` and move it into your `project4_dbt` folder.

> **Security note:** This file contains credentials. Never upload it to GitHub. We will set up `.gitignore` to block this in Step 3.3.

---

## Part 3: Create the dbt Project

### Step 3.1 — Initialise the Project

Make sure your virtual environment is active and you are inside the `project4_dbt` folder, then run:

```bash
dbt init ga4_ecommerce
```

dbt will ask you a series of questions in the terminal. Answer them as follows:

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

A few notes on these prompts:
- For `project`, enter your Google Cloud project ID — find it in the GCP console. It looks like `ga4-analytics-project-123456`.
- For `dataset`, enter `dbt_ga4_dev`. This is your development dataset. We will add the production dataset separately in the next step.
- For `keyfile`, enter the **full absolute path** to the `dbt_service_account.json` file you just moved into `project4_dbt`.

When this finishes, dbt has done two things:
1. Created a `ga4_ecommerce` project folder
2. Written a `profiles.yml` file to a hidden folder on your system (`~/.dbt/profiles.yml` on Mac/Linux, `C:\Users\YourName\.dbt\profiles.yml` on Windows)

---

### Step 3.2 — Add the Production Target to profiles.yml

`dbt init` created a `dev` target in `profiles.yml`. We need to add a `prod` target manually.

Open `~/.dbt/profiles.yml` in VS Code. You will see the `dev` block that dbt wrote. Replace the entire file contents with the following, substituting your actual values:

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

> **Why two targets?**
> - `dev` writes to `dbt_ga4_dev` — your sandbox. Safe to break things here.
> - `prod` writes to `dbt_ga4` — the clean dataset that Looker Studio and Vanna will connect to.
> - `target: dev` at the top means every `dbt run` command defaults to dev unless you explicitly pass `--target prod`. Your production data is always protected.

Save and close `profiles.yml`.

---

### Step 3.3 — Set Up .gitignore

Navigate into the `ga4_ecommerce` folder that dbt created. dbt already generated a `.gitignore` file there — open it and make sure it contains at minimum the following:

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

Open `ga4_ecommerce/dbt_project.yml` in VS Code and replace its entire contents with:

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
- `staging` models will be created as **views** (no storage cost, always fresh)
- `intermediate` models will be created as **views**
- `marts` models will be created as **tables** (stored, fast to query)
- Each layer gets its own schema in BigQuery, prefixed with your dataset name

---

### Step 4.2 — Create the Folder Structure

`dbt init` already created `macros`, `seeds`, `tests`, and a `models` folder. You only need to create the three layer subfolders inside `models`, and clean up the example models dbt generated.

```bash
# From inside the ga4_ecommerce folder
mkdir -p models/staging
mkdir -p models/intermediate
mkdir -p models/marts

# Remove the example models dbt created
rm -rf models/example
```

Your `models` folder should now have three empty subfolders: `staging`, `intermediate`, and `marts`.

---

## Part 5: Verify Everything Works

### Step 5.1 — Run dbt Debug

This command checks that dbt can connect to BigQuery.

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
| `Could not find profile named 'ga4_ecommerce'` | The profile name in `dbt_project.yml` must exactly match the top-level key in `profiles.yml` |
| `FileNotFoundError: dbt_service_account.json` | The `keyfile` path in `profiles.yml` must be the full absolute path to the file |
| `403 Access Denied` | Your service account does not have BigQuery Admin — go back to Step 2.3 |
| `Project not found` | Copy your GCP project ID exactly from the GCP console — it includes the number suffix |

---

### Step 5.2 — Create and Run a Test Model

Create a simple model to confirm dbt can write to BigQuery.

In VS Code, create the file `models/staging/stg_test.sql` with this content:

```sql
SELECT
  'dbt is working' AS message,
  CURRENT_TIMESTAMP() AS run_at
```

Run it from the terminal:

```bash
dbt run --select stg_test
```

You should see:
```
1 of 1 START sql view model dbt_ga4_dev_staging.stg_test .................. [RUN]
1 of 1 OK created sql view model dbt_ga4_dev_staging.stg_test ............. [OK]
```

> **Why does the dataset say `dbt_ga4_dev_staging` and not just `dbt_ga4_dev`?** dbt appends the schema name from `dbt_project.yml` to your dataset name. So `dbt_ga4_dev` (your dataset) + `staging` (your schema config) = `dbt_ga4_dev_staging`. This is expected and correct.

Go to BigQuery in the GCP console and confirm that:
- A dataset called `dbt_ga4_dev_staging` exists
- Inside it, there is a view called `stg_test`

If it is there, dbt is fully working.

**Delete the test file before moving on:**

```bash
rm models/staging/stg_test.sql
```

---

## Part 6: VS Code Setup

### Step 6.1 — Install the dbt Power User Extension

1. Open VS Code
2. Click the Extensions icon (or press `Ctrl+Shift+X`)
3. Search for `dbt Power User`
4. Click **Install**

This extension gives you:
- SQL syntax highlighting for dbt models
- Auto-complete for `ref()` and `source()` functions
- Preview of compiled SQL before you run it
- A visual lineage graph showing how your models connect

### Step 6.2 — Open the Project in VS Code

```bash
code .
```

Or open VS Code and use **File → Open Folder** to select the `ga4_ecommerce` folder.

---

## Setup Complete

You now have:
- [ ] dbt Core installed with the BigQuery adapter
- [ ] A Google Cloud service account with BigQuery access
- [ ] A dbt project with dev and prod targets configured
- [ ] The correct folder structure for staging, intermediate, and marts
- [ ] Verified that dbt can connect to and write to BigQuery

**Your project folder should look like this:**
```
project4_dbt/
├── dbt_service_account.json    ← stays here, never committed to Git
├── dbt_env/                    ← your virtual environment
└── ga4_ecommerce/
    ├── .gitignore
    ├── dbt_project.yml
    ├── models/
    │   ├── staging/            (empty for now)
    │   ├── intermediate/       (empty for now)
    │   └── marts/              (empty for now)
    ├── macros/
    ├── seeds/
    └── tests/
```

When you are ready, move on to [../docs/04_dbt_staging.md](../docs/04_dbt_staging.md).
