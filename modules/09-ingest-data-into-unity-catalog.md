# Module 09: Ingest data into Unity Catalog

Module link: https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/

Part of: [Prepare and Process Data with Azure Databricks](../paths/prepare-and-process-data.md)

## Overview

- Units: introduction, eight content units (Lakeflow Connect, notebooks, SQL methods, CDC feed, Structured Streaming, Auto Loader, Lakeflow Spark Declarative Pipelines), exercise, knowledge check, and summary
- Prerequisites: Unity Catalog basics, Delta Lake basics, and some Python or SQL
- Why it matters: the exam expects you to pick the right ingestion method for a source. The module compares managed connectors, code, SQL loads, CDC, streaming, and Auto Loader.

## Learning objectives

1. Configure managed connectors with Lakeflow Connect.
2. Write notebook code to read and write batch and streaming data.
3. Use SQL methods for file-based batch loads.
4. Process CDC feeds with the AUTO CDC API.
5. Ingest from message buses such as Kafka and Event Hubs with Structured Streaming.
6. Configure Auto Loader for files that arrive in cloud storage.
7. Orchestrate ingestion with Lakeflow Spark Declarative Pipelines.

## Key points

### 1. Choose an ingestion method

The module's summary gives a simple rule: use managed connectors for common enterprise sources, notebooks for complex logic or custom sources, Auto Loader for file-based streaming with automatic schema handling, and Lakeflow pipelines to orchestrate the flow.

| Method | Use it when |
|---|---|
| Lakeflow Connect | A supported SaaS or database source needs managed extraction, with no custom code |
| Notebook (DataFrame API) | You need custom logic, or a source has no connector |
| SQL: CTAS, CREATE OR REPLACE TABLE, COPY INTO | File-based batch loads, one-time loads, or periodic full refreshes |
| CDC with AUTO CDC | Inserts, updates, and deletes must be applied to a target table |
| Structured Streaming | Events arrive from Kafka or Event Hubs and need near real-time processing |
| Auto Loader | New files land in cloud storage and must be picked up exactly once |
| Lakeflow Spark Declarative Pipelines | You want managed orchestration, retries, and checkpoints around the flows |

### 2. Lakeflow Connect

- Three connector types: **SaaS** (for example Salesforce, HubSpot, Jira, Workday), **database CDC** (for example MySQL, PostgreSQL, SQL Server), and **query-based** (a scheduled query with a cursor column, no CDC or gateway).
- A pipeline has three parts: a **connection** (credentials and endpoint, reusable), an **ingestion definition** (which tables go where), and a **pipeline schedule**.
- Database connectors create an **ingestion gateway** that reads the source change log and stages changes. SaaS connectors do not use a gateway.
- The destination for each table is a streaming table in Unity Catalog. Lakeflow Connect creates a job for each schedule, so you can add downstream tasks after ingestion.
- **SCD type 1** overwrites rows with the current source state. **SCD type 2** keeps history and adds `__START_AT` and `__END_AT` columns.
- `include_columns` limits what is loaded. With it, new source columns are not added automatically. With `exclude_columns`, new source columns are included by default.
- A **full refresh** reloads from a new snapshot. Existing data stays in place until the new snapshot completes.
- One pipeline can write one source table to several destinations. `destination_table` changes the table name within the same schema.

### 3. Notebooks: batch and streaming with DataFrames

- Batch read: `spark.read.format(...).option(...).load(...)`. Common formats are CSV, JSON, Parquet, and XML. XML needs `rowTag`.
- JDBC reads use `format("jdbc")` with `url` and `dbtable`. Store user and password in secrets with `dbutils.secrets.get`, not in code. To push a filter to the database, pass a subquery in `dbtable`.
- Write to a table with `df.write.mode(...).saveAsTable("catalog.schema.table")`. Write modes are `overwrite`, `append`, `ignore`, and `error` (the default).
- Azure Databricks writes Delta by default.
- Streaming read: `spark.readStream` instead of `spark.read`. For Kafka, `startingOffsets` is `latest` or `earliest`. After a checkpoint exists, restarts resume from the checkpoint and ignore `startingOffsets`.
- A plain `readStream` file source needs an explicit schema by default, because Spark does not infer one for streams. Auto Loader is different: it infers a schema when `cloudFiles.schemaLocation` is set (see section 7).
- Streaming write output modes: `append` (default for most cases), `update`, and `complete` (for aggregations).
- The VARIANT type stores semi-structured data. `variant_get` reads a field by path.

