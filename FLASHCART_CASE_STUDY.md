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


# PART 13 — Multiple FACT tables

Now we move to the real enterprise.

FlashCart has another requirement:

> "Finance wants payment analytics."

Is payment the same business process as sales?

No.

So we create:

```text
FactPayment
────────────────────────────
payment_key
payment_id
order_id

customer_key
store_key
date_key
payment_method_key

attempt_number
payment_status
payment_amount
processing_fee
```

### Grain

```text
ONE ROW = ONE PAYMENT ATTEMPT
```

Not order line.

That matters enormously.

---

# PART 14 — Why FactSales and FactPayment must stay separate

Imagine:

```text
Order
 ├── Product A
 ├── Product B
 ├── Product C
 └── Product D
```

and the payment process:

```text
Payment attempt 1 → Failed
Payment attempt 2 → Failed
Payment attempt 3 → Successful
```

If you combine them:

```text
4 order lines × 3 payment attempts
```

You can produce a multiplication effect.

You may accidentally make the business think:

> "Revenue is three times larger."

This is called a **grain mismatch / fanout problem**.

This is one of the reasons a senior engineer asks for the grain before writing a query.

---

# PART 15 — Our second STAR

```text
                 DimCustomer
                      |
                      |
DimPaymentMethod -- FactPayment -- DimDate
                      |
                      |
                  DimStore
```

Now we have:

```text
STAR 1
FactSales
   +
Sales dimensions

STAR 2
FactPayment
   +
Payment dimensions
```

This is normal enterprise dimensional modeling.

---

# PART 16 — Shared dimensions

But notice:

```text
FactSales → DimCustomer
FactPayment → DimCustomer
```

Could both use the same customer dimension?

# Yes.

That's a:

# **CONFORMED DIMENSION**

Example:

```text
                       DimCustomer
                     /            \
                    /              \
                   ↓                ↓
              FactSales       FactPayment
```

Similarly:

```text
                       DimDate
                     /        \
                    /          \
                   ↓            ↓
              FactSales      FactPayment
```

Microsoft's current guidance specifically describes conformed dimensions as shared dimensions reused across multiple fact tables/stars and highlights Date and Product as common examples. ([Microsoft Learn][4])

---

# PART 17 — Why shared dimensions matter

Suppose Sales says:

```text
Country = India
```

and Payments uses a completely different definition of country.

Now management asks:

> "Compare sales and payment activity by country."

You can get inconsistent answers.

A conformed dimension gives the organization a shared analytical definition.

Think:

```text
ONE Customer Definition
ONE Product Definition
ONE Date Definition
```

shared across business processes.

This is one of the most important enterprise-level modeling ideas.

---

# PART 18 — The BUS MATRIX

Now we're getting into serious warehouse architecture.

Create a matrix:

| Business Process | Customer | Product | Store | Date | Promotion |
| ---------------- | -------- | ------- | ----- | ---- | --------- |
| Sales            | ✓        | ✓       | ✓     | ✓    | ✓         |
| Payment          | ✓        | —       | ✓     | ✓    | —         |
| Delivery         | ✓        | —       | ✓     | ✓    | —         |
| Inventory        | —        | ✓       | ✓     | ✓    | —         |
| Returns          | ✓        | ✓       | ✓     | ✓    | —         |

Think about what this shows.

Rows:

```text
business processes
```

Columns:

```text
dimensions
```

This lets you see which dimensions are shared and should be standardized.

Kimball's current reference material still calls the enterprise bus matrix an essential planning/communication tool: rows are business processes and columns are dimensions, with shared dimensions becoming conformed. ([Kimball Group][5])

---

# PART 19 — Delivery fact


Business requirement:

> "Operations wants to analyze delivery performance."

Business process:

```text
Delivery
```

Grain:

```text
ONE ROW = ONE DELIVERY
```

Fact:

```text
FactDelivery
────────────────────────────
delivery_key
order_id

customer_key
store_key
courier_key
pickup_date_key
delivery_date_key

delivery_status
promised_minutes
actual_minutes
distance_km
delivery_fee
late_minutes
```

Now look:

```text
pickup_date_key
delivery_date_key
```

Both point to:

```text
DimDate
```

That's called a:

# **ROLE-PLAYING DIMENSION**

Same physical dimension.

Different roles.

```text
                  DimDate
                 /       \
                /         \
       Pickup Date     Delivery Date
             \             /
              \           /
               FactDelivery
```

Microsoft's current dimension guidance explicitly uses this pattern for multiple roles such as order date, ship date, and delivery date. ([Microsoft Learn][4])

---

# PART 20 — Inventory is different

Requirement:

> "Management needs to know how much stock each product had in each store every day."

Ask the most important question:

### What does one row represent?

```text
ONE ROW = ONE PRODUCT + ONE STORE + ONE DAY
```

This is not a normal transaction.

It is a **state snapshot**.

So:

```text
FactInventorySnapshot
────────────────────────────
date_key
product_key
store_key

opening_quantity
received_quantity
sold_quantity
returned_quantity
damaged_quantity
closing_quantity
inventory_value
out_of_stock_flag
```

This is a:

# **Periodic Snapshot Fact**

---

# PART 21 — Why inventory isn't just another sales fact

Because:

```text
Sale = event
Inventory = state
```

A sale says:

> something happened.

An inventory snapshot says:

> this is the observed state at this point in time.

That difference determines the fact type.

---

# PART 22 — Fact table types

You should know these three extremely well.

## 1. Transaction fact

```text
One row = one event
```

Examples:

```text
FactSales
FactPayment
FactReturn
```

---

