# Module 12: Design and implement data pipelines with Azure Databricks

Module link: https://learn.microsoft.com/en-us/training/modules/design-implement-data-pipelines/

Part of: [Deploy and Maintain Data Pipelines and Workloads with Azure Databricks](../paths/deploy-maintain-data-pipelines-workloads.md)

## Overview

- Units: introduction, design order of operations, notebooks vs Lakeflow pipelines, Lakeflow Jobs task logic, error handling, notebook pipeline, declarative pipeline, exercise, knowledge check, and summary
- Level: intermediate, for data engineers
- Prerequisites: Azure Databricks workspaces, data engineering concepts, SQL, and Python notebooks
- Why it matters: the exam asks you to pick a pipeline approach, wire task dependencies, and choose the right error handling for each failure type

## Learning objectives

1. Design the order of operations for a pipeline, from ingestion to serving.
2. Choose between notebooks and Lakeflow pipelines.
3. Design task logic, dependencies, and execution patterns for Lakeflow Jobs.
4. Implement error handling, including retry policies and expectations.
5. Build pipelines with notebooks and with Lakeflow pipelines.

## Key points

### 1. Order of operations

- The usual flow follows the medallion architecture. Ingestion is the bronze layer. Cleaning and validation is the silver layer. Loading to target tables is usually the gold layer. Serving gives consumers views, performance settings, and access control.
- Ingestion keeps data close to its raw form, so you can reprocess it later.
- Cleaning covers schema enforcement, null handling, deduplication, and data quality checks.
- Transformation covers joins, aggregations, denormalization, and business logic.
- Loading chooses full refresh, incremental append, or merge, and plans for schema evolution.
- Monitoring tracks execution times, record counts, data quality trends, and cost.

### 2. Notebooks or Lakeflow pipelines

- **Notebooks** run code step by step. You control reading, transforming, writing, execution flow, error handling, and tuning.
- **Lakeflow pipelines** (Spark Declarative Pipelines) let you declare the tables you want. The engine handles orchestration, parallelism, and recovery. The basic unit is a **flow**. Defining a streaming table or materialized view creates its flow.
- Choose notebooks for complex logic, prototyping, external libraries, and fine-grained tuning.
- Choose Lakeflow pipelines for standard ETL, built-in data quality, automatic dependency handling, lineage, and external streaming targets such as Kafka or Event Hubs.
- The comparison table in the unit: control level, code complexity, error handling, data quality, lineage, team skills, and best use.
- A common approach: notebooks for custom preprocessing or ML training, and Lakeflow pipelines for the core ETL.

### 3. Lakeflow Jobs task logic

- Tasks and their dependencies form a **DAG** (directed acyclic graph). Dependencies set execution order and control how failures spread.
- **Run-if conditions**: All succeeded (default style), At least one succeeded, None failed (skipped tasks allowed), All done, At least one failed, All failed. Use All done for cleanup.
- Serial runs tasks one after another. Parallel runs tasks with no dependency between them at the same time.
- **If/else condition tasks** route on a value, for example `{{tasks.quality_check.values.invalid_count}} > 100`.
- **Run if** dependencies route on upstream outcomes, not on data.
- **For each** runs a nested task once per array element. The array can come from JSON, a task value, or a job parameter. Set concurrency to control how many iterations run at once. Example: `dbutils.jobs.taskValues.set(key=..., value=...)` in an upstream task, then `{{tasks.<name>.values.<key>}}` in the For each task.
- **Job parameters** are defined at job level and referenced with `{{job.parameters.<name>}}`. Dynamic values include `{{job.trigger.time.iso_date}}` and `{{backfill.iso_datetime}}`.
- Compute guidance: serverless jobs compute for notebooks and Python scripts, serverless SQL warehouses for SQL tasks, serverless pipeline compute for Lakeflow pipelines, and classic jobs compute for JAR and Spark Submit tasks.
- Disable a task to skip it during debugging. Its configuration and run history stay.

### 4. Error handling

- The platform handles infrastructure failures, such as a node that fails. You handle data errors and application logic failures.
- **Expectations** (Lakeflow pipelines): Warn, Drop (`ON VIOLATION DROP ROW`), and Fail (`ON VIOLATION FAIL UPDATE`). A fail stops only that flow. Other flows keep running. Manual action is needed before reprocessing.
- **Retry policies** (jobs): retry count (typically 1 to 3) and retry interval. For continuous jobs, exponential backoff increases the wait after each failure.
- **Timeouts**: expected duration sends a warning when exceeded, and maximum duration terminates the task.
- **Streaming backlog metric thresholds** (Public Preview) notify when a streaming source falls behind.
- **Notifications** can go to email, Slack, Microsoft Teams, PagerDuty, or webhooks. Task notifications fire on every retry, so select **Mute notifications until the last retry** to reduce noise.
- Notebook-level handling: catch `PySparkException`, check `getErrorClass()` (for example `TABLE_OR_VIEW_NOT_FOUND`), and re-raise anything unexpected.
- Retry in code with exponential backoff. Return a status string with `dbutils.notebook.exit(...)` so downstream tasks can read the result.
- Best practices: fail fast on critical errors, log context without sensitive data, make operations idempotent so only failed tasks rerun, watch trends in dropped records, and test failures on purpose.

