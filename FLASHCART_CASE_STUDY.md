# FlashCart Enterprise Data Modeling Case Study
## High-Scale Quick-Commerce Analytics Architecture

> **Domain:** Hyper-Growth On-Demand Delivery & Quick Commerce  
> **Ecosystem:** 10M+ Users, 5,000+ Dark Stores, 100K+ Couriers, Real-Time Inventory & Dispatch  
> **Target:** Kimball Dimensional Architecture & Lakehouse Gold Layer

---
Yes. Let’s do this as a **real Data Engineering design case study**, not as a list of definitions.

The goal is that after this, when someone gives you a business requirement, you can think:

> **What business process is this? → What is the grain? → What is the fact? → What are the dimensions? → Do I need one star or multiple stars? → What should be shared? → What history must be preserved? → What architecture should this live in?**

That is the level of thinking you want.

Current Microsoft Fabric guidance still recommends dimensional/star-schema modeling for analytical workloads, and current Databricks guidance explicitly puts fact/dimension star schemas naturally in the Gold layer of a lakehouse. So these are not obsolete textbook ideas. ([Microsoft Learn][1])

---

# PART 1 — First understand the entire picture

Imagine a huge company:

# **FLASHCART**

A global quick-commerce company.

It has:

```text
Customers
Products
Stores
Warehouses
Orders
Payments
Deliveries
Inventory
Returns
Promotions
Couriers
Suppliers
Support tickets
```

Millions or billions of events happen every day.

The company wants to answer:

```text
SALES
- How much did we sell?
- What products sell best?
- Which countries generate revenue?
- Which customers generate the most revenue?

CUSTOMERS
- How many active customers?
- Which customer segments are growing?
- What is customer lifetime value?

PAYMENTS
- Which payment methods fail?
- How much money was collected?
- How many payment attempts happen per order?

DELIVERY
- How long do deliveries take?
- Which stores are late?
- Which couriers perform best?

INVENTORY
- How much stock did we have?
- Which products go out of stock?
- Which stores have excess inventory?

RETURNS
- Which products are returned?
- Why are products returned?
- How much revenue is refunded?

PROMOTIONS
- Which campaigns increase sales?
- How much discount was given?
```

Now pause.

Do you see the important thing?

**These are not one business process.**

They are:

```text
Sales
Payment
Delivery
Inventory
Return
Promotion
Customer activity
```

That observation drives your fact-table design.

Kimball's current reference material still starts dimensional design from **business processes**, because choosing the process gives you the design target from which grain, dimensions, and facts are defined. ([Kimball Group][2])

---

# PART 2 — The first mistake beginners make

A beginner thinks:

> "The company needs analytics, so I'll make one giant Fact table."

Maybe:

```text
FactEverything
────────────────────────────
order_id
customer_id
product_id
payment_id
delivery_id
inventory_id
return_id
sales_amount
payment_amount
delivery_time
inventory_quantity
refund_amount
discount_amount
...
```

This looks like one big solution.

It is actually a disaster.

Why?

Because these things happen at **different grains**.

For example:

```text
Sale
    = one order line

Payment
    = one payment attempt

Delivery
    = one delivery

Inventory
    = one product + store + day

Return
    = one returned order line
```

Those are different business events.

Putting them together creates:

* duplicated data
* ambiguous meaning
* incorrect sums
* massive NULL areas
* difficult pipelines
* difficult maintenance

Microsoft's current fact-table guidance also emphasizes that a fact table needs a consistent grain, and explicitly warns about mixing different levels of detail. ([Microsoft Learn][3])

So the first major rule:

# **Different business processes / incompatible grains → separate fact tables.**

---

# PART 3 — What exactly is a FACT?

Don't memorize:

> "Fact table contains numbers."

That's too weak.

A better definition:

> **A fact table represents a measurable business event, activity, or state at a clearly defined grain.**

Microsoft currently describes fact tables as storing measurements associated with observations/events such as sales or stock balances, together with dimension keys that define their dimensional context. ([Microsoft Learn][3])

Think:

```text
FACT
=
WHAT HAPPENED?
OR
WHAT STATE DID WE OBSERVE?
```

Examples:

```text
Sale
Payment
Shipment
Return
Inventory balance
Website click
Bank transaction
```

---

# PART 4 — What exactly is a DIMENSION?

A dimension describes the **context** around a fact.

Think:

```text
FACT
= what happened

DIMENSIONS
= who / what / where / when / which category
```

For a sale:

```text
Who?
→ Customer

What?
→ Product

Where?
→ Store

When?
→ Date

Which campaign?
→ Promotion
```

So:

```text
                DimCustomer
                     |
                     |
DimProduct ---- FactSales ---- DimDate
                     |
                     |
                 DimStore
                     |
                     |
               DimPromotion
```

Microsoft's current dimensional-modeling guidance uses exactly this conceptual split: dimensions describe business entities/context, while facts store measurements/events. ([Microsoft Learn][1])

---

# PART 5 — The MOST IMPORTANT concept: GRAIN

This is the thing I want permanently in your brain.

# **Grain = What does one row represent?**

Before creating a fact table, write:

> **One row = __________**

You must be able to fill that sentence.

For FlashCart sales, possible grains are:

```text
One row = one order
```

or:

```text
One row = one order line
```

or:

```text
One row = one customer per day
```

or:

```text
One row = one store-product-day
```

These are completely different models.

---

# PART 6 — Why grain changes everything

Suppose one order contains:

```text
Order
 ├── Product A
 ├── Product B
 └── Product C
```

If your grain is:

```text
one row = one order
```

you cannot naturally answer detailed questions like:

> How many units of Product A were sold?

If your grain is:

```text
one row = one order line
```

you can.

So:

```text
BUSINESS QUESTION
       ↓
NEEDED DETAIL
       ↓
GRAIN
       ↓
FACT DESIGN
```

This is why you should **never start with the column list**.

Start with the business question.

Microsoft's current fact guidance calls declaring the grain the first and most important fact-table design step. ([Microsoft Learn][3])

---
