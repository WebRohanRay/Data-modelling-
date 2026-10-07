Yes. Let's do **one giant enterprise case study** and use it to teach the whole thing from requirements → business processes → grain → facts → dimensions → multiple stars → SCD → snapshots → factless facts → bridges → conformed dimensions → Gold layer → physical design → data quality → performance.

I’ll deliberately make it complex enough that you have to **think like a Data Engineer/warehouse architect**, not memorize textbook definitions.

Current Microsoft guidance still treats star schema, fact/dimension separation, consistent grain, surrogate keys, role-playing dimensions, SCDs, and multiple fact tables as core dimensional-modeling practices. Databricks’ current 2026 guidance also places dimensional fact/dimension models naturally in the Gold layer of a medallion/lakehouse architecture. ([Microsoft Learn][1])

# MASTER CASE STUDY: GLOBALMART

Imagine a company:

# **GlobalMart**

A multinational e-commerce + physical retail company.

It has:

```text
50+ countries
20,000+ stores
500+ distribution centers
100M+ customers
10M+ products
billions of sales lines
millions of payment attempts
millions of deliveries
millions of returns
billions of inventory observations
```

It sells through:

```text
Website
Mobile App
Physical Stores
Marketplace Partners
```

It operates:

```text
Customer Service
Warehouses
Stores
Delivery Fleet
Suppliers
Promotions
Loyalty Program
```

And now management comes to you.

---


# PART 1 — THE BUSINESS REQUIREMENTS

This is the most important starting point.

The business gives you this requirement document.

## Sales team

They want:

```text
Revenue by:
- country
- region
- store
- product
- category
- customer
- sales channel
- promotion
- day
- month
- quarter
- year

They also need:
- quantity sold
- gross sales
- discount
- tax
- net sales
- cost
- profit
```

---

## Finance team

They want:

```text
Payment attempts
Successful payments
Failed payments
Payment failure reasons
Payment methods
Payment providers
Fees
Collected amount
Refunds
```

---

## Logistics team

They want:

```text
Order fulfillment time
Packing time
Shipping time
Delivery time
Late deliveries
Courier performance
Distance
Delivery cost
```

---

## Inventory team

They want:

```text
Daily inventory by product and location
Opening stock
Closing stock
Received stock
Sold stock
Returned stock
Damaged stock
Out-of-stock periods
Inventory value
```

---

## Procurement team

They want:

```text
Purchase orders
Supplier performance
Ordered quantity
Received quantity
Rejected quantity
Purchase cost
Lead time
```

---

## Returns team

They want:

```text
Products returned
Return reasons
Refund amount
Return rate
Return by customer
Return by store
Return by product
```

---

## Marketing

They want:

```text
Campaign performance
Promotion usage
Discount
Revenue generated
Customer acquisition
Customer retention
Customer lifetime value
```

---

## Customer support

They want:

```text
Support tickets
Issue category
Resolution time
Customer
Product
Order
Agent
Satisfaction
```

---

## Executive dashboard

CEO wants:

```text
Revenue
Profit
Orders
Customers
Inventory
Returns
Delivery performance
Payment success
Marketing performance
```

Now stop.

**We have not created a single table yet.**

And that's intentional.

---


### Departmental Analytical Architecture & Requirements Flowchart

```mermaid
graph TD
    subgraph Departmental Stakeholders
        S1["Sales Team<br/>Gross/Net Rev, Margins, Discounts"]
        S2["Finance Team<br/>Payment Auth/Failures, Fee Analysis"]
        S3["Supply Chain<br/>Warehouse Stockouts, Safety Stock"]
        S4["Logistics Team<br/>Delivery SLAs, Carrier Speed"]
        S5["Marketing Team<br/>CAC, ROAS, Campaign Attribution"]
        S6["Procurement<br/>Supplier Lead Times, PO Fill Rate"]
        S7["CX Support<br/>Resolution Time, CSAT, Ticket Volume"]
        S8["Executive Office<br/>Consolidated Global GMV & Growth"]
    end

    subgraph Analytical Ingestion Engine
        INGEST["Central Enterprise Ingestion & Conformance Engine"]
    end

    subgraph Gold Dimensional Warehouse
        G1["Sales Star Schema"]
        G2["Finance & Payment Star"]
        G3["Fulfillment & Delivery Star"]
        G4["Inventory Snapshot Star"]
        G5["Procurement & PO Star"]
        G6["Customer Service Star"]
    end

    S1 --> INGEST
    S2 --> INGEST
    S3 --> INGEST
    S4 --> INGEST
    S5 --> INGEST
    S6 --> INGEST
    S7 --> INGEST
    S8 --> INGEST

    INGEST --> G1 & G2 & G3 & G4 & G5 & G6
```
> **Key Architecture Takeaway**: Notice that each departmental requirement reflects a distinct business process with differing event triggers and natural grains. Attempting to force all these requirements into one table creates grain confusion and data corruption.


# PART 2 — YOUR FIRST JOB IS NOT TABLE DESIGN

Your first job is:

# Identify the BUSINESS PROCESSES.