### 4. SQL methods

- **CTAS** (`CREATE TABLE ... AS SELECT`): creates a new table from a query. It fails if the table exists. `IF NOT EXISTS` skips the command instead of updating the table.
- **CREATE OR REPLACE TABLE ... AS SELECT**: replaces the contents for a full refresh. It keeps table history, granted privileges, and row filters or column masks, so downstream users need no reconfiguration.
- **COPY INTO**: loads files from a location into an existing Delta table. It is idempotent: files already loaded are skipped on later runs. The target table must exist first.
- COPY INTO file selection: `PATTERN` for a name pattern, `FILES` for named files.
- `FORMAT_OPTIONS` sets parser options. `COPY_OPTIONS ('mergeSchema' = 'true')` lets the table schema grow.
- `VALIDATE ALL` checks files without loading them. It tests parsing, schema match, nullability, and check constraints.
- `read_files` is a table-valued function for reading files directly in SQL.
- Module guidance: COPY INTO fits files in the order of thousands over time. For millions of files, use Auto Loader, or streaming tables backed by Auto Loader.

### 5. CDC feed and the AUTO CDC API

- A CDC record holds the data columns, an operation type (INSERT, UPDATE, or DELETE), and a sequence column.
- AUTO CDC handles out-of-order records by comparing sequence values. An older late update is ignored.
- Python: `dp.create_streaming_table(...)` for the target, then `dp.create_auto_cdc_flow(...)` with `keys`, `sequence_by`, `apply_as_deletes`, `except_column_list`, and `stored_as_scd_type`. Python examples need `from pyspark import pipelines as dp` at the top. The module samples leave this import out.
- SQL: `CREATE OR REFRESH STREAMING TABLE` for the target, then `CREATE FLOW ... AS AUTO CDC INTO ... KEYS (...) APPLY AS DELETE WHEN ... SEQUENCE BY ... STORED AS SCD TYPE 1 | 2`.
- `ignore_null_updates` (Python) or `IGNORE NULL UPDATES` (SQL) keeps existing values when a partial update sends nulls for unchanged columns.
- `TRACK HISTORY ON ... EXCEPT (...)` (SQL, SCD type 2) avoids new history versions for columns that change often but are not meaningful, such as `last_login`.

### 6. Structured Streaming: Kafka and Event Hubs

- Structured Streaming treats the stream as an unbounded table. It uses the same DataFrame API as batch, with `readStream` and `writeStream`.
- Kafka rows have `key`, `value`, `topic`, `partition`, `offset`, and `timestamp`. `key` and `value` are binary, so cast `value` to string and parse it (for example with `from_json`).
- Event Hubs works two ways: through its Kafka-compatible endpoint (port 9093, SASL_SSL with PLAIN, connection string from a secret), or with the native `eventhubs` format. The native connector puts the payload in `body`.
- `startingPosition` for the native connector takes a JSON offset, sequence number, or enqueued time. `-1` is noted as meaning the latest events (verify this value on the module page before the exam).
- **Checkpoints** store progress. Each streaming query needs its own unique checkpoint location. Sharing one causes processing errors.
- Triggers: default (next micro-batch as soon as the previous one finishes), `processingTime="10 seconds"`, `availableNow=True` (process all available data, then stop), and `once=True` (deprecated).
- Use `availableNow=True` for scheduled incremental runs with Databricks Jobs.
- Monitor with the Structured Streaming tab in the Spark UI, and with `query.status`, `query.recentProgress`, and `query.isActive`.

### 7. Auto Loader

