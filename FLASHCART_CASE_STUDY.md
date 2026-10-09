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


Suppose customer 101 changes country.

Old:

```text
customer_id = 101
country = India
```

New:

```text
customer_id = 101
country = Philippines
```

The business ID did not change.

But the **historical state** did.

With SCD Type 2:

```text
customer_key 10101
customer_id 101
country India
```

and later:

```text
customer_key 10102
customer_id 101
country Philippines
```

Now facts can point to the correct historical version.

---

# PART 32 — Slowly Changing Dimensions

This is essential.

## Type 0

Never change it.

```text
Original value stays forever
```

Memory:

> **0 = freeze**

---

## Type 1

Overwrite.

```text
India
   ↓
Philippines
```

India disappears.

Memory:

> **1 = overwrite**

---

## Type 2

Create a new version.

```text
India        ← old version
Philippines  ← current version
```

Memory:

> **2 = preserve history**

This is the one you should know deeply.

Databricks' current Lakeflow dimensional-modeling guidance explicitly supports SCD Type 2 for Gold-layer dimensions when historical tracking is required. ([Databricks Documentation][6])

---

# PART 33 — Why NOT every dimension attribute should use SCD2

This is important.

Suppose:

```text
customer_name
customer_segment
country
email
```

change.

Do you need historical versions of all of them?

Not necessarily.

The business decides.

Example:

### Name typo

```text
Rohn
↓
Rohan
```

Maybe Type 1.

### Customer segment

```text
Silver
↓
Gold
```

Maybe Type 2 because analysts want historical segmentation.

Senior thinking:

> **SCD strategy is driven by business meaning and historical requirements, not by a blanket rule.**

---

# PART 34 — Junk dimension

Suppose your sales event has tiny flags:

```text
is_gift
is_online
is_first_order
is_coupon_used
```

Creating separate dimensions for each would be silly.

You can combine them:

```text
DimOrderFlags
────────────────────────
flag_key
is_gift
is_online
is_first_order
is_coupon_used
```

That's a:

# **Junk dimension**

---

# PART 35 — Degenerate dimension

Suppose:

```text
order_number
```

is useful for identifying a transaction but doesn't need its own dimension.

Keep it directly in:

```text
FactSales
```

Example:

```text
FactSales
────────────────
order_number
customer_key
product_key
date_key
quantity
net_amount
```

This is a:

# **Degenerate dimension**

The important thing isn't the name.

Understand:

> **A business identifier can live in the fact without becoming a separate dimension.**

---

# PART 36 — Role-playing dimension

We already saw Date.

One fact can reference the same dimension multiple times.

Example:

```text
FactShipment
────────────────
order_date_key
ship_date_key
delivery_date_key
```

All reference:

```text
DimDate
```

Different roles.

Another example:

```text
FactFlight
────────────────
departure_airport_key
arrival_airport_key
```

Both can reference:

```text
DimAirport
```

Microsoft's current dimensional modeling documentation explicitly describes this pattern. ([Microsoft Learn][4])

---

# PART 37 — Conformed dimension


Same dimension used across multiple stars.

Example:

```text
                 DimProduct
                 /    |     \
                /     |      \
               ↓      ↓       ↓
          FactSales FactReturn FactInventory
```

The product definition is shared.

This means everyone agrees:

```text
Product A
Category X
Brand Y
```

has the same analytical meaning across processes.

---

# PART 38 — Hierarchies

Dimensions often contain levels.

Product:

```text
Department
   ↓
Category
   ↓
Subcategory
   ↓
Product
```

Location:

```text
Country
   ↓
State
   ↓
City
   ↓
Store
```

Date:

```text
Year
 ↓
Quarter
 ↓
Month
 ↓
Week
 ↓
Day
```

These help users drill from high level to detail.

---

# PART 39 — Star vs Snowflake schema

Star:

```text
             DimProduct
                  |
                  |
DimCustomer --- FactSales --- DimDate
                  |
               DimStore
```

Snowflake:

