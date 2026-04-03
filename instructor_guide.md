# Project 4: Instructor Guide
## GA4 Ecommerce Analytics Engineering

---

## Course Overview

**Target audience:** Data analysts and aspiring analytics engineers with SQL knowledge. Comfortable with BigQuery from Project 2. No prior dbt or Python experience required.

**Estimated recording time:** 7–8 hours across 9 modules

**Difficulty level in bundle:** Advanced (Module 4 of 4)

**Key differentiators from other courses on the market:**
1. BEAM workshop before any code is written — design-first approach
2. Three-layer dbt architecture with explicit cost management
3. Vanna training data derived from dbt documentation — the feedback loop
4. Every table traces back to a named business question

---

## Module-by-Module Recording Guide

---

### Module 00: Business Problem (Target: 20 min)

**Objective:** Set the scene. Students should feel the pain of the problem before seeing the solution.

**Recording notes:**
- Open with the scenario — you are a newly hired analytics engineer
- Spend at least 3 minutes on the problem: raw GA4 data is unusable, everyone gets different numbers, the data team is a bottleneck
- Write the three business questions on screen — these will appear in every subsequent module as an anchor
- Close by showing the finished project (dashboard + Vanna demo) so students know exactly where they are going

**Common student confusion:**
- "Why can't we just use GA4's own interface?" — acknowledge this. GA4's UI is fine for standard reports. It cannot be customised, version-controlled, or extended with AI. The dbt approach scales.

**Key message to land:** Business problem first. Technology second. This is the discipline that separates analytics engineers from report builders.

---

### Module 01: Data Exploration (Target: 40 min)

**Objective:** Students understand the GA4 schema and can articulate why raw GA4 data is hard to work with.

**Recording notes:**
- Screen share BigQuery UI throughout
- Run each query live — do not pre-record results
- When you show the nested `event_params` array, pause and zoom in. This is the "aha" moment where students understand why unnesting is necessary
- When you run the funnel query (Step 6), make a note of the actual numbers — reference them when you build the funnel dashboard in Module 06

**Live queries to run in order:**
1. Scale query (total events, users, days)
2. Event types query (shows all event names)
3. One purchase event with `SELECT *` — click into event_params
4. Items array query
5. Funnel numbers query
6. Traffic source query

**Cost management teaching moment:**
Before running the first query, show students how to hover over the query editor to see the estimated bytes processed. Explain the 1TB free monthly quota and how to check usage in BigQuery → Administration → Monitoring.

**Common student confusion:**
- "Why are there 92 separate tables?" — explain the GA4 export naming convention (`events_YYYYMMDD`) and the wildcard pattern
- "What is user_pseudo_id?" — explain it is a pseudonymous identifier, not a real user ID, and that its meaning is limited by obfuscation in this dataset

---

### Module 02: BEAM Workshop (Target: 50 min)

**Objective:** Students understand BEAM and can design a dimensional model from business events before touching dbt.

**Recording notes:**
- Use a whiteboard, Miro, or a simple spreadsheet — not code
- Work through the 7Ws for Event 3 (purchase) first. It is the richest event and produces the clearest example
- When you get to Events 1 and 2 (view/cart), note explicitly that the dimensions are shared — this is what star schema is about
- Draw the final dimensional model (fact tables + dim tables) visually before moving to any code

**The critical teaching moment:**
Show the connection between the BEAM output and what dbt will build. Say explicitly: "Every table we are about to build was decided in this room before we opened VS Code. That is the BEAM way."

**The contrast with Project 2:**
If students have completed Project 2 (FinTech star schema), reference it directly: "In Project 2 you built a star schema. BEAM is the process that produces that design. Now you understand where star schemas come from."

**Common student confusion:**
- "Why not just design the tables based on what GA4 gives us?" — this is the most important question to answer well. Designing from source data means your model reflects the limitations of the source. Designing from business events means your model reflects what the business needs to measure. The source data should serve the model, not define it.

---

### Module 03: dbt Setup (Target: 45 min)

**Objective:** Every student has a working dbt installation with a verified BigQuery connection.

**Recording notes:**
- Record this on a clean machine if possible — shows students exactly what they will see
- Show BOTH Mac and Windows where the commands differ (or record two versions)
- When creating the service account, move slowly. This is where most students get stuck
- Show the `profiles.yml` file being edited — zoom in so every character is visible
- Do NOT skip the `dbt debug` step — run it on screen and confirm "All checks passed!"

**Where students get stuck (most common issues in order):**