## 2. Periodic snapshot

```text
One row = state at a regular interval
```

Example:

```text
FactInventorySnapshot
```

---

## 3. Accumulating snapshot

Tracks a process through stages.

Example:

```text
Order Created
      ↓
Packed
      ↓
Shipped
      ↓
Delivered
```

Possible fact:

```text
FactOrderFulfillment
────────────────────────────
order_key
customer_key
store_key

created_date_key
packed_date_key
shipped_date_key
delivered_date_key

processing_minutes
shipping_minutes
delivery_minutes
total_cycle_minutes
```

This fact changes as the process progresses.

---

# PART 23 — Factless fact

Now something surprising.

A fact table doesn't always need numeric measures.

Example:

> "Which students attended which classes—

```text
FactClassAttendance
──────────────────────
student_key
class_key
teacher_key
date_key
```

No obvious numeric measure.

But you can ask:

```text
COUNT(*)
```

to measure attendance.

That's a:

# **FACTLESS FACT TABLE**

The rows themselves represent occurrences.

Microsoft's current fact-table guidance explicitly includes factless facts as a valid fact-table design. ([Microsoft Learn][3])

---

# PART 24 — Return fact

Business:

> "Which products are returned and how much money is refunded—

Grain:

```text
ONE ROW = ONE RETURNED ORDER LINE
```

Fact:

```text
FactReturn
────────────────────────────
return_key
order_id
order_line_number

customer_key
product_key
store_key
return_date_key
return_reason_key

quantity_returned
refund_amount
restocking_fee
```

Again:

```text
different process
+
different grain
=
different fact
```

---

# PART 25 — Multiple stars across the enterprise


Now our system looks like:

```text
                          DimCustomer
                     /        |         \
                    /         |          \
                   ↓          ↓           ↓
             FactSales   FactPayment   FactReturn
                |             |             |
                |             |             |
           DimProduct    DimPayment     DimReturnReason
                |
                |
             DimStore
                |
                |
             DimDate
```

And:

```text
FactDelivery
FactInventorySnapshot
```

also use shared dimensions.

So an enterprise warehouse isn't necessarily:

```text
ONE FACT
```

It can be:

```text
MANY FACT TABLES
+
SHARED DIMENSIONS
=
MULTIPLE STAR SCHEMAS
```

This is the exact mental model you were trying to understand earlier.

---

# PART 26 — How do I decide whether to make a NEW fact?

Use this decision process.

## Question 1

### Is this a different business process?

```text
Sales
vs
Payment
```

Yes.

Likely separate facts.

---

### Question 2

### Is the grain different?

```text
Sales:
one order line

Inventory:
one product-store-day
```

Different.

Separate facts.

---

### Question 3

### Can the measures coexist at the same grain?

If yes, they can often belong in the same fact.

Example:

```text
quantity
gross_amount
discount
tax
net_amount
cost
profit
```

All are valid at:

```text
one order line
```

So:

```text
FactSales
```

is appropriate.

---

### Question 4

### Are you creating a fact only because of a different question?

Be careful.

Suppose:

> Revenue by country

and:

> Revenue by product

Those are NOT two fact tables.

Same business process.

Same grain.

Same measures.

One `FactSales` can answer both.

---

# PART 27 — VERY IMPORTANT: Question ≠ Fact Table

This is crucial.

Business questions:

```text
Revenue by country
Revenue by product
Revenue by customer
Revenue by month
Revenue by store
```

All can use:

```text
FactSales
```

You don't create:

```text
FactSalesByCountry
FactSalesByProduct
FactSalesByCustomer
FactSalesByMonth
```

That would be terrible design.

Instead:

```text
FactSales
   |
   +── DimCustomer
   +── DimProduct
   +── DimStore
   +── DimDate
```

Dimensions give you different analysis perspectives.

---

# PART 28 — When WOULD you create another fact for the same broad business area?

Now you're getting into advanced design.

Suppose Sales has:

### Detailed transaction fact

```text
FactSales
grain = one order line
```

But management runs a report thousands of times per hour:

```text
Daily sales by store and product category
```

You might create:

```text
FactSalesDailyAggregate
```

with a different grain:

```text
ONE ROW = ONE STORE + PRODUCT CATEGORY + DAY
```

That's an **aggregate fact**.

It exists for performance or specialized consumption.

Microsoft's current fact-table guidance explicitly recognizes aggregate fact tables as rollups designed to accelerate common queries. ([Microsoft Learn][3])

The key:

> **This is not because the question is different. It's because the required data grain/performance/use case is different.**

---

# PART 29 — Dimensions aren't always tiny

Another beginner mistake:

> "Fact = huge, Dimension = small."

Usually facts are very large and dimensions often have fewer rows, but dimensions can also be very large and wide.

A customer dimension can contain millions of customers and many attributes.

Microsoft's current guidance explicitly notes that dimensions can be millions of rows and potentially hundreds of attributes; what matters is whether the dimension supports the necessary filtering, grouping, and historical analysis. ([Microsoft Learn][4])

So:

> **Dimension is defined by role, not row count.**

---

# PART 30 — Surrogate keys

Now let's solve history.

Suppose:

```text
customer_id
```

comes from the source system.

You could put that directly into your fact.

But warehouses often create:

```text
customer_key
```

as a surrogate key.

Dimension:

```text
DimCustomer
────────────────────────────
customer_key
customer_id
customer_name
country
customer_segment
effective_from
effective_to
is_current
```

Here:

```text
customer_id
=
business/source key

customer_key
=
warehouse surrogate key
```

---

# PART 31 — Why surrogate keys matter