### 5. Notebook pipeline in Lakeflow Jobs

- A notebook task runs a notebook as a job task. Each task is a node in the DAG.
- Set dependencies with **Depends on** in the task settings. Tasks that depend on the same upstream task run in parallel. A task with several upstream tasks waits for all of them.
- Create the task in **Jobs & Pipelines**: Create, Job, pick a name, Type Notebook, Source Workspace, path, compute, then Create task.
- Pass values with job parameters (**Parameters** section, key and value). Read them in the notebook with `dbutils.widgets.get("name")`. `dbutils.widgets.text` sets a default.
- Best practices: descriptive task names such as `bronze_ingest_orders`, folders that mirror the pipeline, dependencies only where data really flows, and workspace notebooks (not remote Git notebooks) for production jobs that use MLflow.
- Notebook output is capped at 20 MB total and 8 MB per cell. Write large results to a table.

### 6. Declarative pipeline

- You define the end state, and the framework handles execution.
- Benefits: automatic ordering from table dependencies, incremental processing (streaming tables process each record once; materialized views update changed data where possible), and retries at three levels: Spark task, then flow, then pipeline.
- **Streaming table** (append-only, for ingestion): SQL `CREATE OR REFRESH STREAMING TABLE name AS SELECT * FROM STREAM read_files(...)`. Python uses `@dp.table` with `spark.readStream.format("cloudFiles")`.
- **Materialized view** (aggregations, joins, data with updates or deletes): SQL `CREATE OR REPLACE MATERIALIZED VIEW ... AS SELECT ...`. Python uses `@dp.materialized_view`.
- The module's tip: use streaming tables for append-only sources with low latency, and materialized views for aggregations, complex joins, or sources with updates and deletes.
- Expectations apply here too (see section 4).
- The **Lakeflow Pipelines Editor** gives a default folder structure, dry run for validation without processing data, selective execution, an interactive DAG, data preview, and Genie Code for building pipelines from natural language.
- Pipeline code uses `pyspark.pipelines` and streaming table DDL, which run only in the Lakeflow pipeline runtime. They do not run in a regular notebook. Validate with dry run, then run from Jobs & Pipelines.

## Quick review checklist

- [ ] I can map ingestion, cleaning, transformation, loading, and serving to the medallion layers.
- [ ] I can name when to choose notebooks and when to choose Lakeflow pipelines.
- [ ] I know the six run-if conditions and when to use All done.
- [ ] I can explain serial and parallel task execution.
- [ ] I know how For each, If/else, and job parameters work.
- [ ] I can match each task type to its recommended compute.
- [ ] I know the three expectation actions and that a fail stops only one flow.
- [ ] I know the retry, timeout, and notification settings, and the mute-until-last-retry option.
- [ ] I can write a notebook that catches `PySparkException` and re-raises unexpected errors.
- [ ] I know when to use a streaming table and when to use a materialized view.
- [ ] I know that pipeline code does not run in a regular notebook.

## Knowledge check and exercise

The exercise and knowledge check were not captured in these notes. Review them on the module page before the exam.

## References

- Module: https://learn.microsoft.com/en-us/training/modules/design-implement-data-pipelines/
- Unit 2, order of operations: https://learn.microsoft.com/en-us/training/modules/design-implement-data-pipelines/2-design-order-operations-pipeline
- Unit 3, notebooks vs Lakeflow pipelines: https://learn.microsoft.com/en-us/training/modules/design-implement-data-pipelines/3-choose-notebook-lakeflow-pipelines
- Unit 4, Lakeflow Jobs task logic: https://learn.microsoft.com/en-us/training/modules/design-implement-data-pipelines/4-design-task-logic-lakeflow-job
- Unit 5, error handling: https://learn.microsoft.com/en-us/training/modules/design-implement-data-pipelines/5-design-error-handling-pipelines
- Unit 6, notebook pipeline: https://learn.microsoft.com/en-us/training/modules/design-implement-data-pipelines/6-create-pipeline-notebook-precedence
- Unit 7, declarative pipeline: https://learn.microsoft.com/en-us/training/modules/design-implement-data-pipelines/7-create-pipeline-lakeflow-declarative
- Unit 10, summary: https://learn.microsoft.com/en-us/training/modules/design-implement-data-pipelines/10-summary

Source note: several unit pages returned condensed text rather than verbatim content, so some wording above is a summary. Check exact syntax against the module pages before relying on it.

## Back to index

[README](../README.md)
