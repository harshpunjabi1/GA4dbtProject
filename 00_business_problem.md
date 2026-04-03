# Module 00: Business Problem

## The Scenario

You have just joined the data team at the **Google Merchandise Store** — an online shop selling Google-branded products like apparel, accessories, and stationery.

The business has been running Google Analytics 4 (GA4) on the website and the raw event data is being exported automatically into BigQuery every day. Three months of data is sitting there right now.

The problem is that nobody can use it.

---

## The Problem

The marketing team, the product team, and the leadership team all want answers to questions like:

- Which traffic sources are driving the most revenue?
- Where are users dropping off before they purchase?
- Which products are most viewed but least purchased?
- How is conversion rate trending week over week?

Right now, anyone who wants an answer has to either:

1. Ask the data team to write a SQL query (slow, creates a bottleneck)
2. Write their own SQL against the raw GA4 tables (inconsistent, error-prone)
3. Use the GA4 interface (limited, cannot be customised)

Every person is getting slightly different numbers. There is no single source of truth.

---

## What We Are Going To Build

As the analytics engineer on the team, your job is to build a **single source of truth analytics layer** that:

1. Transforms the raw, messy GA4 event data into clean, business-ready tables
2. Organises those tables into a dimensional model that mirrors how the business thinks
3. Powers consistent BI dashboards that answer the core business questions
4. Enables non-technical stakeholders to ask questions in plain English

This is not a one-off SQL script. It is a system — version controlled, tested, documented, and extensible.

---

## The Three Business Questions We Will Answer

Everything we build in this project is anchored to three specific business questions. Keep these in mind throughout.

**Question 1 — Acquisition**
> "Which channels bring us users, and which of those users actually convert to purchase?"

**Question 2 — Funnel**
> "At which step of the purchase journey do we lose the most users, and does that vary by device or source?"

**Question 3 — Product**
> "Which products are generating revenue, and which have high interest but low conversion?"

Every table we design, every mart we build, and every dashboard we create will be traceable back to one of these three questions. If a design decision does not serve one of these questions, we will not make it.

---

## Why This Approach Matters

Most analytics projects fail not because of bad technology but because of bad design. The common failure modes are:

- Building dashboards before agreeing on what questions they should answer
- Designing tables based on what the source data looks like rather than what the business needs
- Skipping documentation so that nobody else can use or trust the output

This project teaches a different approach: **business problem first, design second, technology third**. The BEAM workshop in Module 02 is where that design happens — before a single line of dbt code is written.

---

## Before You Move On

Make sure you can answer these questions in your own words:

- [ ] What is the core problem the Google Merchandise Store data team is facing?
- [ ] What are the three business questions this project will answer?
- [ ] Why is writing ad-hoc SQL against raw GA4 data not a sustainable solution?

When you are ready, move on to [01_data_exploration.md](01_data_exploration.md).
