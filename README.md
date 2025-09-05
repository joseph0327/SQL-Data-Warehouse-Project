# 🏗️ SQL Data Warehouse & Power BI Project

This repository documents the full lifecycle of building a **SQL Server Data Warehouse** with a **Power BI front-end**.  
It covers project planning, data modeling, ETL, and reporting.


![Alt text](https://github.com/joseph0327/SQL-Data-Warehouse-Project/blob/de445d8ade77b97df5d56db12f4672acb5680e82/Data%20Architecture.drawio.png)
![Alt text](https://github.com/joseph0327/SQL-Data-Warehouse-Project/blob/a12a3902c7aed1a9bbd91ccea539f63c669467a2/visual1.png)

---

## 📋 Table of Contents
1. [Planning & Task Management (Notion)](#-step-1-planning--task-management)
2. [Building the Data Warehouse](#-step-2-building-the-data-warehouse)
3. [Separation of Concerns](#-step-3-separation-of-concerns)
4. [Database Creation Scripts](#-step-4-database-creation-scripts)
5. [Source System Analysis](#-step-5-source-system-analysis)
6. [Bronze Layer](#-step-6-bronze-layer)
7. [Silver Layer](#-step-7-silver-layer)
8. [Gold Layer](#-step-8-gold-layer)
9. [Power BI Visualizations](#-step-9-power-bi-visualizations)

---

## 📝 Step 1: Planning & Task Management

We use **Notion** to plan epics and tasks.

- **Create a table:** `/database inline`
- **Relations:** create a relation from the previous database inline
- **Checkboxes:** add checkboxes to tasks
- **Rollups:**  
  - Add relation  
  - Make it a checkbox  
  - Select percent → progress
- **Group tasks by epic:** click on the 3 dots on the task list

This ensures clear tracking of work across Bronze, Silver, and Gold layers.

![Alt text](https://github.com/joseph0327/SQL-Data-Warehouse-Project/blob/1cbac48c0b72816e50059da44de6a6481b3909d0/visual4.png)

---

## 🗄️ Step 2: Building the Data Warehouse

### Medallion Architecture (Used in this Project)
`Sources → Bronze → Silver → Gold → Reporting (Power BI)`

---

### Medallion Architecture Planning

| Layer  | Definition | Objective | Type | Load Method | Transformations | Modeling | Target Audience |
|--------|------------|------------|------|-------------|-----------------|----------|-----------------|
| **Bronze** | Raw, unprocessed data as-is from sources | Traceability & debugging | Tables | Full load (truncate & insert) | None | None | Data Engineers |
| **Silver** | Clean & standardized data | Prepare for analysis | Tables | Full load (truncate & insert) | Cleaning, Standardization, Normalization, Derived Columns, Enrichment | None | Data Analysts & Engineers |
| **Gold** | Business-ready data | Consumed for reporting & analytics | Views | — | Integration, Aggregation, Business Logic | Star Schema, Aggregated Objects, Flat Tables | Analysts & Business Users |

---

## 🔑 Step 3: Separation of Concerns

- Break down modules into smaller pieces.
- Tasks from Bronze should not be applied to Silver or Gold.
- Use **snake_case** for all object names.

Create a GitHub repository for version control.

---

## 🛠️ Step 4: Database Creation Scripts

> ⚠️ **Warning:** Scripts drop the entire data warehouse.

Check if DB exists, drop, and recreate:

```sql
IF EXISTS (SELECT 1 FROM sys.databases WHERE name = 'DataWarehouse')
BEGIN 
 ALTER DATABASE DataWarehouse SET SINGLE_USER WITH ROLLBACK IMMEDIATE;
 DROP DATABASE DataWarehouse;
END;
GO

USE master;
CREATE DATABASE DataWarehouse;
USE DataWarehouse;

CREATE SCHEMA bronze; 
GO
CREATE SCHEMA silver; 
GO
CREATE SCHEMA gold; 
GO
```

---

## 🔎 Step 5: Source System Analysis

**Business Context & Ownership**
1. Who owns the data?
2. What business process does it support?
3. System/data documentation?
4. Data model & data catalog?

**Architecture & Technology Stack**
1. How is data stored? (SQL Server, Oracle, AWS, Azure)
2. Integration capabilities? (API, Kafka, File Extract, Direct DB)

**Extract & Load**
1. Incremental vs full loads?
2. Data scope & historical needs?
3. Expected size of extracts?
4. Data volume limitations?
5. Avoiding impact on source system performance?
6. Authentication & authorization (token, SSH keys, VPN, IP whitelisting)

---

## 🟤 Step 6: Bronze Layer

Create table:

```sql
CREATE TABLE bronze.crm_cust_info (
 cst_id INT,
 cst_key NVARCHAR(50),
 cst_firstname NVARCHAR(50),
 cst_lastname NVARCHAR(50),
 cst_marital_status NVARCHAR(50),
 cst_gender NVARCHAR(50),
 cst_create_date DATE
);
```

Bulk insert:

```sql
BULK INSERT bronze.crm_cust_info
FROM 'C:\...\datasets\source_crm\cust_info.csv'
WITH(
 FIRSTROW=2, 
 FIELDTERMINATOR=',',
 TABLOCK
);
```

Load all Bronze tables via stored procedure:

```sql
CREATE OR ALTER PROCEDURE bronze.load_bronze AS
BEGIN
 TRUNCATE TABLE bronze.crm_sales_details;
 BULK INSERT bronze.crm_sales_details
 FROM 'C:\...\datasets\source_crm\sales_details.csv'
 WITH(FIRSTROW=2, FIELDTERMINATOR=',', TABLOCK);

 TRUNCATE TABLE bronze.crm_prd_info;
 BULK INSERT bronze.crm_prd_info
 FROM 'C:\...\datasets\source_crm\prd_info.csv'
 WITH(FIRSTROW=2, FIELDTERMINATOR=',', TABLOCK);
END;

-- Execute:
EXEC bronze.load_bronze;
```

Model table relationships in **draw.io**.

---

## ⚪ Step 7: Silver Layer

Transform Bronze data into Silver tables.

- Same DDL creation as Bronze but in `silver` schema.
- Add metadata columns:

```sql
dwh_create_date DATETIME2 DEFAULT GETDATE()
```

- Check for duplicates:

```sql
SELECT cst_id, COUNT(*) 
FROM silver.crm_cust_info
GROUP BY cst_id
HAVING COUNT(*) > 1 OR cst_id IS NULL;
```

- Rank duplicates and keep only the latest:

```sql
SELECT *
FROM (
 SELECT *, ROW_NUMBER() OVER(PARTITION BY cst_id ORDER BY cst_create_date DESC) as flag_last
 FROM silver.crm_cust_info
) f
WHERE flag_last = 1 AND cst_id = 29446;
```

- Check unwanted spaces:

```sql
SELECT cst_firstname
FROM silver.crm_cust_info
WHERE cst_firstname != TRIM(cst_firstname);
```

- Check gender values:

```sql
SELECT DISTINCT cst_gender FROM silver.crm_cust_info;
```

- Check product keys not in sales details:

```sql
SELECT
 prd_id,
 REPLACE(SUBSTRING(prd_key,1,5),'-','_') as cat_id,
 SUBSTRING(prd_key,7,LEN(prd_key)) as prd_key,
 prd_nm, prd_cost, prd_line, prd_start_dt, prd_end_dt
FROM bronze.crm_prd_info
WHERE SUBSTRING(prd_key,7,LEN(prd_key)) NOT IN 
(SELECT DISTINCT sls_prd_key FROM bronze.crm_sales_details);
```

Create a stored procedure to load Silver:

```sql
EXEC silver.load_silver;
```

---

## 🟡 Step 8: Gold Layer

Business-ready data for reporting.

- Prefer **LEFT JOIN** over inner join to avoid NULL losses.
- Example joining 3 tables and transforming:

```sql
SELECT 
 ci.cst_id AS customer_id,
 ci.cst_key AS customer_number,
 ci.cst_firstname AS first_name,
 ci.cst_lastname AS last_name,
 la.cntry AS country,
 ci.cst_marital_status AS marital_status,
 CASE 
  WHEN ci.cst_gndr != 'n/a' THEN ci.cst_gndr
  ELSE COALESCE(ca.gen, 'n/a')
 END AS gender,
 ca.bdate AS birthdate,
 ci.cst_create_date AS create_date
FROM silver.crm_cust_info ci
LEFT JOIN silver.erp_cust_az12 ca ON ci.cst_key = ca.cid
LEFT JOIN silver.erp_loc_a101 la ON ci.cst_key = la.cid;
```

- Create surrogate keys:

```sql
ROW_NUMBER() OVER (ORDER BY cst_id) as customer_key
```

---

## 📊 Step 9: Power BI Visualizations

- Connect to SQL Server tables.
- Build a **Star Schema** in Power BI.
- Add DAX measures.
- Create visuals and dashboards.

![Alt text](https://github.com/joseph0327/SQL-Data-Warehouse-Project/blob/a12a3902c7aed1a9bbd91ccea539f63c669467a2/visual1.png)
![Alt text](https://github.com/joseph0327/SQL-Data-Warehouse-Project/blob/a12a3902c7aed1a9bbd91ccea539f63c669467a2/visual2.png)
![Alt text](https://github.com/joseph0327/SQL-Data-Warehouse-Project/blob/de445d8ade77b97df5d56db12f4672acb5680e82/visual3.png)
---

## 📝 Conventions

- Use **snake_case** for tables, columns, and stored procedures.
- Keep each layer’s transformations isolated (Bronze ≠ Silver ≠ Gold).
- Use GitHub for version control.

---

## ⚠️ Disclaimer

Scripts may **drop and recreate** the entire data warehouse.  
Check and backup your data before executing.