```text
             DimCategory
                  |
             DimProduct
                  |
DimCustomer --- FactSales --- DimDate
                  |
               DimStore
```

Snowflake normalizes parts of the dimension.

### Star

Usually:

* simpler
* easier for BI
* fewer joins
* more denormalized

### Snowflake

Usually:

* more normalized
* more tables
* more joins

Don't memorize:

> "Star is always better."

Instead:

> **Choose based on workload, usability, governance, performance and platform behavior.**

Microsoft currently recommends star schema for Fabric analytical workloads, but modern cloud systems can use variations when the workload calls for them. ([Microsoft Learn][1])

---

# PART 40 — Normalized model vs dimensional model

Now connect this with what you learned earlier.

A transactional application might look like:

```text
Customer
   ↓
Order
   ↓
OrderItem
   ↓
Product
```

This is often highly normalized.

Why?

Because the application needs:

```text
consistent updates
transaction integrity
reduced duplication
```

Analytical systems often reshape this into:

```text
             DimCustomer
                  |
DimProduct --- FactSales --- DimDate
                  |
               DimStore
```

Why?

Because analysts want:

```text
filter
group
aggregate
compare
drill down
```

So:

```text
OLTP
→ optimized for operating the business

OLAP
→ optimized for analyzing the business
```

That distinction is foundational.

---

# PART 41 — Conceptual vs Logical vs Physical modeling

You also need to know these.

## Conceptual model

Just business concepts:

```text
Customer
Product
Order
Payment
Store
```

No implementation details.

---

## Logical model

Relationships and attributes:

```text
Customer
---------
customer_id
name
country

Order
---------
order_id
customer_id
order_date
```

---

## Physical model

Actual implementation:

```text
customer_key BIGINT
customer_id VARCHAR(50)
customer_name VARCHAR(200)
country_code CHAR(2)
```

Now you think about:

```text
data types
partitioning
clustering
storage
indexes
platform
performance
```

Memory:

```text
Conceptual = WHAT
Logical    = HOW IT RELATES
Physical   = HOW IT IS IMPLEMENTED
```

---

# PART 42 — ER modeling vs dimensional modeling

They are not the same thing.

## ER / relational modeling

Usually focuses on:

```text
entities
relationships
normalization
transaction systems
```

## Dimensional modeling

Focuses on:

```text
business processes
facts
dimensions
grain
analytics
```

A Data Engineer should know both.

But for analytical Data Engineering:

# **Grain + Facts + Dimensions + Star Schema + SCD are especially important.**

---

# PART 43 — Data Lake and Lakehouse enter the picture


Now connect modeling to your current Azure/Databricks roadmap.

A modern lakehouse pipeline can look like:

```text
SOURCE
   ↓
BRONZE
   ↓
SILVER
   ↓
GOLD
   ↓
BI / ML / APPLICATIONS
```

Databricks' current documentation, updated September 11, 2026, describes Bronze as raw, Silver as validated/cleaned, and Gold as enriched/business-ready, with progressive improvement of structure and quality through the layers. ([Databricks Documentation][7])

Now the important distinction:

### Bronze

```text
Raw source data
```

Not where you create your final star model.

### Silver

```text
Cleaned
Validated
Standardized
Reusable
```

Still fairly general-purpose.

### Gold

```text
Business-ready
Analytical
Optimized for consumers
```

This is where a star schema may appear:

```text
Gold
 │
 ├── FactSales
 ├── DimCustomer
 ├── DimProduct
 ├── DimDate
 │
 ├── FactPayment
 ├── DimPaymentMethod
 │
 ├── FactDelivery
 ├── DimCourier
 │
 └── FactInventorySnapshot
```

Databricks' current documentation explicitly describes star schemas fitting naturally in Gold, with Bronze/Silver handling ingestion and cleaning. ([Databricks Documentation][6])

---

# PART 44 — Gold does NOT automatically mean star schema

This is a subtle but important correction.

Don't think:

```text
Gold = star schema
```

Think:

```text
Gold = business-serving layer
```

It might contain:

