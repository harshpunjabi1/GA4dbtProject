# BEAM Recording Pointers
## GA4 Ecommerce — Mapping Business Processes to the 7Ws

---

## Before You Hit Record

Have these open and ready:
- The data exploration results from Module 01 (the event list query output)
- A blank whiteboard, Miro board, or spreadsheet — you will fill this live
- The 7Ws slide image on a second monitor or printed out

The BEAM module should feel like a **workshop**, not a lecture. You are thinking out loud, making decisions, and explaining why. Students should feel like they are in the room with you as a practitioner.

---

## Opening Hook (2 min)

Start by saying this — or close to it:

> "Most analytics projects design tables based on what the source data looks like. That means your model inherits all the limitations and quirks of the source system. BEAM flips that. We start with what the business needs to measure, and we make the data serve that. So before we open VS Code, before we write a single line of SQL, we are going to do a design session."

Then open your blank whiteboard and say:

> "The first question we ask is not 'what columns does this table have?' It is: who is doing what?"

---

## Step 1: Identify the Business Events (5 min)

Write on screen — just free text for now, nothing structured:

```
What things happen on the Google Merchandise Store that the business cares about?

- A customer visits the site
- A customer looks at a product
- A customer puts something in their cart
- A customer buys something
```

Then narrow it down with a business lens:

> "Not everything that happens is worth modeling. We are not here to model every scroll event or every page load. We want the events that the business can act on. The question I always ask is: if this event happened more or less, would someone in the business care?"

Circle or highlight the four that matter:

```
✓ Customer views a product        ← marketing wants to know what interests users
✓ Customer adds item to cart      ← product wants to know intent
✓ Customer begins checkout        ← funnel analysis
✓ Customer completes a purchase   ← revenue, the most important one
```

Say explicitly: "These four events are the spine of the entire project. Everything we build will connect back to one of these."

---

## Step 2: Start with the Most Important Event (3 min)

Tell students:

> "In BEAM you always start with the richest, most important event and work outward. For an ecommerce store, that is the purchase. If we get the purchase event right, the others follow naturally."

Write at the top of your whiteboard/screen:

```
EVENT: Customer completes a purchase
STORY: A customer buys one or more products on the Google Merchandise Store website
```

The "story" format — subject + verb + object — is a BEAM convention. It forces you to name the actor, the action, and what is acted upon. If you cannot write the story in one sentence, you do not understand the event well enough to model it.

---

## Step 3: Apply the 7Ws — Walk Through Each W Slowly (15 min)

This is the core of the module. Do not rush it. For each W:
1. Ask the question out loud
2. Think about it — pause, say "so what does that mean for our data?"
3. Write the answer in the table
4. Connect it to a model output

Use a simple table structure on screen:

```
| W          | Question              | Answer (GA4 field)              | Model output         |
|------------|-----------------------|---------------------------------|----------------------|
| Who        |                       |                                 |                      |
| What       |                       |                                 |                      |
| When       |                       |                                 |                      |
| Where      |                       |                                 |                      |
| Why        |                       |                                 |                      |
| How Many   |                       |                                 |                      |
| How        |                       |                                 |                      |
```

Fill it in live, one row at a time. Here is the exact content for each row and what to say:

---

### WHO (2 min)

**Say:** "Who is involved in this event? In ecommerce the answer is almost always the customer. But we need to be specific — how does GA4 identify a customer?"

**Write:** user_pseudo_id

**Then say:** "This is a pseudonymous identifier — not a real user ID. GA4 anonymises it. This is fine for our purposes. Every unique user gets one of these. So the WHO maps to a dimension we will call dim_users."

**Write in table:**

```
| Who | Which user bought? | user_pseudo_id | → dim_users |
```

**Pointer:** Pause here and say "notice we write 'dim_' — this is the Kimball naming convention for dimension tables. Any time a W answer is an entity that describes the event, it becomes a dimension."

---

### WHAT (2 min)

**Say:** "What was bought? This seems obvious — products. But let us think about what GA4 actually gives us."

**Write:** items[] array — item_id, item_name, item_category

**Then say:** "The interesting thing here is that one purchase can contain multiple products. A customer might buy a hoodie and a mug in the same transaction. So the WHAT is actually a list of items, not a single product. That will affect our grain decision later."

**Write in table:**

```
| What | Which product(s)? | items[].item_id, item_name | → dim_products |
```

---

### WHEN (1 min)

**Say:** "When is always time. The question is: what level of time precision do we need?"

**Write:** event_timestamp (microseconds in GA4)

**Then say:** "GA4 gives us microsecond precision. We will convert this to a datetime in the staging layer. We also want a date column for partitioning and for the date dimension."

**Write in table:**

```
| When | At what point in time? | event_timestamp → converted to datetime | → dim_date + purchased_at |
```

---

### WHERE (2 min)

**Say:** "Where has two meanings in digital analytics. Physical location — country, city — and digital location — which device, which browser. Both matter."

**Write:**
- device.category (desktop / mobile / tablet)
- geo.country

**Then say:** "For this project we care more about device than geography, because device type affects funnel behaviour — mobile users drop off differently than desktop users. That is one of our three business questions."

**Write in table:**

```
| Where | On which device? From which country? | device.category, geo.country | → dim_device |
```

---

### WHY (2 min)

**Say:** "Why did the customer come to the site in the first place? This is traffic source. Where did they come from before they arrived?"

**Write:** traffic_source.source, traffic_source.medium, traffic_source.name

**Then say:** "In GA4, traffic_source is a user-level attribute — it records where the user first came from, not the source of each individual session. This is an important nuance. A user might have first visited from a Google ad, then come back directly three days later to purchase. The traffic_source will show the original ad, not the direct visit. This is first-touch attribution."

