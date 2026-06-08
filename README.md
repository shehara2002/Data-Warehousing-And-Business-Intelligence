# 🛒 E-Commerce Data Warehouse & Business Intelligence System

---

## 📌 Project Overview

This project implements a full end-to-end **Data Warehouse and Business Intelligence solution** for the **Fecom Inc. E-Commerce Marketplace**, a CRM-based multi-vendor platform. It covers the complete BI lifecycle — from raw source data ingestion through ETL pipelines, dimensional modelling, OLAP cube construction, and interactive Power BI dashboards.

The solution processes **90,000+ order records** spanning customers, products, sellers, payments, and reviews across multiple European countries (2022–2024).

---

## 🗂️ Dataset

**Source:** [Fecom Inc. E-Commerce Marketplace Dataset – Kaggle](https://www.kaggle.com/datasets/cemeraan/fecom-inc-e-com-marketplace-orders-datacrm?select=Fecom+Inc+Order+Payments.csv)

| File | Format | Records | Description |
|------|--------|---------|-------------|
| Orders | CSV | ~99,441 | Order lifecycle & status timestamps |
| Reviews | CSV | ~99,329 | Customer ratings (1–5) & timestamps |
| Customers | CSV | ~102,727 | Demographics, geography, subscription info |
| Order Payments | Text | ~103,886 | Payment type, installments, value |
| Products | CSV | ~32,951 | Category, weight, dimensions |
| Sellers | Text | ~3,095 | Seller profile & location |
| Order Items | CSV | ~112,651 | Line-item details per order |

---

## 🏗️ Solution Architecture

```
┌─────────────────────┐
│   Source Files      │  CSV / Text flat files (7 tables)
│  (Flat Files)       │
└────────┬────────────┘
         │  SSIS ETL (Extract & Load)
         ▼
┌─────────────────────┐
│   ecommerce_OLTP    │  Raw staging database (SQL Server)
│   (Staging DB 1)    │
└────────┬────────────┘
         │  SSIS ETL (Transform & Load)
         ▼
┌─────────────────────┐
│   ecommerce_stag    │  Typed & cleansed staging layer
│   (Staging DB 2)    │
└────────┬────────────┘
         │  SSIS ETL (Dimensional Load)
         ▼
┌─────────────────────┐
│   ecommerce_WH      │  Star Schema Data Warehouse
│   (Data Warehouse)  │
└────────┬────────────┘
         │
    ┌────┴─────┐
    ▼          ▼
 SSAS Cube   Power BI
 (OLAP)      (Reports)
```

---

## 📐 Data Warehouse Design — Star Schema

The warehouse follows a **Star Schema** with one central fact table and six dimension tables.

### Fact Table
**`FACT_Order_Items`** — Central fact table recording every order line item.

| Column | Type | Description |
|--------|------|-------------|
| Order_Item_SK | INT (PK) | Surrogate key |
| Customer_SK | INT (FK) | → DIM_Customer |
| Product_SK | INT (FK) | → DIM_Product |
| Seller_SK | INT (FK) | → DIM_Seller |
| Payment_SK | INT (FK) | → DIM_Payment |
| Review_SK | INT (FK) | → DIM_Review |
| Date_SK | INT (FK) | → DIM_Date |
| Order_ID | NVARCHAR | Business key |
| Order_Status | NVARCHAR | e.g. delivered, cancelled |
| Price | FLOAT | Item price |
| Freight_Value | FLOAT | Shipping cost |
| Payment_Value | FLOAT | Total payment amount |
| Order_Purchase_Timestamp | DATETIME2 | Purchase time |
| Order_Approved_At | DATETIME2 | Approval time |
| Order_Delivered_Carrier_Date | DATE | Carrier pickup date |
| Order_Delivered_Customer_Date | DATE | Customer receipt date |
| Order_Estimated_Delivery_Date | DATE | Estimated delivery |
| Shipping_Limit_Date | DATETIME2 | Seller shipping deadline |
| accm_txn_create_time | DATETIME2 | Accumulating: txn start |
| accm_txn_complete_time | DATETIME2 | Accumulating: txn end |
| txn_process_time_hours | FLOAT | Processing duration (hrs) |

### Dimension Tables

| Table | Key Attributes | SCD Type |
|-------|---------------|----------|
| **DIM_Customer** | Customer_SK, Gender, Age, City, Country, Postal Code, Effective_Start_Date, Effective_End_Date, Is_Current | **Type 2** |
| **DIM_Date** | Date_SK, Full_Date, Day, Month, Month_Name, Quarter, Year, Weekday_Name | Type 1 |
| **DIM_Product** | Product_SK, Product_ID, Category_Name, Weight_G, Length_Cm, Height_Cm, Width_Cm | Type 1 |
| **DIM_Seller** | Seller_SK, Seller_ID, Seller_Name, Postal_Code, City, Country_Code, Country | Type 1 |
| **DIM_Payment** | Payment_SK, Payment_Sequential, Payment_Type, Payment_Installments | Type 1 |
| **DIM_Review** | Review_SK, Review_ID, Review_Score, Review_Creation_Date, Review_Answer_Timestamp | Type 1 |

---

## ⚙️ ETL Pipeline

Built using **SQL Server Integration Services (SSIS)** in Visual Studio.

### Phase 1 — Load to OLTP (`Load_Staging.dtsx`)
Flat files → `ecommerce_OLTP` database. Handles data type mapping for all 7 source files.

### Phase 2 — Load to Staging (`Load_Staging.dtsx`)
`ecommerce_OLTP` → `ecommerce_stag`. Applies type casting, null handling, and cleansing across all tables:
- `stg_customer_details`
- `stg_sellers`
- `stg_Products`
- `stg_Orders`
- `stg_order_item`
- `stg_order_payment`
- `stg_reviews`

### Phase 3 — Load to Warehouse (`Load_DW.dtsx`)
`ecommerce_stag` → `ecommerce_WH`. Executes in order:
1. **Load_DIM_Customer** — SCD Type 2 with Conditional Split, Sort, Slowly Changing Dimension component, Derived Column, Union All
2. **Load_DIM_Product** — Deduplication & upsert
3. **Load_DIM_Seller** — Deduplication
4. **Load_DIM_Payment** — Sort & Conditional Split
5. **Load_DIM_Review** — Conditional Split & Derived Column
6. **Execute SQL Task** — Populate DIM_Date via T-SQL date generation script
7. **Load_FACT_Order_Items** — Lookup transformations for all 5 dimension SKs + OLE DB Destination

### Phase 4 — Accumulating Fact Update (`Update_WH.dtsx`)
Enriches `FACT_Order_Items` with transaction lifecycle timing:
```sql
ALTER TABLE FACT_Order_Items ADD
    accm_txn_create_time   DATETIME2(7) NULL,
    accm_txn_complete_time DATETIME2(7) NULL,
    txn_process_time_hours FLOAT        NULL;
```
Uses a `txn_complete_updates` staging table and a Lookup + Derived Column + OLE DB Command pattern to calculate `txn_process_time_hours`.

---

## 📊 OLAP Cube (SSAS)

Built as a **Multidimensional SSAS Cube** (`ecommerce_Cube`) in Visual Studio and deployed to SQL Server Analysis Services.

- **Data Source View:** All 7 warehouse tables (1 fact + 6 dimensions)
- **Date Hierarchy:** Year → Quarter → Month → Day
- **Measures:** Payment Value, Freight Value, Price, Order Item Count, Txn Process Time Hours

### OLAP Operations Demonstrated (via Excel PivotTable)

| Operation | Description |
|-----------|-------------|
| **Roll-Up** | Day → Month → Year aggregation of Payment Value |
| **Drill-Down** | Year → Quarter → Month expansion in PivotTable |
| **Slice** | Single slicer on Payment Type (e.g. Credit Card only) |
| **Dice** | Two slicers combined: Payment Type + Year (e.g. Credit Card, 2023) |
| **Pivot** | Country dimension rotated from columns to rows for cross-market comparison |

---

## 📈 Power BI Reports

Four interactive report pages connected to the data warehouse:

### Report 1 — Payment & Freight Matrix
Matrix visual showing Sum of Payment Value and Sum of Freight Value per Customer Country × Payment Type. Highlights dominant markets: Germany, France, Netherlands, UK.

### Report 2 — Sales Dashboard with Cascading Slicers
- Country → City cascading slicers
- Line chart: Payment Value by Month
- Pie chart: Payment Value by Payment Type
- Bar chart: Freight Value by Customer City
- KPI cards: Total Payment, Average Payment, Order Count

### Report 3 — Hierarchical Drill-Down Chart
Clustered column chart with drill-down through Year → Quarter → Month → Day. Reveals seasonal payment patterns and mid-year performance peaks.

### Report 4 — Drill-Through Navigation
- Overview page: Horizontal bar chart of Payment Value by Country
- Right-click any country → Drill Through → Detail page
- Detail page: Postal Code, Review Score, Year, Order Status, Payment Type per transaction
- Back navigation button included

---

## 🛠️ Technology Stack

| Layer | Technology |
|-------|-----------|
| Source Data | CSV / Text flat files |
| RDBMS | SQL Server 2017+ |
| ETL | SSIS (SQL Server Integration Services) |
| Data Warehouse | SQL Server (Star Schema) |
| OLAP Cube | SSAS (SQL Server Analysis Services) Multidimensional |
| OLAP Client | Microsoft Excel PivotTable |
| BI Reporting | Microsoft Power BI Desktop |
| Development IDE | Visual Studio (SSDT) |

---

## 📁 Repository Structure

```
ecommerce-data-warehouse/
│
├── data/                          # Source flat files
│   ├── orders.csv
│   ├── customers.csv
│   ├── products.csv
│   ├── sellers.txt
│   ├── order_items.csv
│   ├── order_payment.txt
│   └── reviews.csv
│
├── sql/                           # SQL scripts
│   ├── create_oltp_tables.sql
│   ├── create_staging_tables.sql
│   ├── create_warehouse_tables.sql
│   ├── generate_dim_date.sql
│   └── accumulating_fact_update.sql
│
├── ssis/                          # SSIS packages
│   ├── Load_Staging.dtsx
│   ├── Load_DW.dtsx
│   └── Update_WH.dtsx
│
├── ssas/                          # SSAS cube project
│   └── ecommerce_Cube/
│
├── powerbi/                       # Power BI reports
│   └── datawarehouse.pbix
│
├── docs/                          # Assignment reports
│   ├── Assignment_01.pdf
│   └── Assignment_02.pdf
│
└── README.md
```

---

## 🚀 Getting Started

### Prerequisites
- SQL Server 2017 or later
- Visual Studio with SSDT (SQL Server Data Tools)
- SQL Server Analysis Services
- Power BI Desktop
- Microsoft Excel (for OLAP PivotTable demos)

### Setup Steps

**1. Create Databases**
```sql
CREATE DATABASE ecommerce_OLTP;
CREATE DATABASE ecommerce_stag;
CREATE DATABASE ecommerce_WH;
```

**2. Import Source Files**
Use SQL Server Import Wizard or the SSIS `Load_Staging.dtsx` package to load flat files into `ecommerce_OLTP`.

**3. Run SSIS Packages (in order)**
```
1. Load_Staging.dtsx   → Populates ecommerce_stag
2. Load_DW.dtsx        → Populates ecommerce_WH (Star Schema)
3. Update_WH.dtsx      → Updates accumulating fact columns
```

**4. Deploy SSAS Cube**
Open `ecommerce_Cube` project in Visual Studio → Build → Deploy to your SSAS instance.

**5. Open Power BI Reports**
Open `datawarehouse.pbix` in Power BI Desktop and update the data source connection to your SQL Server instance.

---

## 📋 Key Assumptions

- All monetary values (Price, Freight_Value, Payment_Value) are in a **single currency** — no conversion required.
- Each order has **at most one review** linked through the fact table.
- DIM_Customer implements **SCD Type 2** to preserve historical customer attribute changes (city, country, postal code).

---

## 📚 References

- [ETL Process in Data Warehouse – GeeksforGeeks](https://www.geeksforgeeks.org/etl-process-in-data-warehouse/)
- [ETL Process – JavaTPoint](https://www.javatpoint.com/etl-process-in-data-warehouse)
- [Fecom Inc. Dataset – Kaggle](https://www.kaggle.com/datasets/cemeraan/fecom-inc-e-com-marketplace-orders-datacrm)
