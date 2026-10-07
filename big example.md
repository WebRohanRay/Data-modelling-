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
