# Enterprise Data Modeling Masterclass & Reference Architecture

[![Data Engineering](https://img.shields.io/badge/Domain-Data%20Engineering%20%26%20Architecture-blue.svg)](#)
[![Methodology](https://img.shields.io/badge/Methodology-Kimball%20Dimensional%20Modeling-orange.svg)](#)
[![Lakehouse](https://img.shields.io/badge/Target-Gold%20Medallion%20Layer-green.svg)](#)
[![Case Studies](https://img.shields.io/badge/Case%20Studies-GlobalMart%20%7C%20FlashCart-purple.svg)](#)

> **"If your business process is wrong, your grain is wrong. If your grain is wrong, your facts are wrong. If your facts are wrong, your entire data warehouse is a lie."**

---

## 🏛️ Executive Architecture Overview

This repository serves as a definitive, production-grade guide to **Enterprise Dimensional Data Modeling** for modern data warehouses and lakehouses (Snowflake, Databricks Unity Catalog, Google BigQuery, Microsoft Fabric).

Rather than abstract definitions, this repository approaches data modeling through two comprehensive, enterprise-scale case studies:
1. **[GlobalMart 78-Part Deep-Dive (`big example.md`)](./big%20example.md)** — A multinational retail & e-commerce conglomerate spanning 50+ countries, 20,000+ stores, 100M+ customers, and billions of rows.
2. **[FlashCart Quick-Commerce Study (`FLASHCART_CASE_STUDY.md`)](./FLASHCART_CASE_STUDY.md)** — A hyper-growth, on-demand delivery ecosystem operating across orders, micro-fulfillment centers, couriers, real-time inventory, and dispatch logistics.

---

## 🎯 The Core Philosophy: The Senior Architect Mindset

When high-level data engineers and data architects design an enterprise analytical model, they follow a disciplined mental model:

```mermaid
flowchart LR
    A[1. Business Requirements] --> B[2. Identify Business Process]
    B --> C[3. Declare the Grain]
    C --> D[4. Identify Dimensions]
    D --> E[5. Identify Numeric Facts]
    E --> F[6. Star Schema / Bus Matrix]
    F --> G[7. Physical Lakehouse Gold Layer]
```

### Golden Rules of Dimensional Modeling
1. **Never Model from Questions Alone:** Questions change daily; underlying business processes generate events consistently for years.
2. **One Business Process = One Fact Family:** Never cram Sales, Payments, and Logistics Deliveries into a single monolithic fact table.
3. **Respect Grain Invariance:** Every dimension key and every measurement in a fact table must exist at the exact declared grain of the row.
4. **Conformed Dimensions are Non-Negotiable:** If `DimCustomer` or `DimProduct` means different things across departments, enterprise cross-process analytics is impossible.
5. **Surrogate Keys Rule the Warehouse:** Never use operational natural keys directly as foreign keys in fact tables.
---

## ?? Corporate Bus Matrix Architecture

The Enterprise Bus Matrix is the master blueprint that connects business processes to conformed dimensions. Each row represents a distinct business process (fact table), and each column represents a shared dimension.

| Business Process / Fact Table | Grain | DimDate | DimCustomer | DimProduct | DimStore | DimWarehouse | DimCourier | DimPromotion |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **FactSalesLine** | One line item per order | ?? | ?? | ?? | ?? | ? | ? | ?? |
| **FactPaymentAttempt** | One transaction attempt | ?? | ?? | ? | ?? | ? | ? | ? |
| **FactDeliveryTrip** | One delivery dispatch | ?? | ?? | ? | ?? | ?? | ?? | ? |
| **FactInventorySnapshot** | One product in warehouse / day | ?? | ? | ?? | ?? | ?? | ? | ? |
| **FactProcurementReceipt** | One goods receipt line | ?? | ? | ?? | ? | ?? | ? | ? |
| **FactCustomerReturn** | One returned item line | ?? | ?? | ?? | ?? | ? | ? | ? |

> **Architect's Principle:** Two disparate facts can only be drilled across and compared directly in business reporting if they share at least one **conformed dimension** (e.g. comparing sales revenue and inventory holding value by DimProduct and DimDate).

---

## ?? Grain Declaration & Numeric Fact Taxonomy

Every measurement stored in an enterprise analytical fact table must fall into one of three mathematical categories:

`	ext
+------------------------------------------------------------------------+
�                        NUMERIC MEASURE TAXONOMY                        �
+------------------------------------------------------------------------�
� 1. Fully Additive  � Sums across all dimensions (Time, Store, Product) �
�                    � Example: sales_quantity, revenue_amount, discount �
+--------------------+---------------------------------------------------�
� 2. Semi-Additive   � Sums across some dimensions, NOT across Time      �
�                    � Example: inventory_on_hand, account_balance       �
+--------------------+---------------------------------------------------�
� 3. Non-Additive    � Cannot be summed across any dimension (Ratios, %) �
�                    � Example: unit_price, profit_margin_pct, discount% �
+------------------------------------------------------------------------+
`

### The Cardinal Rule of Semi-Additive Facts
Never SUM() inventory balances across dates. A warehouse holding 50 laptops on Monday and 50 laptops on Tuesday does not have 100 laptops in inventory. Use AVG(), FIRST_VALUE(), or LAST_VALUE() over the temporal dimension.


---

## ⏳ Slowly Changing Dimensions (SCD) Engineering Patterns

In enterprise systems, entities change over time (e.g., a customer changes tiers, a store relocates, a product category is reclassified).

| Pattern | Strategy | Warehouse Consequence |
| :--- | :--- | :--- |
| **SCD Type 0** | Retain Original | Attribute never changes (e.g., date of birth) |
| **SCD Type 1** | Overwrite In-Place | Destroys historical reality; rewrites past reporting |
| **SCD Type 2** | Add New Versioned Row (Standard) | Full historical truth preserved with valid_from/valid_to |
| **SCD Type 3** | Add Previous Column | Preserves exactly one prior state (rarely scalable) |
| **SCD Type 4** | Mini-Dimension Outrigger | High-frequency changing attributes split out into mart |

### Production SCD Type 2 Table Blueprint
```sql
CREATE TABLE gold.dim_customer (
    customer_sk         BIGINT NOT NULL,          -- Surrogate key for warehouse joins
    customer_nk         VARCHAR(64) NOT NULL,      -- Natural/Business key from source OLTP
    customer_name       VARCHAR(255),
    customer_tier       VARCHAR(50),               -- Gold, Silver, Platinum
    city                VARCHAR(100),
    country             VARCHAR(100),
    is_current          BOOLEAN NOT NULL,          -- TRUE for the active version
    valid_from          TIMESTAMP NOT NULL,        -- Effective start timestamp
    valid_to            TIMESTAMP NOT NULL         -- Effective end timestamp (or 9999-12-31)
);
```


---

## 🏅 Medallion Architecture & Dimensional Placement

Modern Lakehouse architectures (Databricks, Fabric, Snowflake) place dimensional modeling squarely within the **Gold Layer**:

```text
┌─────────────────┐      ┌─────────────────────────┐      ┌─────────────────────────┐
│  BRONZE LAYER   │ ---> │      SILVER LAYER       │ ---> │       GOLD LAYER        │
│   (Raw Data)    │      │ (Cleaned & Conformed)   │      │  (Dimensional Models)   │
├─────────────────┤      ├─────────────────────────┤      ├─────────────────────────┤
│ • Append-only   │      │ • Deduplicated          │      │ • Star Schemas          │
│ • Raw payloads  │      │ • Validated schema      │      │ • Conformed Dimensions  │
│ • CDC streams   │      │ • Normalized/3NF tables │      │ • Business Fact Tables  │
│ • Preserved API │      │ • Silver entities       │      │ • Sub-second BI Marts   │
└─────────────────┘      └─────────────────────────┘      └─────────────────────────┘
```

> **Crucial Rule:** The Gold layer does not mirror the Silver layer. Silver models operational data fidelity; Gold models business questions, analytical grains, and cross-process decision making.
