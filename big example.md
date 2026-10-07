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
