# Module 08: Design and implement data modeling with Azure Databricks

Module link: https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/

Part of: [Prepare and Process Data with Azure Databricks](../paths/prepare-and-process-data.md)

## Overview

- Units: 14 (introduction, 10 content units, exercise, knowledge check, summary)
- Prerequisites: basic Azure Databricks and Unity Catalog knowledge, SQL and data warehouse concepts, and Delta Lake basics
- Why it matters: this module covers the design decisions behind a lakehouse table: how to ingest, which format, how to partition or cluster, how to track history, and which table type to use

## Learning objectives

1. Design ingestion logic and configure source connections.
2. Choose an ingestion tool.
3. Choose between Delta Lake, Apache Iceberg, and other formats.
4. Design partitioning.
5. Choose and implement SCD types, including SCD Type 2.
6. Design temporal (history) tables.
7. Choose the right granularity.
8. Choose managed or external tables.
9. Design clustering.

## Key points

### 1. Ingestion design

**Extraction types**
- **Full:** reads everything each run. Use it for small data, no change tracking, or simple rebuilds.
- **Incremental:** reads only new or changed rows. Needs a reliable change column (timestamp, sequence, or version), state tracking, and logic for late records.
- **Streaming:** always-on, near real-time. Use it when freshness is seconds or minutes.
- **CDC:** captures inserts, updates, and deletes, with the operation type. It needs a sequencing column.

**Change sources:** database transaction logs, change data feeds (such as Delta), or periodic snapshots compared for differences.

**File formats**
- Text (CSV, JSON): readable, widely supported, but slow and large. Use for landing zones.
- Columnar (Parquet, ORC): compressed, column pruning, embedded schema. Prefer it when you control the upstream export.
- Semi-structured (XML, nested JSON): plan how to flatten or keep the nesting.
- Avro with a schema registry suits high-frequency streaming.

**Storage design**
- Use Unity Catalog external locations for cloud storage. Authenticate with managed identities, service principals, or storage credentials.
- Use Hive-style folders such as `year=2025/month=11/day=28/` so partition values are read from folder names.
- Use processing-state folders (incoming, processing, archive, failed).

**Decision table from the unit**

| If requirements say | Choose |
|---|---|
| Small volume, no change tracking | Full extraction |
| Large volume, reliable change column, hourly or daily is fine | Incremental |
| Near real-time, events published | Streaming |
| Must capture deletes or keep an audit trail | CDC |

Worked example in the unit: 2 million Postgres rows, 50,000 new per day, hourly freshness, and no native CDC. The answer was hourly incremental extraction, landing Parquet files in ADLS.

### 2. Ingestion tools

- **Lakeflow Connect managed connectors:**
  - SaaS connectors (for example Salesforce, ServiceNow, Dynamics 365, Workday) run on serverless compute and write to streaming tables.
  - Database connectors (MySQL, PostgreSQL, SQL Server) use CDC from transaction logs. They need an ingestion gateway and a staging volume.
  - Community connectors cover sources with no managed connector.
- **Query-based connectors:** query source tables with a cursor column. No gateway needed. History modes are SCD Type 1, SCD Type 2, and append-only. They put more load on the source than CDC does.
- **Auto Loader:** incremental file ingestion from cloud storage. Tracks processed files in a checkpoint. Uses directory listing or file notification mode.
- **COPY INTO:** SQL batch load. It is idempotent, so reruns skip files already loaded. Good for one-time migrations of large data, or simpler SQL-first batch loads.
- **Spark Structured Streaming:** exactly-once streaming from Kafka, Event Hubs, Delta change feeds, or cloud storage. Use it when you need custom windowing or fine control.
- **JDBC/ODBC notebooks:** for databases without a managed connector. Store credentials in Databricks secrets.
- **Azure Data Factory:** for orchestration and landing data from Azure sources. Do not use it for core transformations.

Start with the most managed option that meets the requirements.

### 3. Table formats

- A **file format** (Parquet, CSV, JSON) stores bytes. A **table format** (Delta Lake, Iceberg) adds a metadata layer for transactions, schema changes, and file tracking.
- **Parquet** is columnar. It supports column pruning and predicate pushdown. That is why it is fast for analytics.
- **Delta Lake** is the default format for Azure Databricks tables. It adds ACID transactions, time travel, schema enforcement and evolution, and unified batch and streaming. **Change data feed** is Delta-only.
- **Apache Iceberg** is for interoperability with engines outside Azure Databricks (for example Snowflake, Athena, Trino, Spark, and Flink). Unity Catalog can manage managed Iceberg tables.
- **CSV and JSON** are for landing zones. Convert them to Delta during ingestion.
- Liquid clustering and predictive optimization work with both Delta and managed Iceberg tables.
- For most cases, Delta is the answer unless you need to share with engines that cannot read Delta.

### 4. Partitioning

