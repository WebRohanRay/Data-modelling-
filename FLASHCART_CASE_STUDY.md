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


# PART 7 — Our FlashCart Sales requirement

Business says:

> "We want product-level sales analysis by customer, store, date, promotion, and product."

Now design.

## Step 1 — Business process

```text
Sales
```

## Step 2 — Grain

```text
ONE ROW = ONE PRODUCT LINE IN ONE CUSTOMER ORDER
```

This is an order-line transaction fact.

## Step 3 — Dimensions

We need:

```text
DimCustomer
DimProduct
DimStore
DimDate
DimPromotion
```

## Step 4 — Measures

We need things like:

```text
quantity
unit_price
gross_amount
discount_amount
tax_amount
net_amount
cost_amount
profit_amount
```

## Step 5 — Fact

```text
FactSales
────────────────────────────
sales_key
order_id
order_line_number

customer_key
product_key
store_key
date_key
promotion_key

quantity
unit_price
gross_amount
discount_amount
tax_amount
net_amount
cost_amount
profit_amount
```

Notice:

We didn't randomly invent `FactSales`.

We derived it.

```text
Requirement
     ↓
Sales process
     ↓
Order-line grain
     ↓
Dimensions
     ↓
Measures
     ↓
FactSales
```

---

# PART 8 — Our dimensions

## DimCustomer

```text
DimCustomer
────────────────────────────
customer_key
customer_id
customer_name
customer_segment
gender
country
state
city
signup_date
acquisition_channel
```

What is this table doing?

It answers:

```text
Who bought?
Which country?
Which segment?
Which city?
Which acquisition channel?
```

---

## DimProduct

```text
DimProduct
────────────────────────────
product_key
product_id
product_name
brand
category
subcategory
product_type
unit
size
```

It answers:

```text
What was sold?
Which brand?
Which category?
Which subcategory?
```

---

## DimStore

```text
DimStore
────────────────────────────
store_key
store_id
store_name
store_type
city
state
country
region
opening_date
```

---

## DimDate

```text
DimDate
────────────────────────────
date_key
full_date
day
week
month
month_name
quarter
year
day_name
is_weekend
is_holiday
```

---

## DimPromotion

```text
DimPromotion
────────────────────────────
promotion_key
promotion_id
promotion_name
promotion_type
campaign
discount_type
start_date
end_date
```

---

# PART 9 — Our first STAR SCHEMA

Now put it together.

```text
                         DimCustomer
                              |
                              |
                              |
DimProduct ------------- FactSales ------------- DimDate
                              |
                              |
                         DimStore
                              |
                              |
                        DimPromotion
```

That is a **star schema**.

Why is it called a star?

Because:

```text
                DIM
                 |
DIM ----------- FACT ----------- DIM
                 |
                DIM
```

The fact sits in the center.

Microsoft's current Fabric documentation still describes star schema as a recommended analytical modeling approach, and explicitly notes that one warehouse commonly contains multiple fact tables and therefore multiple stars. ([Microsoft Learn][1])

---

# PART 10 — Why fact in center?

Because most analytical questions look like:

> Give me a metric **by some dimensions**.

For example:

```text
Revenue
BY
country
BY
product category
BY
month
```

The fact supplies:

```text
revenue
```

The dimensions supply:

```text
country
product category
month
```

So the query logically becomes:

```text
FactSales
    ↓
join/filter
    ↓
DimCustomer
DimProduct
DimDate
    ↓
aggregate
    ↓
answer
```

This is why dimensional modeling is so useful for analytics.

---

# PART 11 — Measures vs dimensions

This distinction must become automatic.

Suppose you have:

```text
Country
Product
Category
Month
Revenue
Quantity
Profit
```

Then:

### Dimensions

```text
Country
Product
Category
Month
```

### Measures

```text
Revenue
Quantity
Profit
```

A simple rule:

> **Dimension = something you group/filter by**

> **Measure = something you calculate/summarize**

Example:

```sql
SUM(revenue)
GROUP BY country
```

`revenue` is a measure.

`country` is a dimension attribute.

---

# PART 12 — Additive, semi-additive, non-additive facts

This is where your knowledge starts becoming more advanced.

## Additive

Can be summed across all relevant dimensions.

Examples:

```text
sales_amount
quantity
cost
discount_amount
```

For example:

```text
SUM(sales_amount)
```

usually makes sense across:

```text
customer
product
store
date
```

---

## Semi-additive

Can be summed across some dimensions but not all.

Classic example:

```text
bank_account_balance
inventory_balance
```

You can perhaps sum balances across:

```text
customers
```

but summing the same balance across time can be meaningless.

If:

```text
Monday balance = 100
Tuesday balance = 120
Wednesday balance = 90
```

then:

```text
100 + 120 + 90
```

doesn't mean "Wednesday inventory."

Instead you might use:

```text
LAST_VALUE
AVERAGE
MIN
MAX
```

depending on the business question.

---

## Non-additive

Cannot meaningfully be summed.

Examples:

```text
percentage
ratio
unit price
conversion rate
```

For example:

```text
conversion_rate
```

You usually don't:

```text
SUM(conversion_rate)
```

You calculate it from appropriate underlying measures.

### Senior-level thought

Don't ask:

> "Is this column numeric—

Ask:

> **"How is this measure mathematically valid to aggregate—**

---
