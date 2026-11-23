# Module 05: Create and organize objects in Unity Catalog

Module link: https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/

Part of: [Set up and Configure an Azure Databricks Environment](../paths/set-up-and-configure-environment.md)

## Overview

- Units: 12 (introduction, 7 content units, exercise, knowledge check, summary)
- Prerequisites: basic Azure Databricks workspace knowledge, Unity Catalog basics, and SQL
- Why it matters: this is where you build the catalog, schema, table, and volume structure that the exam expects you to know

## Learning objectives

1. Apply naming conventions to Unity Catalog objects.
2. Create catalogs and schemas for environments.
3. Create managed and external tables, views, and volumes.
4. Write DDL operations, including functions and stored procedures.
5. Set up foreign catalogs for external databases.
6. Configure AI/BI Genie instructions.

## Key points

### 1. Naming conventions

- Names are limited to 255 characters and are stored in lowercase. Queries are case-insensitive.
- Do not use periods, spaces, slashes, or control characters. Wrap names with special characters in backticks, for example `` `sales-data` ``.
- Use lowercase with underscores, such as `customer_data`. Avoid camelCase.
- Medallion schema names: `bronze`, `silver`, `gold`.
- Prefixes: `vw_` for views and materialized views, `tmp_` for staging tables.
- Environment strategies: a separate catalog per environment (`sales_dev`, `sales_prod`), or schemas per layer inside environment catalogs.
- Use permissions to control who creates objects. Permissions cannot enforce naming by themselves. Check names in CI/CD.

### 2. Catalogs

- A catalog is the top level of the `catalog.schema.table` namespace. It is the main unit of isolation, with its own storage location and access policies.
- Catalogs usually split by environment (prod, dev), by sensitivity, or by domain.
- To create one, you need the metastore admin role or the **CREATE CATALOG** privilege.
- A new catalog automatically includes `default` and `information_schema` schemas.

```sql
CREATE CATALOG IF NOT EXISTS dev_catalog
COMMENT 'Development environment for data engineering experiments';
```

- **Managed location:** strongly recommended. It needs an external location and the **CREATE MANAGED STORAGE** privilege on it. If the metastore has no default storage, you must set a managed location when you create the catalog.

```sql
CREATE CATALOG IF NOT EXISTS prod_catalog
MANAGED LOCATION 'abfss://prod-data@mystorageaccount.dfs.core.windows.net/catalog-root'
COMMENT 'Production catalog with dedicated ADLS Gen2 storage';
```

- New catalogs are visible from all attached workspaces by default. Bind production catalogs to production workspaces only.

### 3. Schemas

- A schema is the middle level of the namespace. Schemas group objects by project, team, or stage.
- `CREATE DATABASE` is an alias for `CREATE SCHEMA`.
- Prerequisites: **USE CATALOG** and **CREATE SCHEMA** on the parent catalog.
- Users need **USE SCHEMA** to see the schema, plus object-level privileges such as **CREATE TABLE**.
- Optional managed location, with the same requirements as catalogs.

```sql
CREATE SCHEMA IF NOT EXISTS prod_catalog.customer_analytics
COMMENT 'Customer behavior and segmentation analysis';
```

### 4. Tables

**Managed tables**
- The default table type. Unity Catalog manages metadata and data files.
- Delta Lake format by default.
- Dropping the table deletes the metadata and the data files.

**External tables**
- Unity Catalog registers the metadata. The data stays in external storage.
- You must give a `LOCATION`, and the location must be covered by a Unity Catalog external location.
- Dropping the table removes only the metadata.
- For new pipelines, prefer managed tables.

```sql
CREATE EXTERNAL TABLE archived_sales
USING CSV
LOCATION 'abfss://container@storage.dfs.core.windows.net/sales/archive'
OPTIONS (header 'true', inferSchema 'true');
```

**Constraints and keys**
- Primary and foreign keys are supported in Databricks Runtime 11.3 LTS and later. They are fully supported in Runtime 15.2 and later.
- They are informational only. They are **not enforced**, but they help the optimizer.
- Primary key columns are implicitly `NOT NULL`.
- Check constraints and generated columns are also available.

```sql
CREATE TABLE orders (
  order_id BIGINT NOT NULL PRIMARY KEY,
  customer_id BIGINT,
  order_date DATE,
  total_amount DECIMAL(10,2),
  CONSTRAINT orders_customers_fk FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

### 5. Views

**Standard views**
- A saved query that runs each time you read it. It stores no data.
- Always shows current data.

**Dynamic views**
- Add row-level filters and column masks based on the user, using `is_account_group_member()`.
- Need SQL warehouses, standard access mode compute, or dedicated compute on Runtime 15.4 LTS or later.

```sql
CREATE VIEW sales_redacted AS
SELECT user_id,
  CASE WHEN is_account_group_member('auditors') THEN email ELSE 'REDACTED' END AS email,
  country_region, product, total