```text
fact + dimension tables
aggregated tables
wide analytical tables
feature tables
serving tables
semantic-model-ready tables
```

Star schema is **one very important option**.

That's how you avoid becoming dogmatic about architecture.

---

# PART 45 — Modern architecture vs old textbook thinking

A traditional warehouse might look like:

```text
Source
   ↓
ETL
   ↓
Data Warehouse
   ↓
Star Schema
   ↓
BI
```

A modern lakehouse might look like:

```text
Sources
   ↓
ADLS / cloud object storage
   ↓
Bronze
   ↓
Silver
   ↓
Gold
   ↓
Fact + Dimensions
   ↓
BI / ML
```

The modeling principles did not disappear.

The **storage and processing architecture changed**.

That's why you should learn:

```text
Data Modeling
+
Data Warehousing
+
Lakehouse
+
Delta Lake
+
Databricks
```

instead of thinking they're competing definitions.

---

# PART 46 — Current 2026 practice: don't confuse logical model with physical storage

This is a very important high-level concept.

You might logically design:

```text
FactSales
DimCustomer
DimProduct
DimDate
```

But physically store/process the data using:

```text
ADLS
Delta Lake
Databricks
Fabric
Snowflake
BigQuery
```

Those are different decisions.

Think:

```text
LOGICAL DESIGN
       ↓
What does the data mean?
       ↓
FACTS / DIMENSIONS / GRAIN
       ↓
PHYSICAL PLATFORM
       ↓
How do we store/process it?
```

That is why your warehouse concepts transfer across platforms.

---

# PART 47 — What companies/platforms are doing now

The important current pattern isn't "every company uses exactly the same model."

Modern platforms support several architectures.

What is clearly current in 2026:

### Microsoft

Microsoft currently teaches and recommends dimensional/star-schema design for Fabric Warehouse analytical workloads and says star schemas are well suited to filtering, grouping, sorting and summarization. ([Microsoft Learn][1])

### Databricks

Databricks currently documents:

```text
Bronze → Silver → Gold
```

and explicitly documents star-schema fact/dimension models in Gold-layer Lakeflow pipelines. ([Databricks Documentation][7])

### Important conclusion

Modern Data Engineers are not choosing between:

```text
"old warehouse modeling"
```

and:

```text
"modern lakehouse"
```

They often combine them:

```text
Lakehouse architecture
       +
Dimensional analytical modeling
```

That is exactly the kind of combination you should understand.

---

# PART 48 — A full FlashCart enterprise model

Now let's make our final model.

```text
                         ┌──────────────┐
                         │ DimCustomer  │
                         └──────┬───────┘
                                │
                                │
         ┌──────────────────────┼─────────────────────────┐
         │                      │                         │
         ▼                      ▼                         ▼
   ┌───────────┐         ┌────────────┐            ┌────────────┐
   │ FactSales │         │FactPayment │            │FactReturn  │
   └─────┬─────┘         └─────┬──────┘            └─────┬──────┘
         │                     │                         │
         │                     │                         │
         ▼                     ▼                         ▼
   DimProduct           DimPaymentMethod          DimReturnReason
         │
         │
         ├───────────────┐
         │               │
         ▼               ▼
     DimStore          DimDate
         │
         │
         ▼
 FactInventorySnapshot
         │
         │
      DimProduct
         │
      DimStore
         │
      DimDate


 FactDelivery
      │
      ├── DimCustomer
      ├── DimStore
      ├── DimCourier
      ├── DimDate (pickup role)
      └── DimDate (delivery role)
```

This is no longer a beginner's "one fact table" model.

It is an **enterprise dimensional model**.

---

# PART 49 — What you should think when a new requirement arrives


Suppose management says:

> "We need to know which products are frequently out of stock."

Don't immediately create:

```text
FactOutOfStock
```

Ask:

### 1. What business process?

Inventory.

### 2. What is the observation?

Product/store inventory state.

### 3. What is the grain?

Maybe:

```text
one row = one product + one store + one day
```