1. **Wrong Python version** — `python` vs `python3` on Mac. Show `which python3` and `which python` to diagnose
2. **Virtual environment not activated** — symptoms: `dbt: command not found`. Solution: always activate before working
3. **Service account key path** — relative vs absolute paths. Always use absolute paths in `profiles.yml`
4. **Wrong GCP project ID** — the project ID is not the project name. Show where to find it in the GCP console (it looks like `my-project-123456`)
5. **BigQuery API not enabled** — show the APIs & Services page and how to enable it

**Tip for recording:**
Create a new GCP project live during recording — do not use a pre-existing one. Students are doing this from scratch and the recording should match their experience.

---

### Module 04: Staging Layer (Target: 55 min)

**Objective:** Students understand what the staging layer is for and can write all three staging models.

**Recording notes:**
- Open the `_sources.yml` creation first — explain that this is dbt's way of declaring external data sources
- When writing `stg_ga4__events.sql`, explain each section:
  - The `config` block — what `materialized = 'view'` means
  - The dev limit pattern — why this exists and how to switch it off
  - The surrogate key — why we need it (GA4 has no single unique event ID)
  - The `UNNEST` in the session_id construction — first real unnesting students see
- When writing `stg_ga4__event_params.sql`, explain the four value types (`string_value`, `int_value`, `float_value`, `double_value`) — this is a GA4-specific quirk that confuses everyone
- Write the `_staging.yml` documentation as you build each model, not after. This shows documentation as part of the development process, not a chore added at the end

**The documentation payoff setup:**
When writing column descriptions in `_staging.yml`, say explicitly: "These descriptions are going to do something important later in the project. For now, just know that good descriptions here pay off in Module 08."

**dbt run demonstration:**
Run `dbt run --select staging` and show students:
1. The terminal output with green [OK] messages
2. The resulting views in BigQuery
3. The query plan BigQuery shows for the view

---

### Module 05: Intermediate and Marts (Target: 85 min)

**Objective:** Students build the full dimensional model in dbt and understand why each layer exists.

**Recording structure:**
- Part 1 (20 min): Intermediate models — sessionisation logic, funnel tagging
- Part 2 (50 min): Mart models — fct_purchases, fct_product_interactions, fct_sessions
- Part 3 (15 min): Full run, testing, lineage graph

**Key teaching moments:**

**On intermediate models:**
"We could write this session logic directly in the mart. But then every mart that needs session data would duplicate it. Intermediate models mean we write the logic once and reference it many times."

**On partitioning:**
Show the BigQuery cost impact of partitioning. Query `fct_purchases` with and without a date filter — show the bytes processed estimate. This is real cost management that professionals do every day.

**On the lineage graph:**
Run `dbt docs generate && dbt docs serve` and show the full DAG. Make students find:
- The three source nodes
- Where staging models sit
- How intermediate models connect to marts
- "This graph is automatically generated from your code. When a new engineer joins the team, this is how they understand the data model."

**The prod run:**
Before switching to `--target prod`, warn students about BigQuery costs. Show them the bytes estimate first. Run it with them watching.

---

### Module 06: Tableau (Target: 55 min)

**Objective:** Students connect Tableau to BigQuery and build all three dashboards.

**Recording notes:**
- Record the BigQuery connection setup slowly — the authentication flow trips students up
- For each dashboard, start by saying the business question out loud before touching Tableau
- Show calculated fields being created — do not skip or hide this
- The scatter chart in Dashboard 1 (volume vs conversion rate by channel) is the most insightful chart in the project. Give it time.
- The funnel chart is technically the hardest to build — record step by step

**Design principle to reinforce:**
Before building each chart, say: "Before I drag anything, what is the one question this chart needs to answer?" Then build it to answer that question. This trains students to think like analysts, not report builders.

**Common Tableau issues:**
- BigQuery connection requires signing in via browser popup — this sometimes blocks in some firewall setups. Alternative: export mart tables to CSV and connect to local files
- Calculated fields with COUNTD and IF statements can be slow to load — show the performance optimisation of connecting to partitioned tables

---

### Module 07: Vanna Setup (Target: 30 min)

**Objective:** Students have a working Vanna installation connected to BigQuery.

**Recording notes:**
- Show the Google AI Studio API key creation live — students need to see this
- Explain ChromaDB in plain language: "It is a local database that stores your training data as vectors. Think of it as Vanna's long-term memory."
- Run `test_connection.py` live — show the row count result
- Address the free tier question directly: "250 requests per day is more than enough for this project. You are not building a production system — you are building a portfolio project."

---

### Module 08: Vanna Training and Demo (Target: 55 min)

**Objective:** Students train Vanna and see it generate correct SQL from natural language questions. They understand the documentation-quality connection.

**Recording structure:**
- Part 1 (15 min): Run `train_vanna.py` — explain each training type as it runs
- Part 2 (10 min): The quality demonstration — before/after documentation effect
- Part 3 (30 min): Live demo with `vanna_demo.py` — run all 10 questions