**Pointer:** This is a real-world data nuance — students appreciate when you flag things like this rather than glossing over them.

**Write in table:**

```
| Why | From which channel? | traffic_source.source/medium | → dim_traffic_source |
```

---

### HOW MANY (2 min)

**Say:** "How Many is where the measures live. This is the only W that does not produce a dimension — it produces the columns in the fact table itself."

Ask on screen: "What can we count or measure about a purchase?"

Then list:
- Revenue (ecommerce.purchase_revenue)
- Number of items (count of items[] array)
- Quantity of units (items[].quantity summed)
- Average order value (derived: revenue / item count)

**Then say:** "These are all additive measures — you can SUM them across any combination of dimensions. Revenue by channel. Revenue by device. Revenue by product category. The dimensional model makes all of that possible with a simple GROUP BY."

**Write in table:**

```
| How Many | How much revenue? How many items? | purchase_revenue, item count, quantity | → measures in fct_purchases |
```

---

### HOW (1 min)

**Say:** "How is a catch-all for anything that classifies the event but does not fit the other Ws. In this case it is the transaction ID — the business reference number for the purchase."

**Write:** ecommerce.transaction_id

**Then say:** "This becomes the natural key on the fact table — the real-world identifier for this specific transaction. We use it for deduplication and for joining back to order management systems if needed."

**Write in table:**

```
| How | Transaction reference | ecommerce.transaction_id | → Natural key on fct_purchases |
```

---

## Step 4: Read the Completed Table Back (2 min)

Once the table is complete, pause and read it back as a story:

> "So what we have said is: a customer — that is our WHO — bought one or more products — the WHAT — at a specific timestamp — WHEN — from a desktop or mobile device — WHERE — after arriving from Google organic search — WHY — spending $55 on two items — HOW MANY — with transaction ID T_12345 — HOW."

Then say:

> "That one sentence contains every row in our fact table. Every purchase in the dataset is one version of that story. And every bold word in that sentence is either a measure or a key to a dimension table."

---

## Step 5: Derive the Fact Table (3 min)

Now derive fct_purchases from the table:

```
fct_purchases (grain: one row per transaction)
├── purchase_id          (surrogate key)
├── transaction_id       (natural key — from HOW)
├── user_pseudo_id       (FK to dim_users — from WHO)
├── item_id              (FK to dim_products — from WHAT)
├── purchased_at         (timestamp — from WHEN)
├── purchase_date        (FK to dim_date — from WHEN)
├── device_category      (FK to dim_device — from WHERE)
├── traffic_source       (FK to dim_traffic_source — from WHY)
├── revenue              (measure — from HOW MANY)
├── item_count           (measure — from HOW MANY)
└── total_quantity       (measure — from HOW MANY)
```

Say: "The fact table is just the 7W table expressed as SQL columns. Every decision we made in the workshop is now a column name."

---

## Step 6: Repeat Quickly for the Other Events (5 min)

Do NOT redo the full 7W exercise for view_item and add_to_cart. Instead, say:

> "For the product view and add-to-cart events, the 7Ws give us almost identical answers. Same WHO — the user. Same WHAT — the product. Same WHEN, WHERE, WHY. The only thing that changes is the HOW MANY — there is no revenue, just a count of the action."

Write on screen:

```
Shared across all four events:
  WHO     → dim_users         (same table)
  WHEN    → dim_date          (same table)
  WHERE   → dim_device        (same table)
  WHY     → dim_traffic_source (same table)

Different per event:
  WHAT    → dim_products applies to view_item, add_to_cart, purchase
            (not session_start — session has no product)
  HOW MANY → revenue only on purchase
              count of action on view/cart events
```

Then say: "This is one of BEAM's most powerful outputs — shared dimensions. Because the same dim_users, dim_date, dim_device, and dim_traffic_source serve multiple fact tables, we can compare across events without any extra work. Conversion rate by channel is simply: purchases in fct_purchases divided by sessions in fct_sessions, joined through dim_traffic_source."

---

## Step 7: Draw the Final Model (3 min)

Draw the star schema. Do it by hand on screen — do not show a pre-built diagram. The act of drawing it live is more credible and more memorable.

Start with the three fact tables in the center:

```
fct_purchases
fct_product_interactions
fct_sessions
```

Then draw the dimension tables around them and connect with lines:

```
dim_users ────────── fct_purchases
                     fct_sessions

dim_products ──────── fct_purchases
                      fct_product_interactions

dim_date ──────────── fct_purchases
                      fct_product_interactions
                      fct_sessions

dim_device ─────────── fct_sessions
                       fct_product_interactions

dim_traffic_source ─── fct_sessions
                       fct_purchases
```

Say as you draw: "This is the dimensional model. It came from the BEAM workshop. Not from looking at GA4 tables. Not from a BI report request. From a conversation about what the business needs to measure."

---

## Closing Statement (1 min)

End the BEAM module with:

> "We are not going to open VS Code until the next module. And that is intentional. The design is done. Every table we build in dbt will implement what we just decided here. If we find ourselves building a table that does not appear in this model — we should ask why. The design is the contract."

---

## Common Mistakes to Avoid on Camera

- Do not rush through the 7Ws — each one should feel deliberate
- Do not skip the "why does this map to a dimension?" explanation for each W
- Do not pre-build the table before recording — fill it in live
- Do not say "we will figure this out later" — every W should have an answer before you move on
- Do not use technical jargon without defining it — say "surrogate key" then immediately say "which just means a unique ID we create ourselves"