From the requirements:

```text
Sales
Payments
Fulfillment
Delivery
Inventory
Procurement
Returns
Marketing
Customer Support
Customer Activity
```

Now we've moved from:

```text
business questions
```

to:

```text
business processes
```

That's the first big architectural step.

Kimball's current dimensional-modeling guidance still starts with the business process because the selected process becomes the design target from which grain, dimensions, and facts are defined. ([Kimball Group][2])

---


# PART 3 — WHY BUSINESS PROCESS COMES FIRST

Suppose management says:

> "Show revenue by country."

You don't create:

```text
FactRevenueByCountry
```

Why?

Because that's a **question**, not necessarily a business process.

Revenue was generated by some process.

Probably:

```text
Sales
```

Then you identify:

```text
Sales → Order → Order Line
```

Then determine the grain.

So:

```text
Question
 ↓
Business process
 ↓
Event/state
 ↓
Grain
 ↓
Fact
```

This is the chain I want you to remember.

---


# PART 4 — BUILD A BUS MATRIX

Now let's make an enterprise bus matrix.

| Business process | Customer | Product | Store | Date | Promotion | Supplier | Courier | Payment Method |
| ---------------- | -------- | ------- | ----- | ---- | --------- | -------- | ------- | -------------- |
| Sales            | ✓        | ✓       | ✓     | ✓    | ✓         | —        | —       | —              |
| Payment          | ✓        | —       | ✓     | ✓    | —         | —        | —       | ✓              |
| Fulfillment      | ✓        | —       | ✓     | ✓    | —         | —        | ✓       | —              |
| Delivery         | ✓        | —       | ✓     | ✓    | —         | —        | ✓       | —              |
| Inventory        | —        | ✓       | ✓     | ✓    | —         | —        | —       | —              |
| Procurement      | —        | ✓       | ✓     | ✓    | —         | ✓        | —       | —              |
| Returns          | ✓        | ✓       | ✓     | ✓    | —         | —        | —       | —              |
| Support          | ✓        | ✓       | ✓     | ✓    | —         | —        | —       | —              |

This matrix answers a major question:

> **Which dimensions should be shared between different fact tables?**

For example:

```text
DimCustomer
DimProduct
DimStore
DimDate
```

may become **conformed dimensions**.

Kimball's enterprise bus matrix specifically uses business processes as rows and dimensions as columns to identify shared/conformed dimensions. ([Kimball Group][3])

---


### Enterprise Data Warehouse Bus Matrix Visual Relationship Map

```mermaid
graph LR
    subgraph Core Business Processes
        BP_SALES["Retail & E-Com Sales"]
        BP_PAY["Payment Processing"]
        BP_SHIP["Delivery & Logistics"]
        BP_INV["Inventory Snapshots"]
        BP_PROC["Procurement POs"]
        BP_RET["Merchandise Returns"]
        BP_SUPP["Customer Support"]
    end

    subgraph Conformed Dimensional Bus
        D_DATE["DimDate (Calendar & Fiscal)"]
        D_PROD["DimProduct (SKU Hierarchy)"]
        D_CUST["DimCustomer (Demographics)"]
        D_STORE["DimStore (Physical & Digital)"]
        D_CHAN["DimSalesChannel (Omnichannel)"]
        D_SUPP["DimSupplier (Vendor Network)"]
    end

    BP_SALES === D_DATE & D_PROD & D_CUST & D_STORE & D_CHAN
    BP_PAY === D_DATE & D_CUST & D_STORE & D_CHAN
    BP_SHIP === D_DATE & D_CUST & D_STORE
    BP_INV === D_DATE & D_PROD & D_STORE
    BP_PROC === D_DATE & D_PROD & D_SUPP
    BP_RET === D_DATE & D_PROD & D_CUST & D_STORE
    BP_SUPP === D_DATE & D_CUST & D_CHAN
```
> **Bus Matrix Rule**: The rows represent business processes; the columns represent conformed dimensions. An intersection signifies that the dimensional context is natively valid at that process grain.


# PART 5 — NOW DESIGN ONE PROCESS AT A TIME

Let's start with:

# SALES

Business requirement:

> "We need product-level revenue analysis by customer, product, store, channel, promotion and time."

Now think.

---

## Step 1 — Business process

```text
Sales
```

## Step 2 — What happened?

```text
A product was sold.
```

## Step 3 — What level of detail?

The business wants:

```text
product-level
```

So ask:

> What should one row represent?

Our decision:

# ONE ROW = ONE PRODUCT LINE IN ONE ORDER

That's our grain.

This is the critical decision.

Microsoft currently emphasizes that the fact-table grain should be explicitly defined and consistently maintained; its current guidance also notes that storing data at too high a grain can make later detailed analysis impossible. ([Microsoft Learn][1])

---


# PART 6 — BUILD FACTSALES

