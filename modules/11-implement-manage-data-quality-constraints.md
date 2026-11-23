# Module 11: Implement and manage data quality constraints with Azure Databricks

Module link: https://learn.microsoft.com/en-us/training/modules/implement-manage-data-quality-constraints-unity-catalog/

Part of: [Prepare and Process Data with Azure Databricks](../paths/prepare-and-process-data.md)

## Overview

- Units: introduction, validation checks, data type checks, schema drift, pipeline expectations, exercise, knowledge check, and summary
- Prerequisites: Lakeflow Spark Declarative Pipelines basics, Delta Lake basics, and SQL or Python
- Why it matters: the exam tests which mechanism enforces a rule (table constraint, Delta schema enforcement, Auto Loader evolution mode, or pipeline expectation) and what each action does on a violation

## Learning objectives

1. Implement nullability, cardinality, and range checks.
2. Implement data type checks with schema enforcement, casts, and CHECK constraints.
3. Detect and manage schema drift.
4. Manage data quality with pipeline expectations, including actions and monitoring.

## Key points

### 1. Two validation mechanisms

- **Pipeline expectations** run inside Lakeflow Spark Declarative Pipelines during transformations. They can warn, drop invalid records, or fail. They suit streaming tables and materialized views.
- **Table constraints** are Delta constraints checked at write time. They reject invalid data before it is stored. They suit batch loads and strict integrity rules.
- The module groups checks into three dimensions: nullability, cardinality (uniqueness), and range.

### 2. Nullability checks

- Pipeline expectation: `@dp.expect_or_drop("name", "col IS NOT NULL")` in Python, or `CONSTRAINT name EXPECT (col IS NOT NULL) ON VIOLATION DROP ROW` in SQL.
- Delta table: define the column as `NOT NULL`, or run `ALTER TABLE ... ALTER COLUMN ... SET NOT NULL`.
- Adding `NOT NULL` to an existing table fails if any existing row has a null.

### 3. Cardinality (uniqueness) checks

- A pipeline expectation can count duplicates with a window function, for example `count("*").over(Window.partitionBy("ssn"))` with `@dp.expect("name", "ssn_count = 1")`.
- A materialized view can list duplicate keys with `GROUP BY ... HAVING COUNT(*) > 1` for investigation.
- A Delta `PRIMARY KEY` constraint is documented but **not enforced**. It helps optimization and documents the model. Use expectations to enforce uniqueness.

### 4. Range checks

- Pipeline expectations: `@dp.expect_or_fail("name", "age BETWEEN 0 AND 150")` in Python, or `CONSTRAINT ... EXPECT (...) ON VIOLATION DROP ROW` in SQL.
- Delta `CHECK` constraints are enforced on insert and update: `ALTER TABLE t ADD CONSTRAINT name CHECK (...)`.
- Business rules can combine conditions in one expectation. The module's example limits a discount to 0 to 100 percent and allows more than 50 percent only for premium customers.

### 5. Validation actions

| Action | Python | SQL | Result |
|---|---|---|---|
| Warn (default) | `@dp.expect` | `EXPECT (...)` | Invalid records are written; metrics are recorded |
| Drop | `@dp.expect_or_drop` | `ON VIOLATION DROP ROW` | Invalid records are removed before the write |
| Fail | `@dp.expect_or_fail` | `ON VIOLATION FAIL UPDATE` | The update fails and partial updates roll back |

- Group expectations with `expect_all`, `expect_all_or_drop`, or `expect_all_or_fail`, which take a dictionary of name to constraint. In SQL, separate constraints with commas.
- In a pipeline with parallel flows, a failure in one flow does not stop the others.

### 6. Data type checks

- **Schema enforcement**: Delta compares incoming column types with the table schema and tries a safe cast. If the cast fails, the write raises an error. Inserted columns must exist in the target table.
- **`cast` vs `try_cast`**: `cast` raises an error on invalid input. `try_cast` returns `NULL`, which lets you flag or quarantine bad values.
- **CHECK constraints** on types can use `try_cast(col AS DATE) IS NOT NULL` or a `REGEXP` pattern. Adding one to an existing table checks all existing rows first.
- **Quarantine pattern**: send rows where `try_cast` succeeds to the silver table, and rows where it returns `NULL` to a quarantine table with a reason column.
- **Expectations** on types: `try_cast(col AS DECIMAL(10,2)) IS NOT NULL` with `expect_or_drop`, in Python or SQL.
- The module recommends layering these checks: schema enforcement, explicit casts, CHECK constraints, and pipeline expectations.

