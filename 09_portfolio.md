# Module 09: Portfolio Packaging

## What Goes on GitHub

Your GitHub repository is how hiring managers evaluate this project. It should tell the complete story without them having to watch the full course.

---

## Repository README Template

Replace your project README with this structure:

```markdown
# GA4 Ecommerce Analytics Engineering
### BigQuery · dbt Core · BEAM Modeling · Tableau · Vanna AI

## Project Overview

End-to-end analytics engineering project built on three months of real
(obfuscated) Google Merchandise Store GA4 data. Transforms raw event data
into a business-ready dimensional model and conversational analytics layer.

**Business questions answered:**
1. Which channels bring users — and which convert them to purchase?
2. At which funnel step do we lose the most users?
3. Which products have high interest but low conversion?

## Architecture

[Include your workflow diagram here]

Raw GA4 Events (BigQuery Public Dataset)
  → BEAM Workshop (dimensional model design)
  → dbt Core (staging → intermediate → marts)
  → Tableau Public (3 business dashboards)
  → Vanna AI + Gemini (conversational analytics)

## Dashboards

[Tableau Public link here]

- Acquisition Performance: sessions and conversion rate by channel
- Purchase Funnel: drop-off analysis by device and source
- Product Performance: revenue and view-to-purchase ratio by product

## Key Technical Decisions

**Why BEAM modeling?**
The BEAM workshop was run before writing any code. Business event stories
were collected for the three core user actions (view, cart, purchase) and
the 7Ws framework was used to derive the dimensional model. This ensures
every table answers a specific business question.

**Why dbt Core?**
All transformations are version-controlled SQL. The three-layer architecture
(staging → intermediate → marts) separates concerns clearly. Dev targets
use date filters to minimise BigQuery costs during development.

**Why Vanna + Gemini?**
Vanna uses RAG to train on the mart schema and documentation. The dbt
schema.yml descriptions written during development double as Vanna training
data — making documentation a first-class engineering artifact.

## Repository Structure

[Include your folder structure]

## Setup Instructions

[Link to 03_dbt_setup.md and 07_vanna_setup.md]

## Dataset

Source: bigquery-public-data.ga4_obfuscated_sample_ecommerce
Three months of GA4 event data from the Google Merchandise Store (Nov 2020 - Jan 2021)
```

---

## How to Talk About This Project in Interviews

The most common mistake is describing what tools you used. Interviewers know what dbt is. What they want to hear is the decisions you made and why.

**Weak answer:**
> "I used dbt to build a data pipeline on GA4 data, then connected it to Tableau and Vanna."

**Strong answer:**
> "I started with a BEAM workshop to identify the business events that mattered — product views, cart adds, and purchases. Running the 7Ws for each event gave me the dimensional model before I wrote a single line of SQL. The staging layer handles all the GA4 unnesting, the intermediate layer reconstructs sessions, and the marts are what the business actually queries. I also trained Vanna on the mart documentation I wrote in dbt — so the same schema.yml descriptions that document the tables for engineers also teach the LLM to generate correct SQL for business users."

Practice answering:
- Why did you use BEAM instead of designing the schema from the source data?
- How did you handle the cost of running dbt against BigQuery?
- What is the grain of each fact table and why did you choose it?
- How does Vanna know to use `SUM(converted) / COUNT(*)` for conversion rate?

---

## Final Checklist

Before calling this project complete:

**dbt:**
- [ ] All models documented in `_staging.yml` and `_marts.yml`
- [ ] All dbt tests passing (`dbt test`)
- [ ] Lineage graph generated (`dbt docs generate`)
- [ ] `.gitignore` includes credentials and generated files

**Tableau:**
- [ ] All three dashboards published to Tableau Public
- [ ] Each dashboard title is phrased as a business question
- [ ] Tableau Public URL in README

**Vanna:**
- [ ] Training script runs without errors
- [ ] All 10 example questions return correct SQL
- [ ] Demo script interactive prompt works

**GitHub:**
- [ ] README tells the complete project story
- [ ] No credentials or API keys in any committed file
- [ ] Folder structure is clean and logical
- [ ] Module guides are readable and accurate
