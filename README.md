# 🔄 GitHub-to-SQL Incremental Load
 
An automated ETL pipeline that performs incremental data loading from a JSON source hosted on GitHub into an Azure SQL Database, leveraging the Staging/Merge pattern for robust data synchronization

## 📋 Project Overview

This project demonstrates a production-ready **ETL pipeline** built in **Azure Data Factory (ADF)** that automates the process of syncing data from a GitHub repository into Azure SQL Database.

### Key Features
- **Incremental Loading**: Only new and modified records are processed
- **Automated Synchronization**: Uses MERGE statements to handle upserts intelligently
- **Data Integrity**: Staging table pattern prevents data leakage and ensures consistency
- **Timestamp-Based Logic**: Leverages modification timestamps for accurate change detection

The pipeline utilizes a **Staging (Temp) Table pattern** combined with **SQL MERGE statements** to efficiently handle "upserts"—ensuring new records are inserted and existing records are updated based on their modification timestamp.

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|-----------|
| **Source** | GitHub Repository (Raw `Order.json`) |
| **Orchestration** | Azure Data Factory (ADF) |
| **Destination** | Azure SQL Database |
| **Staging Strategy** | Truncate-and-Load to `temp_orders` |
| **Transformation Logic** | SQL Stored Procedure (`MERGE` statement) |

---

## 🔀 Pipeline Workflow

The pipeline consists of two primary activities working in seamless coordination:

### 1️⃣ Copy Data Activity (Stage)

**Purpose**: Extract raw data from GitHub and load it into a staging table

```
GitHub JSON → Staging Table (temp_orders)
```

**Process Details:**

- **Pre-copy Script**: Before data transfer begins, ADF executes a cleanup operation:
  ```sql
  TRUNCATE TABLE temp_orders
  ```
  This ensures the staging table is clean, preventing "data leakage" from previous pipeline runs.

- **Source**: HTTP Connector pointing to the GitHub raw URL containing `Order.json`
- **Sink**: Azure SQL Database (`temp_orders` table)

### 2️⃣ Stored Procedure Activity (Upsert)

**Purpose**: Merge staged data into the production table with intelligent insert/update logic

```
Staging Table (temp_orders) → Production Table (dbo.Orders)
```

Once the staging table is populated, the stored procedure `sp_upsert_orders` is triggered to move data to the final production table: `dbo.Orders`.

#### The MERGE Logic

The procedure uses a **MERGE statement** to compare staging data with production data:

**Matching Criteria:**
- Uses `OrderID` as the unique identifier to match records

**Update Logic:**
- If `OrderID` exists in the production table **AND** the `Source.UpdatedDate` is newer than `Target.UpdatedDate`, the existing record is updated with the latest values

**Insert Logic:**
- If `OrderID` does not exist in the production table, a new record is inserted with all available source data

---

## 💾 SQL Implementation

### Stored Procedure: `sp_upsert_orders`

```sql
-- Core logic of sp_upsert_orders
MERGE dbo.Orders AS Target
USING dbo.temp_Orders AS Source
ON Target.OrderID = Source.OrderID
WHEN MATCHED AND Source.UpdatedDate > Target.UpdatedDate THEN
    UPDATE SET Target.Amount = Source.Amount, ...
WHEN NOT MATCHED BY TARGET THEN
    INSERT (OrderID, CustomerID, ...) VALUES (Source.OrderID, ...);
```

**Key Highlights:**
- **MERGE Statement**: Performs conditional insert/update in a single operation
- **Timestamp Comparison**: Ensures only newer data overwrites existing records
- **Atomic Operation**: All changes are transactional, maintaining data integrity

---

## 🚀 How It Works: Step-by-Step

1. **Pipeline Triggers**: ADF pipeline is scheduled or manually executed
2. **Clean Staging**: Pre-copy script truncates `temp_orders` table
3. **Extract Data**: HTTP connector downloads JSON from GitHub
4. **Load to Staging**: Data is inserted into `temp_orders`
5. **Transform & Merge**: Stored procedure compares and synchronizes data
6. **Update Production**: New records inserted, existing records updated with latest values
7. **Completion**: Pipeline completes with success status

---

## 📊 Data Flow Diagram

```
┌─────────────────────────────────────────────────────────┐
│         GitHub Repository (Order.json)                   │
└──────────────────┬──────────────────────────────────────┘
                   │
                   │ (HTTP Connector)
                   ▼
┌─────────────────────────────────────────────────────────┐
│  Azure Data Factory (ADF)                               │
│  ┌──────────────────────────────────────────────────┐   │
│  │ 1. Copy Activity (Stage)                         │   │
│  │    - Pre-copy: TRUNCATE temp_orders             │   │
│  │    - Sink: temp_orders table                     │   │
│  └──────────────────────────────────────────────────┘   │
│                      ▼                                   │
│  ┌──────────────────────────────────────────────────┐   │
│  │ 2. Stored Procedure Activity (Upsert)            │   │
│  │    - Execute: sp_upsert_orders                   │   │
│  │    - MERGE logic applied                         │   │
│  └──────────────────────────────────────────────────┘   │
└──────────────────┬──────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────┐
│      Azure SQL Database (dbo.Orders)                    │
│      ✓ New records inserted                             │
│      ✓ Existing records updated (if newer)              │
└─────────────────────────────────────────────────────────┘
```