- Partition only when the table is large. The unit says not to partition tables under **1 TB**.
- Each partition should hold at least **1 GB**. Over-partitioning creates many small files.
- Use **low-cardinality** columns, such as dates, regions, or status. Avoid high-cardinality columns such as customer or transaction IDs.
- Partition by the columns users filter on most.
- Create a partitioned table with `PARTITIONED BY`:

```sql
CREATE TABLE sales.transactions (
    transaction_id STRING,
    customer_id STRING,
    product_id STRING,
    quantity INT,
    unit_price DECIMAL(10,2),
    total_amount DECIMAL(12,2),
    transaction_date DATE
)
PARTITIONED BY (transaction_date);
```

- A generated column can derive the partition date from a timestamp. Delta then creates partition filters for queries on the timestamp.
- Changing partition columns means rewriting the whole table. Plan before loading large data.
- **Liquid clustering is not compatible with partitioned tables.** Choose one for each table. Liquid clustering is recommended for new tables where partitioning would limit performance.

### 5. Slowly changing dimensions (SCD)

| Type | What happens | Use it for |
|---|---|---|
| Type 0 | Never update | Constants such as birth dates or audit values |
| Type 1 | Overwrite old values | Corrections, contact details, current-state reporting |
| Type 2 | Add a new row per version, with validity dates and a current flag | Historical attribution, audits, trend analysis |
| Type 3 | Keep one previous value in a column | Before/after comparisons, when storage is tight |

- Type 1 changes past aggregates. A sale made in Seattle moves to Portland if the city was overwritten.
- Type 2 needs a surrogate key, valid-from and valid-to dates, and a current flag. Fact tables link to the specific version with the surrogate key.
- Mixed strategies are common. For example: Type 0 for registration date, Type 1 for email, and Type 2 for region.
- The unit covers SCD Types 0 to 3. Types 4 to 6 are not covered in the page content.

**SCD Type 2 table**

```sql
CREATE TABLE sales.customer (
    customer_sk BIGINT GENERATED ALWAYS AS IDENTITY,
    customer_id STRING NOT NULL,
    customer_name STRING,
    email STRING,
    city STRING,
    region STRING,
    valid_from TIMESTAMP NOT NULL,
    valid_to TIMESTAMP,  -- NULL for the current row, so this column cannot be NOT NULL
    is_current BOOLEAN
)
USING DELTA
TBLPROPERTIES (delta.enableChangeDataFeed = true);
```