FROM sales_raw;
```

Prefer dynamic views over copying tables for each group.

**Materialized views**
- Store the results as a Delta table, so queries are fast for expensive aggregations.
- Refresh manually or on a schedule, using either **incremental** or **full** refresh.
- Incremental refresh needs Delta source tables with row tracking enabled (Runtime 14.1 or later). Enabling row tracking on an existing table cannot be undone.

```sql
CREATE MATERIALIZED VIEW daily_sales_summary
SCHEDULE EVERY 1 DAY
AS SELECT transaction_date, COUNT(*) AS transaction_count, SUM(amount) AS total_sales
FROM sales_transactions
GROUP BY transaction_date;
```

**Streaming tables**
- Process only new rows since the last refresh. Use them for incremental ingestion, such as Auto Loader files or Kafka events.
- A serverless pipeline that Unity Catalog creates manages them.
- Refresh on a schedule, on upstream updates (`TRIGGER ON UPDATE`), or manually.
- Use streaming tables for append-only sources. Use materialized views for aggregations over full datasets.

### 6. Volumes

- Volumes store files, such as CSV, images, or models, that are not tables.
- **Managed volumes:** Unity Catalog manages storage and lifecycle. Dropping one marks files for deletion after 7 days.
- **External volumes:** you point to an existing path in an external location. Dropping one removes only the registration.
- Create them with `CREATE VOLUME` (managed) or `CREATE EXTERNAL VOLUME ... LOCATION` (external).
- Path format: `/Volumes/<catalog>/<schema>/<volume>/<path>/<file>`. Works with Spark, pandas, SQL, and `dbutils.fs`.
- Privileges: **READ VOLUME**, **WRITE VOLUME**, **CREATE VOLUME**, and **CREATE EXTERNAL VOLUME**.
- Access also needs **USE** on the catalog and schema.
- Volumes do not support folder-level permissions. Create a separate volume for each security boundary.

### 7. DDL operations

**Functions**
- Scalar functions return one value. Table-valued functions return rows.
- Write them in SQL or Python. Python functions run on serverless or pro SQL warehouses.
- **Temporary functions** exist only in the session and are not stored in the catalog.

```sql
CREATE FUNCTION dev_catalog.finance_schema.calculate_discount(price DOUBLE, discount_rate DOUBLE)
RETURNS DOUBLE
COMMENT 'Calculate discounted price'
RETURN price * (1 - discount_rate);
```

**Stored procedures**
- Group several SQL steps into one call. They need Runtime 17.0 or later.
- `SQL SECURITY INVOKER` runs the procedure with the caller's permissions.
- Parameters can be `IN`, `OUT`, or `INOUT`. Call with `CALL`.

**Changing objects**
- `ALTER CATALOG ... SET OWNER TO`, `SET TAGS`, and `ENABLE PREDICTIVE OPTIMIZATION`.
- `ALTER TABLE` can add, rename, drop, and change column types. It needs the **MODIFY** privilege. Renaming columns needs Delta column mapping.

### 8. Foreign catalogs

- A foreign catalog lets you query an external database from Azure Databricks, with Unity Catalog governance. No data is moved.
- A **connection** stores how to reach the system. One connection can back several foreign catalogs.
- Named examples include PostgreSQL, MySQL, Snowflake, BigQuery, and SQL Server / Azure SQL.
- Store credentials with `secret()`, not in plain text.
- Prerequisites: **CREATE CONNECTION** on the metastore, runtime 13.3 LTS or later, and network access to the database.
- Create the catalog with `CREATE FOREIGN CATALOG ... USING CONNECTION ... OPTIONS (database '...')`.
- Permissions: `USE CATALOG`, `USE SCHEMA`, and `SELECT` on each table.
- Metadata refreshes on each query by default. Use `REFRESH FOREIGN CATALOG` (or schema/table) for faster cached access.
- Queries are pushed down to the external database when possible.

### 9. AI/BI Genie instructions

- Genie answers natural-language questions. You teach it with a knowledge store.
- A Genie space can include up to 25 tables or views. Start with five or fewer.
- Write table and column descriptions in business language. Add synonyms. Hide technical columns.
- Joins come from Unity Catalog foreign keys. You can also define joins manually.
- **Prompt matching** maps user phrasing to values (for example, "California" to "CA"). Entity matching is turned on per column.
- Add SQL expressions (measures, filters), example SQL queries, and parameters (`:parameter_name`). Parameterized matches are marked **Trusted**.
- **Limits:** 100 instructions and 200 knowledge store snippets. Entity matching allows up to 1,024 values per column, across 120 columns per space.
- Refine the setup with testing, benchmarks, monitoring, and recommendations.

## Exam focus

- The three-level namespace and which privilege is needed at each level.
- Catalog and schema managed locations, and the order Unity Catalog uses to choose a location.
- Managed table vs. external table: what is deleted on drop.
- Standard, dynamic, materialized, and streaming objects, and when to use each.
- Managed vs. external volumes, and the volume privileges.
- Foreign catalogs need a connection, and access needs `SELECT` on tables.
- Genie limits: 25 tables per space (5 recommended), 100 instructions, 200 snippets.

## Knowledge check

Assessment: https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/11-knowledge-check

The module assessment did not load when I read the page, so I have no questions to record. Add them after you take the assessment.

## Quick review checklist

- [ ] Write `CREATE CATALOG`, `CREATE SCHEMA`, and `CREATE TABLE` statements from memory
- [ ] Know the privileges for each object type
- [ ] Know what dropping a managed vs. external table or volume does
- [ ] Explain when to use dynamic vs. materialized views
- [ ] Explain streaming tables vs. materialized views
- [ ] Set up a foreign catalog conceptually
- [ ] Complete the lab exercise

## Notes on conflicting numbers

- The catalog unit says dropped managed table files are deleted after 8 days. The architecture module and volume unit say 7 days. Check the official docs before the exam.

## References

- Module: https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/
- Knowledge check: https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/11-knowledge-check
- Naming conventions: https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/2-apply-naming-conventions
- Create catalog: https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/3-create-catalog
- Create schema: https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/4-create-schema
- Tables and views: https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/5-create-tables-views-materialized
- Volumes: https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/6-create-volumes
- DDL operations: https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/7-implement-data-definition-language-operations
- Foreign catalogs: https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/8-implement-foreign-catalog
- AI/BI Genie instructions: https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/9-configure-ai-bi-genie-instructions
- Summary: https://learn.microsoft.com/en-us/training/modules/create-and-organize-objects-in-unity-catalog/12-summary

## Back to index

[README](../README.md)