### 7. Schema drift

- Schema drift is a structural change over time: columns are added, removed, or renamed. It is different from type validation. Delta schema enforcement rejects structural mismatches by default.
- Three ways to handle it:
  1. **Fail** (strictest): `cloudFiles.schemaEvolutionMode` set to `failOnNewColumns`. A new column raises `UnknownFieldException`, and the stream stops until the schema is updated or the offending data is removed.
  2. **Adapt automatically**: `mergeSchema` on a Delta write, or `MERGE WITH SCHEMA EVOLUTION INTO ...` for MERGE. This can add unexpected columns, so use it on purpose.
  3. **Rescue**: `cloudFiles.schemaEvolutionMode` set to `rescue` with `rescuedDataColumn`. The target schema stays fixed, and unexpected data goes to a JSON `_rescued_data` column. The stream keeps running.
- Auto Loader evolution modes:
  - `addNewColumns` (default): the stream stops once, new columns are added, and it resumes on restart.
  - `rescue`: no evolution, extra data is rescued.
  - `failOnNewColumns`: the stream stops until someone fixes the schema.
  - `none`: no evolution, new columns are ignored unless a rescued data column is set.
  - `addNewColumnsWithTypeWidening`: adds columns and widens supported types, such as `INT` to `LONG`. Needs Runtime 16.4 or later, and is in Public Preview according to the module. Unsupported changes, such as `INT` to `STRING`, still go to rescued data.
- Best practices: document expected schemas in version control, alert on drift even when it is handled, standardize column naming, test edge cases in development, and balance strictness against maintenance.

### 8. Pipeline expectations in detail

- Each expectation has a **name** (use a descriptive name, since it appears in monitoring), a **constraint** (a SQL Boolean expression per record, with no custom Python functions, external calls, or subqueries), and an **action**.
- Warn, drop, and fail are described in section 5.
- **Fail** rolls back the update. Warn and drop record violation counts. Fail does not record metrics, but the error names the expectation and the input record.
- Store shared rules in a Unity Catalog Delta table, not in notebook dictionaries. One row change then updates every pipeline that reads the rule. Load them by tag with a helper function that returns a dictionary, then pass it to `expect_all_or_drop`.
- Monitor in **Jobs & Pipelines**, then the pipeline, then a dataset, then the **Data quality** tab. You can also query expectation metrics from the pipeline event log.

## Quick review checklist

- [ ] I can name the two validation mechanisms and say when each suits a workload.
- [ ] I know the three expectation actions and their Python and SQL forms.
- [ ] I know that a Delta PRIMARY KEY is not enforced.
- [ ] I know that adding NOT NULL or a CHECK to an existing table fails if existing rows break the rule.
- [ ] I can explain `cast` against `try_cast`, and when to quarantine rows.
- [ ] I can name the Auto Loader schema evolution modes and their default behavior.
- [ ] I know that `addNewColumnsWithTypeWidening` needs Runtime 16.4 or later.
- [ ] I know the difference between schema drift and type validation.
- [ ] I know that fail-action violations do not show in the metrics, but the error message does.
- [ ] I know that expectation constraints cannot use custom Python functions, external calls, or subqueries.

## Knowledge check and exercise

The exercise and knowledge check were not captured in these notes. Review them on the module page before the exam.

## References

- Module: https://learn.microsoft.com/en-us/training/modules/implement-manage-data-quality-constraints-unity-catalog/
- Unit 2, validation checks: https://learn.microsoft.com/en-us/training/modules/implement-manage-data-quality-constraints-unity-catalog/2-implement-validation-checks
- Unit 3, data type checks: https://learn.microsoft.com/en-us/training/modules/implement-manage-data-quality-constraints-unity-catalog/3-implement-data-type-checks
- Unit 4, schema drift: https://learn.microsoft.com/en-us/training/modules/implement-manage-data-quality-constraints-unity-catalog/4-detect-manage-schema-drift
- Unit 5, pipeline expectations: https://learn.microsoft.com/en-us/training/modules/implement-manage-data-quality-constraints-unity-catalog/5-manage-data-quality-pipeline-expectations

Source note: several unit pages returned condensed text rather than verbatim content, so some wording above is a summary. Check exact syntax against the module pages before relying on it.

## Back to index

[README](../README.md)