- Auto Loader is a streaming source (`format("cloudFiles")`) that picks up new files from cloud storage. Each file is ingested once. Progress is kept in the checkpoint.
- Required options: `cloudFiles.format` (JSON, CSV, Parquet, Avro, ORC, XML, text, or binary) and `cloudFiles.schemaLocation` (where the inferred schema is stored). Keep it separate from the checkpoint if you want to reset a stream without losing the schema.
- **Detection modes:** directory listing (needs only storage access) and file notification (uses cloud notifications, better at scale). For production, the module recommends file notification mode with file events enabled on the Unity Catalog external location.
- For JSON and CSV, columns are inferred as strings unless `cloudFiles.inferColumnTypes` is `true`.
- **Schema evolution modes** (`cloudFiles.schemaEvolutionMode`): `addNewColumns` (the default when no schema is given; stream fails once, then continues), `addNewColumnsWithTypeWidening` (Runtime 16.4+), `rescue` (schema fixed, new or mismatched data goes to `_rescued_data`), `failOnNewColumns` (stream fails until the schema is updated), and `none` (default when a schema is given; new columns are ignored).
- `_rescued_data` holds data that does not match the schema. `rescuedDataColumn` renames it.
- `cloudFiles.schemaHints` overrides the inferred type of known columns.
- Batch size limits: `cloudFiles.maxFilesPerTrigger` and `cloudFiles.maxBytesPerTrigger`.
- `cloud_files_state(...)` shows the discovered-file state. Do not let lifecycle policies delete checkpoint files, because that corrupts the stream.
- The `_metadata` column exposes `file_path`, `file_name`, `file_size`, and `file_modification_time`.
- In declarative pipelines, `STREAM read_files(...)` in SQL, or `cloudFiles` in Python, gives Auto Loader behavior, and the pipeline manages checkpoints and schema.

### 8. Lakeflow Spark Declarative Pipelines

- A framework for batch and streaming pipelines in SQL or Python. You declare the transformations. The pipeline handles orchestration, retries, and incremental processing.
- **Streaming tables** are the main ingestion targets. Each source row is processed exactly once.
- **Flows** move data from a source to a target. The default flow comes from the table's query. Explicit flows let several sources feed one table.
- Multiple sources into one table: SQL `CREATE FLOW ... AS INSERT INTO ... BY NAME`, or Python `@dp.append_flow(target=...)`.
- **Flow names identify checkpoints.** Renaming a flow discards its checkpoint, so the flow reprocesses from the start.
- Python uses `@dp.table` for a table function. Message bus sources include Kafka, Event Hubs, Kinesis, and Pub/Sub. In SQL, `read_kafka` is used inside a `STREAM` source.
- Secrets belong in `dbutils.secrets.get`, not in code.
- Schema evolution modes for pipelines are similar to Auto Loader (`addNewColumns` default, `rescue`, `failOnNewColumns`, `none`). Use schema hints for strict control.
- Keep pipeline source code in a Git folder. The default structure uses `transformations` for code and `explorations` for ad hoc work.
- Monitor with the pipeline event log and the Lakeflow Pipelines Editor, and set notifications in pipeline settings.

## Quick review checklist

- [ ] I can name the three Lakeflow Connect connector types and say which one needs a gateway.
- [ ] I know the difference between SCD type 1 and type 2 in Lakeflow Connect and AUTO CDC.
- [ ] I can explain COPY INTO idempotency and when to use Auto Loader instead.
- [ ] I know what CREATE OR REPLACE TABLE keeps (privileges, row filters, column masks).
- [ ] I can list the AUTO CDC clauses: KEYS, SEQUENCE BY, APPLY AS DELETE WHEN, STORED AS SCD TYPE.
- [ ] I know that each streaming query needs its own checkpoint location.
- [ ] I can name the Auto Loader detection modes and the schema evolution modes.
- [ ] I know that renaming a pipeline flow resets its checkpoint.
- [ ] I know that secrets go in dbutils.secrets.get, never in code.

## Knowledge check and exercise

Assessment: https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/10-knowledge-check

The lab exercise and knowledge check were not captured in these notes. The module page loaded only the content units. Review the exercise and questions on the module page before the exam.

## References

- Module: https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/
- Knowledge check: https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/10-knowledge-check
- Unit 1, introduction: https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/1-introduction
- Unit 2, Lakeflow Connect: https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/2-ingest-data-lakeflow-connect
- Unit 3, notebooks: https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/3-ingest-data-notebooks
- Unit 4, SQL methods: https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/4-ingest-data-sql-methods
- Unit 5, CDC feed: https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/5-ingest-data-change-data-capture-feed
- Unit 6, Structured Streaming: https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/6-ingest-data-spark-structured-streaming
- Unit 7, Auto Loader: https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/7-ingest-data-auto-loader
- Unit 8, Lakeflow Spark Declarative Pipelines: https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/8-ingest-data-lakeflow-declarative-pipelines
- Unit 11, summary: https://learn.microsoft.com/en-us/training/modules/ingest-data-into-unity-catalog/11-summary

Source note: several unit pages returned condensed text rather than verbatim content, so some wording above is a summary. Check exact syntax against the module pages before relying on it.

## Back to index

[README](../README.md)
