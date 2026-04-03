# Module 02: BEAM Workshop

## What is BEAM?

**Business Event Analysis & Modeling (BEAM)** is a method for designing data warehouse and BI solutions by starting with the business — not the source data and not the reports.

It was developed by Lawrence Corr and is described in the book *Agile Data Warehouse Design*. The core idea is simple:

> Ask the business "who is doing what?" and use those event stories to design your dimensional model.

BEAM produces a **star schema** — fact tables at the centre holding business events, surrounded by dimension tables holding the context that describes those events. This is the same dimensional modeling approach used in Project 2, but now you understand where the design comes from.

---

## The 7Ws Framework

For every business event, BEAM asks seven questions:

| W | Question | What It Produces |
|---|---|---|
| **Who** | Who performs or is involved in the event? | `dim_users` |
| **What** | What is acted upon? | `dim_products` |
| **When** | When does the event happen? | `dim_date`, timestamp |
| **Where** | Where does it happen? | `dim_device`, geography |
| **Why** | Why does it happen? | `dim_traffic_source` |
| **How many** | How much, how many, how often? | Measures in the fact table |
| **How** | How is it performed or classified? | Additional dimension attributes |

---

## Step 1: Identify the Business Events

From our data exploration we know the key events. Working with the business (in a real project this would be a workshop with stakeholders), we identify three core events to model:

**Event 1:** Customer views a product
**Event 2:** Customer adds a product to their cart
**Event 3:** Customer purchases products

There is also a supporting event:
**Event 4:** Customer starts a session (needed for session-level analysis)

---

## Step 2: Apply the 7Ws to Each Event

### Event 3: Customer Purchases Products
*(We start with the most important event)*

| 7W | Field in GA4 | Dimension/Measure |
|---|---|---|
| **Who** | `user_pseudo_id` | → `dim_users` |
| **What** | `items[].item_id`, `item_name` | → `dim_products` |
| **When** | `event_timestamp` | → `dim_date` + timestamp |
| **Where** | `device.category`, `geo.country` | → `dim_device` |
| **Why** | `traffic_source.source/medium` | → `dim_traffic_source` |
| **How many** | `ecommerce.purchase_revenue`, `items[].quantity` | → measures in `fct_purchases` |
| **How** | `ecommerce.transaction_id` | → natural key on `fct_purchases` |

**The fact table grain:** One row per transaction (one purchase event = one row)

---

### Event 1 & 2: Customer Views / Adds to Cart

Applying the same 7Ws, we notice that these events share almost identical dimensions with the purchase event — same user, same product, same device, same traffic source. They only differ in the measure (no revenue, just a count of the action).

**Key insight:** Shared dimensions mean shared dimension tables. `dim_users`, `dim_products`, `dim_device`, and `dim_traffic_source` serve all three fact tables.

**The fact table grain:** One row per product event (view or cart action)

---

### Event 4: Session Start

| 7W | Field in GA4 | Dimension/Measure |
|---|---|---|
| **Who** | `user_pseudo_id` | → `dim_users` |
| **When** | Session start timestamp | → `dim_date` |
| **Where** | `device.category` | → `dim_device` |
| **Why** | Traffic source at session level | → `dim_traffic_source` |
| **How many** | Session duration, page views in session | → measures in `fct_sessions` |
| **How** | Did the session result in a purchase? | → `converted` flag |

**The fact table grain:** One row per session

---

## Step 3: The Dimensional Model

From the BEAM workshop we can now draw the full dimensional model. This is what dbt will implement.

```
                    dim_date
                       │
        dim_device ────┤
                       │
dim_traffic_source ────┼──── fct_purchases
                       │
        dim_users ─────┼──── fct_product_interactions
                       │
      dim_products ────┼──── fct_sessions
                       │
                    dim_date
```

### Fact Tables

| Table | Grain | Key Measures |
|---|---|---|
| `fct_purchases` | One row per transaction | `revenue`, `item_count`, `avg_order_value` |
| `fct_product_interactions` | One row per product event | `interaction_type` (view/cart), count |
| `fct_sessions` | One row per session | `session_duration`, `page_views`, `converted` |

### Dimension Tables

| Table | Grain | Key Attributes |
|---|---|---|
| `dim_users` | One row per user | `first_visit_date`, `acquisition_channel`, `user_ltv` |
| `dim_products` | One row per product | `item_name`, `item_category`, `item_brand` |
| `dim_date` | One row per date | `date`, `week`, `month`, `day_of_week` |
| `dim_device` | One row per device type | `device_category`, `operating_system`, `browser` |
| `dim_traffic_source` | One row per source/medium | `source`, `medium`, `campaign` |

---

## Step 4: Connecting Design to Business Questions

Verify the model answers the three business questions from Module 00:

**Question 1 — Acquisition**
> "Which channels bring us users, and which convert?"

Answered by: `fct_sessions` joined to `dim_traffic_source` and `dim_users`. Filter on `converted = true` for conversion rate.

**Question 2 — Funnel**
> "Where do users drop off in the purchase journey?"

Answered by: `fct_product_interactions` ordered by funnel step, compared to `fct_purchases`. Drop-off rate = (step N users - step N+1 users) / step N users.

**Question 3 — Product**
> "Which products have high interest but low conversion?"

Answered by: `fct_product_interactions` (views) vs `fct_purchases` (buys), joined to `dim_products`.

If all three questions are answerable, the model is correctly designed. ✓

---

## Step 5: Document Your Event Stories

Before writing any code, document each event story in plain English. This documentation will later be used to train Vanna AI.

**Template:**
```
Event: [name]
Story: [Subject] [verb] [object]
Grain: [one row per ...]
Key measures: [list]
Key dimensions: [list]
Business question answered: [which of the 3]
```

**Example — fct_purchases:**
```
Event: Purchase
Story: Customer completes purchase of products on the website
Grain: One row per transaction
Key measures: revenue, item_count, avg_order_value
Key dimensions: user, product, date, device, traffic_source
Business question answered: Q1 (Acquisition), Q3 (Product)
```

Write out the same for `fct_product_interactions` and `fct_sessions` before moving on.

---

## What You Have Now

At the end of this module you have:
- [ ] Identified 4 key business events
- [ ] Applied the 7Ws to each event
- [ ] Designed a full dimensional model (3 fact tables, 5 dimension tables)
- [ ] Verified the model answers all 3 business questions
- [ ] Written event story documentation

This documentation is your contract. Everything built in dbt will implement this design — nothing more, nothing less.

When you are ready, move on to [../setup/03_dbt_setup.md](../setup/03_dbt_setup.md).