**The quality demonstration (most important segment in the course):**

Do this live on camera:
1. Create a temporary Vanna instance trained only on DDL (no documentation)
2. Ask: "What is the session conversion rate by channel?"
3. Show the SQL it generates — likely to be incorrect or use the wrong formula
4. Now train documentation block for `fct_sessions` that says: "Conversion rate = SUM(converted) / COUNT(session_id)"
5. Ask the same question
6. Show the correct SQL

Then say: "The only thing that changed was the description we wrote in `_staging.yml` and `_marts.yml` back in Modules 04 and 05. That documentation — which looked like optional busywork — is what makes Vanna work correctly. Every word you write in a schema description is LLM training data."

**Live demo tips:**
- Run questions live, do not pre-record results
- When a question produces wrong SQL, do not hide it — say "this is a gap in our training data, here is how we would fix it" (add a new question/SQL pair)
- Let the student input prompt run — type a question students are likely to ask

---

### Module 09: Portfolio Packaging (Target: 25 min)

**Objective:** Students know exactly how to present this project on GitHub and in interviews.

**Recording notes:**
- Walk through the README template live in VS Code
- Show a before/after of a weak vs strong interview answer (script is in the module)
- Run through the final checklist — check off each item on camera

**Key message:**
"The project is not complete when the code works. The project is complete when someone who has never seen your code can understand what it does, why it was built, and what it answers — in under five minutes of reading your README."

---

## Timing Summary

| Module | Target Time | Buffer |
|---|---|---|
| 00 Business Problem | 20 min | +5 min |
| 01 Data Exploration | 40 min | +10 min |
| 02 BEAM Workshop | 50 min | +10 min |
| 03 dbt Setup | 45 min | +15 min |
| 04 dbt Staging | 55 min | +10 min |
| 05 dbt Marts | 85 min | +15 min |
| 06 Tableau | 55 min | +10 min |
| 07 Vanna Setup | 30 min | +5 min |
| 08 Vanna Training | 55 min | +10 min |
| 09 Portfolio | 25 min | +5 min |
| **Total** | **460 min (~7.7 hrs)** | **+95 min** |

---

## Frequently Asked Student Questions

**"Can I use dbt Cloud instead of dbt Core?"**
Yes, but dbt Cloud has a free tier limit of one developer seat and limited job runs. dbt Core is recommended because it is fully free, teaches students how things work under the hood, and the setup process itself is a learning experience. If a student specifically wants dbt Cloud experience, point them to the dbt Cloud docs after completing the Core project.

**"My BigQuery free tier ran out — what do I do?"**
This should not happen if students follow the dev limit patterns. If it does: (1) check which queries ran without the dev filter, (2) enable billing with a $10 prepaid credit — the actual usage for this project is under $1 if dev limits are used throughout, (3) remind them that the prod run should only be done once.

**"Can I use PowerBI instead of Tableau?"**
Yes. The dashboard designs work identically in PowerBI Desktop. PowerBI's DAX calculated fields work the same way as Tableau's calculated fields. The BigQuery connector is native in PowerBI. Students choosing PowerBI should refer to the Project 2 guide for connection setup.

**"Vanna keeps generating wrong SQL for my custom question"**
This is expected and is a teaching moment. Three fixes in order: (1) add the specific question as a training pair with the correct SQL, (2) improve the documentation description for the relevant table, (3) rephrase the question to use terms that appear in the training data.

**"Can I use a different LLM instead of Gemini?"**
Yes. Vanna supports OpenAI, Anthropic, and local models via Ollama. The `config.py` swap is straightforward — replace `GoogleGeminiChat` with `OpenAI_Chat` or `AnthropicChat`. However, OpenAI and Anthropic require paid API keys after free credits expire. Stick with Gemini for the course recommendation.

---

## Pre-Recording Checklist

Before recording each module:

- [ ] Virtual environment activated
- [ ] dbt debug passes
- [ ] BigQuery connection working
- [ ] Screen recording software running at 1080p or higher
- [ ] Font size in VS Code set to minimum 16px (students watch on small screens)
- [ ] Terminal font size minimum 16px
- [ ] BigQuery UI zoom at 110% minimum
- [ ] All sensitive values (API keys, project IDs) replaced with placeholders before screenshare

---

## What Not To Do

**Do not:**
- Skip the BEAM workshop and go straight to dbt — the design-first principle is the core differentiator of this course
- Pre-run all queries and show static results — students need to see the live execution
- Ignore errors when they happen — work through them on camera, students will encounter the same errors
- Use dbt Cloud — the setup difference from Core causes confusion
- Connect Tableau to the dev dataset — students will see one week of data and think the dashboards are incomplete