```text
FactSales
────────────────────────────────────
sales_key
order_id
order_line_number

customer_key
product_key
store_key
date_key
channel_key
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

Notice the structure.

### Keys

```text
customer_key
product_key
store_key
date_key
channel_key
promotion_key
```

These connect the fact to dimensions.

### Measures

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

### Business identifiers

```text
order_id
order_line_number
```

These can remain as degenerate/business identifiers in the fact.

---


### FactSales Atomic Grain Resolution Flowchart

```mermaid
flowchart TD
    CART["Customer Checkout Order<br/>OrderID #98721 (Multi-Item Cart)"] --> SPLIT["Line Item Decomposition"]
    SPLIT --> L1["Item 1: 4K Monitor (SKU-A)"]
    SPLIT --> L2["Item 2: HDMI Cable (SKU-B)"]
    SPLIT --> L3["Item 3: USB Hub (SKU-C)"]

    subgraph Atomic Line Processing
        L1 --> G1["Atomic Grain Declaration:<br/>1 Row = 1 Product Line Item in 1 Order"]
        G1 --> SURR["Surrogate Key Lookup Engine"]
        SURR --> KEYS["Resolve DateKey, ProductKey, CustomerKey, StoreKey, PromotionKey"]
        KEYS --> METRICS["Compute Fully Additive Measures:<br/>Quantity, GrossAmount, DiscountAmount, TaxAmount, NetAmount, CostAmount, MarginAmount"]
        METRICS --> INSERT["Insert FactSales Atomic Record"]
    end
```
> **Fact Table Grain Rule**: Never store order-level summary totals in an order-line fact table without allocating them, because doing so creates aggregation errors when summing across dimensions.


# PART 7 — WHY DID WE NOT PUT PRODUCT NAME IN FACTSALES?

Bad design:

```text
FactSales
─────────────────────────
order_id
product_id
product_name
brand
category
customer_name
country
store_name
city
amount
```

You are repeating descriptive information billions of times.

Instead:

```text
FactSales
    |
product_key
    ↓
DimProduct
```

And:

```text
FactSales
    |
customer_key
    ↓
DimCustomer
```

The fact represents **the event**.

Dimensions describe the event.

Current Microsoft guidance explicitly defines dimensions as descriptive entities and facts as event/measurement tables used in analytical aggregation. ([Microsoft Learn][4])

---


# PART 8 — DESIGN THE DIMENSIONS

## DimCustomer

```text
DimCustomer
────────────────────────────────
customer_key
customer_id
customer_name
customer_segment
gender
date_of_birth
country
state
city
postal_code
signup_date
acquisition_channel
loyalty_tier
```

Questions this answers:

```text
Who purchased?
Which segment?
Which country?
Which city?
Which loyalty tier?
```

---

## DimProduct

```text
DimProduct
────────────────────────────────
product_key
product_id
product_name
brand
category
subcategory
department
product_type
unit
size
color
supplier_category
```

Questions:

```text
What was sold?
Which category?
Which brand?
Which department?
```

---

## DimStore

```text
DimStore
────────────────────────────────
store_key
store_id
store_name
store_type
city
state
country
region
store_format
opening_date
```

---

## DimDate

```text
DimDate
────────────────────────────────
date_key
full_date
day
week
month
month_name
quarter
year
day_of_week
is_weekend
is_holiday
fiscal_month
fiscal_quarter
fiscal_year
```

---

## DimChannel

```text
DimChannel
────────────────────────
channel_key
channel_code
channel_name
channel_type
device_type
```

---

## DimPromotion

```text
DimPromotion
────────────────────────────────
promotion_key
promotion_id
promotion_name
campaign_name
promotion_type
discount_type
campaign_start_date
campaign_end_date
```

---


### Dimension Attribute Hierarchies & Normalization Mapping

```mermaid
graph TD
    subgraph Product Dimension Hierarchy
        SKU["SKU Level (Lowest Grain)"] --> SUBCAT["Sub-Category (e.g., Wireless Mice)"]
        SUBCAT --> CAT["Category (e.g., Computer Accessories)"]
        CAT --> DEPT["Department (e.g., Electronics)"]
    end

    subgraph Store Geography Hierarchy
        STORE["Store ID (Retail Location / DC)"] --> CITY["City (e.g., Seattle)"]
        CITY --> STATE["State / Province (e.g., WA)"]
        STATE --> REGION["Territory / Region (e.g., Pacific NW)"]
        REGION --> CTRY["Country (e.g., United States)"]
    end

    subgraph Calendar Dimension Hierarchy
        DAY["Date / Day (YYYY-MM-DD)"] --> MONTH["Fiscal / Calendar Month"]
        MONTH --> QTR["Fiscal Quarter (Q1-Q4)"]
        QTR --> YEAR["Calendar Year (YYYY)"]
    end
```
> **Dimensional Design Principle**: In dimensional modeling, attribute hierarchies are deliberately denormalized into wide dimension tables to eliminate multi-table joins and optimize BI self-service query performance.


# PART 9 — OUR SALES STAR

```text
                         DimCustomer
                              |
                              |
DimProduct ------------- FactSales ------------- DimDate
                              |
                              |
                          DimStore
                              |
                              |
                         DimChannel
                              |
                              |
                        DimPromotion
```

That is one star.

But GlobalMart needs much more.

---
