Data Pipeline Project: GitHub-to-SQL Incremental Load
📌 Project Overview
This project demonstrates an automated ETL pipeline built in Azure Data Factory (ADF) that performs an incremental load from a JSON source hosted on GitHub into an Azure SQL Database.

The pipeline utilizes a Staging (Temp) Table pattern and a SQL MERGE statement to handle "Upserts"—ensuring new records are inserted and existing records are updated based on their modification timestamp.

🛠️ Tech Stack
Source: GitHub Repository (Raw Order.json)

Orchestration: Azure Data Factory (ADF)

Destination: Azure SQL Database

Staging Strategy: Truncate-and-Load to temp_orders

Transformation Logic: SQL Stored Procedure (MERGE statement)

⚙️ Pipeline Workflow
The pipeline consists of two primary activities:

1. Copy Data Activity (Stage)
The first step moves raw data from the GitHub JSON file into a staging table named temp_orders.

Pre-copy Script: Before the data is moved, ADF executes truncate table temp_orders. This ensures that the staging table is cleaned of any old data from previous runs, preventing "data leakage."

Source: HTTP Connector (pointing to GitHub raw URL).

Sink: Azure SQL Database (temp_orders table).

2. Stored Procedure Activity (Upsert)
Once the staging table is populated, a Stored Procedure named sp_upsert_orders is triggered to move the data into the final production table, dbo.Orders.

The MERGE Logic:
The procedure uses a MERGE statement to compare the staging data with the production data:

Matching Criteria: Uses OrderID to identify matching records.

Update Logic: If a match is found AND the Source.UpdatedDate is newer than the Target.UpdatedDate, the existing record is updated with the latest values.

Insert Logic: If the OrderID does not exist in the target table, a new record is inserted.
