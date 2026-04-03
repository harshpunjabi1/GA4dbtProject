# Project 4: GA4 Ecommerce Analytics Engineering
## BigQuery · dbt Core · BEAM Modeling · Tableau · Vanna AI

---

## What You Will Build

By the end of this project you will have built a complete, production-style analytics engineering pipeline from raw GA4 event data through to a conversational analytics layer. Every technical decision in this project is driven by a business question first.

```
Raw GA4 Events (BigQuery)
        ↓
  Business Problem + Data Exploration
        ↓
  BEAM Workshop → Star Schema Design
        ↓
  dbt Core (Staging → Intermediate → Marts)
        ↓
  Tableau Dashboards (3 business views)
        ↓
  Vanna AI + Gemini (Conversational Analytics)
```

---

## Prerequisites

Before starting this project you should be comfortable with:
- SQL (joins, CTEs, window functions)
- Basic Python (running scripts, pip installs)
- BigQuery UI (you used this in Project 2)
- Git and GitHub basics

---

## Tools Used

| Tool | Purpose | Cost |
|---|---|---|
| Google BigQuery | Cloud data warehouse | Free tier |
| dbt Core | Data transformation framework | Free / open source |
| Tableau Public | Business intelligence dashboards | Free |
| Vanna AI | Text-to-SQL conversational layer | Free / open source |
| Google Gemini API | LLM for Vanna | Free tier |
| Python 3.9+ | Running scripts and Vanna | Free |
| VS Code | Code editor | Free |
| GitHub | Version control | Free |

---

## Project Modules

| Module | File | Description |
|---|---|---|
| 00 | [00_business_problem.md](docs/00_business_problem.md) | Business context and the problem we are solving |
| 01 | [01_data_exploration.md](docs/01_data_exploration.md) | Exploring the GA4 dataset in BigQuery |
| 02 | [02_beam_workshop.md](docs/02_beam_workshop.md) | BEAM workshop and dimensional model design |
| 03 | [03_dbt_setup.md](setup/03_dbt_setup.md) | Complete dbt Core setup guide for beginners |
| 04 | [04_dbt_staging.md](docs/04_dbt_staging.md) | Building the staging layer |
| 05 | [05_dbt_marts.md](docs/05_dbt_marts.md) | Building intermediate models and marts |
| 06 | [06_tableau.md](docs/06_tableau.md) | Connecting Tableau and building dashboards |
| 07 | [07_vanna_setup.md](setup/07_vanna_setup.md) | Complete Vanna AI setup guide |
| 08 | [08_vanna_training.md](docs/08_vanna_training.md) | Training Vanna and running the demo |
| 09 | [09_portfolio.md](docs/09_portfolio.md) | Packaging the project for your portfolio |

---

## Dataset

**Source:** `bigquery-public-data.ga4_obfuscated_sample_ecommerce`

This is the Google Merchandise Store's GA4 data exported to BigQuery. It covers three months of real (obfuscated) ecommerce behaviour from November 2020 to January 2021 and includes page views, product interactions, cart events, and purchases.

---

## Quick Start

If you want to jump straight in:

1. Complete [03_dbt_setup.md](setup/03_dbt_setup.md) — this takes about 30 minutes
2. Run `dbt debug` to confirm your connection works
3. Come back and start from Module 00

---

## Repository Structure

```
project4/
├── README.md
├── docs/                        # Module guides
├── setup/                       # Setup guides (dbt, Vanna)
├── dbt_project/                 # Your dbt project lives here
│   ├── dbt_project.yml
│   ├── profiles.yml (gitignored)
│   ├── models/
│   │   ├── staging/
│   │   ├── intermediate/
│   │   └── marts/
│   ├── macros/
│   ├── seeds/
│   └── tests/
├── vanna/                       # Vanna scripts
├── scripts/                     # Utility scripts
└── instructor_guide/            # Instructor materials
```