**Caution about the sample MERGE:** the unit closes the current row and inserts the new version as two separate steps. Those steps are not one transaction, so a failure between them can leave the table with a closed row and no new version. Check the pattern before you use it. Safer options are one MERGE that reads a staged union of the changed rows and the new versions, or `AUTO CDC` in Lakeflow pipelines (the unit's tip).

**Querying history**
- Point in time: filter where `valid_from <= ts` and `valid_to > ts`.
- Full history: filter by the business key and order by `valid_from`.
- Time travel: `TIMESTAMP AS OF` or `VERSION AS OF`.

### 6. Temporal (history) tables

- **SCD Type 2** tracks business changes for reporting. **Temporal tables** track technical changes for auditing, debugging, and reproducing reports.
- **Change data feed (CDF)** is off by default. It only records changes made after you turn it on.

```sql
ALTER TABLE bronze.sales_transactions
SET TBLPROPERTIES (delta.enableChangeDataFeed = true);

SELECT transaction_id, amount, _change_type, _commit_version, _commit_timestamp
FROM table_changes('bronze.sales_transactions', '2026-01-01', '2026-01-31')
WHERE _change_type IN ('insert', 'update_postimage');
```

- CDF columns: `_change_type` (insert, update_preimage, update_postimage, delete), `_commit_version`, and `_commit_timestamp`.
- Time travel: `TIMESTAMP AS OF`, `VERSION AS OF`, and `DESCRIBE HISTORY`.
- For long-term audit, stream the change feed into a separate history table, using `trigger(availableNow=True)` in scheduled runs.
- Default retention: deleted files 7 days (`delta.deletedFileRetentionDuration`), logs 30 days (`delta.logRetentionDuration`). Set both properties for longer history. The unit says they "must be set together", but you can set one alone. Check the current docs.
- The unit's delete-capture example uses `ignoreDeletes`. That option skips delete commits when a Delta table is read as a stream. It does not capture deletes. To capture deletes, use the change data feed, which records `_change_type = 'delete'`. Verify this before you rely on it in an audit pipeline.

### 7. Granularity

- **Granularity** (grain) is the level of detail each row holds.
- **Fact tables:** the grain is the event each row records. Declare it explicitly before adding columns. Example: "one row per sales order line item."
- **Dimension tables:** the grain is how each entity is versioned. Options are one row per entity (Type 1), one row per version (Type 2), or one row per period (snapshot).
- **Atomic grain** keeps the most detail. You can always aggregate up, but never down.
- **Aggregate grain** is pre-summarized for speed. It limits the questions you can answer.
- Keep both when you can. Materialized views can keep aggregates current.
- **Alignment rule:** dimension grain must match fact grain, so a fact can find the exact version of a dimension row it refers to.
- Compliance can require a specific grain. When unsure, start finer.

### 8. Managed vs. external tables

| | Managed | External |
|---|---|---|
| Storage | Unity Catalog managed location | Location you specify (`LOCATION` clause) |
| Data files | Unity Catalog manages | You manage |
| Formats | Delta, Iceberg | Delta, CSV, JSON, Avro, Parquet, ORC, text |
| Predictive optimization | Yes | No |
| DROP TABLE | Recoverable with UNDROP for **7 days** by default, then an asynchronous purge deletes the data files | Files stay in storage |

- Databricks recommends managed tables for most cases.
- Use external tables when other systems must read or write the files directly. Note that direct access bypasses Unity Catalog controls and auditing.
- Use external tables for migration from Hive metastore. Convert them to managed tables afterward.
- Do not register the same external table in multiple metastores.
- Use **Delta Sharing** for sharing data across regions or with partners.

### 9. Clustering

- **Data skipping:** Delta keeps min and max statistics per file. Queries skip files that cannot match. Skipping only works well when related rows sit together.
- **Liquid clustering** (recommended for new tables):
  - Change keys without rewriting data.
  - Up to four clustering keys.
  - `OPTIMIZE` is incremental. Schedule it every one or two hours for busy tables.
  - Use predictive optimization to automate it.

```sql
CREATE TABLE sales.transactions (
    transaction_id STRING,
    customer_id STRING,
    product_category STRING,
    region STRING,
    transaction_date DATE,
    amount DECIMAL(12,2)
)
CLUSTER BY (region, transaction_date);

ALTER TABLE sales.transactions CLUSTER BY (customer_id, transaction_date);
OPTIMIZE sales.transactions FULL;   -- reclusters existing data
CREATE TABLE sales.orders CLUSTER BY AUTO;   -- automatic key selection (needs predictive optimization)
```

- **Z-ordering** is legacy. Changing its keys requires a full rewrite. To move from Z-ordering to liquid clustering, run `ALTER TABLE ... CLUSTER BY` and then `OPTIMIZE ... FULL`.
- **Deletion vectors** mark rows as deleted without rewriting files. They are on by default for new tables on Runtime 14.1 and later. For compliance, run `REORG TABLE ... APPLY (PURGE)` and then `VACUUM`.

## Exam focus

- Extraction types, and when CDC is needed.
- Ingestion tool choice: managed connector, query-based connector, Auto Loader, COPY INTO, streaming, JDBC.
- Delta is the default. Iceberg is for interoperability. CSV and JSON are for landing zones.
- The **1 TB** threshold for partitioning, and the **1 GB** minimum per partition.
- Liquid clustering and partitioning cannot be combined.
- SCD Types 0 to 3, and what Type 2 needs (surrogate key, validity dates, current flag).
- CDF is off by default and Delta-only.
- Managed table data files are purged after a **7-day** recovery window by default. The 8-day figure in this unit was wrong (checked on the Learn managed tables page).
- Liquid clustering syntax: `CLUSTER BY`, `OPTIMIZE`, `OPTIMIZE ... FULL`, and `CLUSTER BY AUTO`.

## Knowledge check

Assessment: https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/13-knowledge-check

The module assessment did not load when I read the page, so I have no questions to record. Add them after you take the assessment.

## Quick review checklist

- [ ] Pick an extraction type and ingestion tool from a scenario
- [ ] Explain Delta vs. Iceberg vs. CSV/JSON
- [ ] Apply the 1 TB and 1 GB partitioning rules
- [ ] Compare SCD Types 0 to 3
- [ ] Explain the difference between SCD Type 2 and temporal tables
- [ ] Explain managed vs. external tables
- [ ] Write liquid clustering statements
- [ ] Complete the lab exercise

## Notes on conflicting or unclear content

- The managed table drop window is 7 days by default. This module's 8-day figure was wrong. Earlier modules are correct.
- The SCD Type 2 MERGE sample may not work as written. See section 5.
- The unit's retention advice ("must be set together") and its delete-capture example should be checked against the docs.

## References

- Module: https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/
- Knowledge check: https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/13-knowledge-check
- Ingestion design: https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/2-design-data-ingestion-logic
- Ingestion tools: https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/3-choose-data-ingestion-tool
- Table formats: https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/4-choose-data-table-format
- Partitioning: https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/5-design-data-partitioning-scheme
- SCD types: https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/6-choose-slowly-changing-dimension-type
- SCD Type 2 implementation: https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/7-implement-slowly-changing-dimension-type-2
- Temporal tables: https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/8-design-temporal-history-table
- Granularity: https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/9-choose-granularity
- Managed vs. external tables: https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/10-choose-managed-vs-unmanaged-tables
- Clustering: https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/11-design-clustering-strategy
- Summary: https://learn.microsoft.com/en-us/training/modules/design-implement-data-modeling-unity-catalog/14-summary

## Back to index

[README](../README.md)