### 4. What are the measures?

```text
closing_quantity
out_of_stock_flag
inventory_value
```

### 5. What dimensions?

```text
Product
Store
Date
```

### 6. What fact type?

Periodic snapshot.

Now you have a defensible design.

---

# PART 50 — How a HIGH-LEVEL engineer chooses a fact

This is the exact thought process I want from you.

## DON'T THINK:

> "What columns should my fact table have—

## THINK:

> **What business event/state am I representing?**

Then:

> **What does one row mean?**

Then:

> **Which dimensions give the analytical context?**

Then:

> **Which measures belong naturally at that grain?**

Then:

> **How will this connect to other business processes?**

Then:

> **What history is needed?**

Then:

> **What is the expected volume?**

Then:

> **How will this be consumed?**

Then:

> **Do I need an atomic fact, snapshot, aggregate, or something else?**

That is much more sophisticated than memorizing "fact = numeric, dimension = text."

---

# PART 51 — The 10 questions you should ask before designing a fact

Keep this permanently.

```text
1. What business process am I modeling?

2. What exactly does one row represent?

3. What is the grain?

4. Is this an event or a state?

5. What measures exist at that grain?

6. Which dimensions describe those measures?

7. Which dimensions are shared with other facts?

8. Do I need historical tracking?

9. Is the fact atomic, snapshot, accumulating, or aggregate?

10. What business questions must this model answer?
```

If you can answer these, you're doing actual modeling.

---

# PART 52 — The rules you should memorize

Don't memorize 50 definitions.

Memorize these **mental rules**:

### Rule 1

**Business process comes before fact table.**

### Rule 2

**Grain comes before columns.**

### Rule 3

**Every fact must have a clearly declared grain.**

### Rule 4

**Same process + same grain → usually same fact.**

### Rule 5

**Different process or incompatible grain → separate fact.**

### Rule 6

**Different question does NOT automatically mean a new fact.**

### Rule 7

**Fact = event or measurable state.**

### Rule 8

**Dimension = descriptive context.**

### Rule 9

**Measures must make sense at the fact's grain.**

### Rule 10

**Conformed dimensions connect multiple stars.**

### Rule 11

**SCD2 preserves historical dimension state.**

### Rule 12

**Star schema optimizes analytical usability.**

### Rule 13

**Gold is a serving/business layer, not automatically "the star schema."**

### Rule 14

**Logical modeling and physical implementation are different decisions.**

### Rule 15

**Don't optimize for textbook purity; optimize for business requirements, correctness, maintainability and workload.**

---

# PART 53 — The most common mistakes you must avoid

## Mistake 1 — No grain

```text
FactSales
```

but nobody knows what one row means.

**Never do this.**

---

## Mistake 2 — Mixing grains

```text
order line
+
payment attempt
+
delivery
```

in one table.

Bad.

---

## Mistake 3 — One fact for every business question

Bad:

```text
FactSalesByCountry
FactSalesByProduct
FactSalesByCustomer
```

when the same atomic FactSales can answer all three.

---

## Mistake 4 — One giant fact

```text
FactEverything
```

Usually a warning sign.

---

## Mistake 5 — Duplicating dimensions unnecessarily

```text
DimCustomerSales
DimCustomerPayment
DimCustomerReturn
```

when all should use the same conformed customer definition.

---

## Mistake 6 — Storing descriptive attributes everywhere

Putting:

```text
customer_name
customer_country
product_name
category
store_city
```

into every massive fact can create unnecessary duplication.

---

## Mistake 7 — Blindly using SCD2

Not every attribute needs historical versioning.

---

## Mistake 8 — Thinking every numeric column is a measure

An ID can be numeric.

That doesn't make it a business measure.

---

## Mistake 9 — Summing semi-additive metrics

For example, inventory balance across dates.

---

## Mistake 10 — Designing from available columns instead of business questions

The source gives you:

```text
20 columns
```

That doesn't mean the warehouse should have exactly the same 20-column structure.

---

# PART 54 — Now connect this to SQL