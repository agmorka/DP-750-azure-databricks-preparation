# Module 10: Cleanse, transform, and load data into Unity Catalog

Module link: https://learn.microsoft.com/en-us/training/modules/cleanse-transform-load-data-into-unity-catalog/

Part of: [Prepare and Process Data with Azure Databricks](../paths/prepare-and-process-data.md)

## Overview

- Units: introduction, content units on profiling, column types, duplicates and nulls, filters and aggregations, joins and set operators, denormalization and pivots, and loading (merge, insert, append), then exercise, knowledge check, and summary
- Prerequisites: Unity Catalog basics, SQL, and PySpark basics
- Why it matters: the exam tests how to check data before and after transforming it, and which load operation fits a given case

## Learning objectives

1. Profile data with summary statistics.
2. Choose appropriate column data types.
3. Find and resolve duplicates and nulls.
4. Filter, group, and aggregate data.
5. Combine tables with joins and set operators.
6. Flatten, pivot, and unpivot data.
7. Load data with append, overwrite, and merge operations.

## Key points

### 1. Profile data

- Profiling produces counts, averages, value distributions, and null percentages. It shows missing values, outliers, and unexpected patterns before they reach downstream tables.
- SQL: `ANALYZE TABLE ... COMPUTE STATISTICS` (table level), `FOR COLUMNS col1, col2` (specific columns), or `FOR ALL COLUMNS`.
- `DESC EXTENDED table col` shows min, max, null count, distinct count, and average and maximum length for a column.
- `DESCRIBE TABLE EXTENDED` shows table-level details.
- PySpark: `df.describe()` returns count, mean, standard deviation, min, and max for numeric columns.
- Run `ANALYZE TABLE` after large loads. For managed tables, predictive optimization runs `ANALYZE` automatically.
- Data quality monitoring in Unity Catalog has two parts. **Anomaly detection** runs at schema level and checks freshness and completeness. **Data profiling** is set per table.
- Profile types: **snapshot** (whole table on each refresh), **time series** (tables with a timestamp column), and **inference** (ML model inference tables).
- A profile creates a profile metrics table and a drift metrics table. Drift uses the Kolmogorov-Smirnov test for numeric columns and the chi-squared test for categorical columns.

### 2. Choose column data types

- Integral: `TINYINT`, `SMALLINT`, `INT`, `BIGINT`.
- Decimal: `DECIMAL(p,s)`, exact. Use it for money.
- Floating point: `FLOAT`, `DOUBLE`, approximate. They can cause rounding errors with base-10 values such as currency.
- Temporal: `DATE`, `TIMESTAMP` (stored in UTC and shown in the session time zone), `TIMESTAMP_NTZ` (no time zone conversion).
- Text and binary: `STRING`, `BINARY`.
- Complex: `ARRAY<T>`, `MAP<K,V>`, `STRUCT<...>`. Access them with `arr[0]`, `map['key']`, and `struct.field`.
- Best practices: match the type to the data's range, document nullability (types do not enforce `NOT NULL` by default), cast early during ingestion, and use explicit casts rather than relying on implicit conversion.

### 3. Duplicates and nulls

- Find duplicates with `GROUP BY ... HAVING COUNT(*) > 1`, or with `QUALIFY COUNT(*) OVER (PARTITION BY ...) > 1` to see the full rows.
- PySpark: a window count with `filter("row_count > 1")`, or `df.exceptAll(df.dropDuplicates([...]))`. `exceptAll` keeps duplicate occurrences. `except` removes them all.
- Remove duplicates: `dropDuplicates([cols])` keeps one row per combination, but the row kept is not guaranteed. `distinct()` removes fully identical rows.
- To choose which duplicate to keep, use `ROW_NUMBER() OVER (PARTITION BY key ORDER BY updated_at DESC) = 1` with `QUALIFY`.
- Nulls: `isnull()` or `IS NULL` finds them. `dropna()` removes rows (`how="any"` is the default, `"all"` drops only fully null rows, `subset=` limits the columns).
- `fillna()` replaces nulls. `COALESCE(col, default)` does the same in SQL.
- Choosing a strategy: drop rows with nulls in identifiers, fill numeric fields for aggregation, use a placeholder for optional categories, and decide case by case for nulls that would skew results. Document the decision.

### 4. Filter, group, and aggregate

- PySpark `filter()` and `where()` are the same. Combine conditions with `&` and `|`, with each condition in parentheses.
- Use `isNull()` and `isNotNull()`. `!= None` does not reliably filter nulls.
- `groupBy()` returns a `GroupedData` object. Apply `agg()` to get results.
- SQL: `WHERE` filters rows, `GROUP BY` groups them, and `HAVING` filters groups after aggregation. Every selected column must be in `GROUP BY` or wrapped in an aggregate.
- Aggregates: `count`, `sum`, `avg`, `min`, `max`.
- Spark is lazy. Put filters early in a chain to reduce the data that later steps process.

### 5. Joins and set operators

- Join types: `INNER`, `LEFT`, `RIGHT`, `FULL`, `SEMI` (left rows with a match), `ANTI` (left rows without a match), and `CROSS` (all combinations).
- PySpark `how=` values include `"inner"`, `"left"`, `"full"`, `"semi"`, and `"anti"`. A string column name in `on=` avoids duplicate key columns.
- Set operators need the same column count and compatible types.
  - `UNION` removes duplicates. `UNION ALL` keeps them. PySpark `union()` behaves like `UNION ALL`, so add `.distinct()` for `UNION`.
  - `INTERSECT` returns rows present in both queries.
  - `EXCEPT` returns rows from the first query that are not in the second. `MINUS` is a synonym.
- Operator precedence: `INTERSECT` binds tighter than `UNION` and `EXCEPT`. Use parentheses when chaining.
- Joins enrich rows or test existence. Set operators stack rows from sources with the same schema.

### 6. Flatten, pivot, and unpivot

- **Denormalization** flattens related tables into one wider table with joins done ahead of time. It trades storage and refresh effort for faster reads. Keep normalized data in bronze and silver layers, and denormalize gold tables.
- **PIVOT** turns values of a column into columns. It needs an aggregate, and `FOR col IN (value AS alias, ...)` lists the values. Several aggregates can be used in one pivot.
- **UNPIVOT** turns columns into rows. By default it drops nulls. Use `UNPIVOT INCLUDE NULLS` to keep them. The clause needs Databricks Runtime 12.2 LTS or later.
- Choose: denormalize for report queries, pivot for side-by-side comparisons, unpivot to normalize wide data.

### 7. Load data: append, overwrite, merge

- **Append**: `INSERT INTO` for new batches that do not overlap existing rows. It does not check for duplicates, so deduplicate first. PySpark: `write.mode("append")` or `insertInto`.
- **Overwrite**:
  - `INSERT OVERWRITE` replaces the table or a partition.
  - `REPLACE WHERE <predicate>` deletes matching rows, then inserts. Use it to correct a date range.
  - `REPLACE USING (cols)` is the recommended dynamic overwrite. It works on all compute types, including SQL warehouses and serverless. Runtime 16.3 to 17.1 requires `USING` to list all partition columns. Runtime 17.2 and later accept any columns.
  - `REPLACE ON (condition)` supports custom or NULL-safe matching. It needs Runtime 17.1 or later.
  - `INSERT OVERWRITE ... PARTITION` with `partitionOverwriteMode=dynamic` works only on classic compute, not on SQL warehouses or serverless. The module says `REPLACE USING` is preferred for new pipelines, because partition overwrite can use stale data when partitioning changes.
- **Merge**: `MERGE INTO target USING source ON key WHEN MATCHED THEN UPDATE ... WHEN NOT MATCHED THEN INSERT ...`. It supports conditional clauses, `DELETE` in a matched clause, and `INSERT *` or `UPDATE SET *` when schemas match.
- PySpark merge uses `DeltaTable.forName(...).merge(...).whenMatchedUpdate(...).whenNotMatchedInsert(...).execute()`.
- Important: each target row can match only one source row. If several source rows match the same target row, the merge fails. Deduplicate the source first.

## Quick review checklist

- [ ] I can run `ANALYZE TABLE` with the right scope and read column stats with `DESC EXTENDED`.
- [ ] I know the difference between snapshot, time series, and inference profiles.
- [ ] I can choose DECIMAL for money and explain why DOUBLE is not exact.
- [ ] I know the difference between TIMESTAMP and TIMESTAMP_NTZ.
- [ ] I can use `exceptAll` to find duplicates and `QUALIFY ROW_NUMBER()` to keep one row.
- [ ] I know that `dropDuplicates` does not guarantee which row remains.
- [ ] I know that `!= None` is not a valid null check and `isNull()` or `IS NULL` is.
- [ ] I can state the difference between `UNION` and `UNION ALL`, and between `EXCEPT` and `MINUS`.
- [ ] I can list the join types, including SEMI and ANTI.
- [ ] I know the PIVOT and UNPIVOT syntax and that UNPIVOT drops nulls by default.
- [ ] I can choose among INSERT INTO, REPLACE WHERE, REPLACE USING, and MERGE for a scenario.
- [ ] I know that a MERGE fails when several source rows match one target row.

## Knowledge check and exercise

The exercise and knowledge check were not captured in these notes. Review them on the module page before the exam.

## References

- Module: https://learn.microsoft.com/en-us/training/modules/cleanse-transform-load-data-into-unity-catalog/
- Unit 2, profile data: https://learn.microsoft.com/en-us/training/modules/cleanse-transform-load-data-into-unity-catalog/2-profile-data-summary-statistics
- Unit 3, column data types: https://learn.microsoft.com/en-us/training/modules/cleanse-transform-load-data-into-unity-catalog/3-choose-appropriate-column-data-types
- Unit 4, duplicates and nulls: https://learn.microsoft.com/en-us/training/modules/cleanse-transform-load-data-into-unity-catalog/4-resolve-duplicate-missing-null-values
- Unit 5, filter, group, aggregate: https://learn.microsoft.com/en-us/training/modules/cleanse-transform-load-data-into-unity-catalog/5-transform-data-filter-group-aggregate
- Unit 6, joins and set operators: https://learn.microsoft.com/en-us/training/modules/cleanse-transform-load-data-into-unity-catalog/6-transform-data-join-union-intersect-except
- Unit 7, flatten, pivot, unpivot: https://learn.microsoft.com/en-us/training/modules/cleanse-transform-load-data-into-unity-catalog/7-transform-data-flatten-pivot-unpivot
- Unit 8, merge, insert, append: https://learn.microsoft.com/en-us/training/modules/cleanse-transform-load-data-into-unity-catalog/8-load-data-merge-insert-append

Source note: several unit pages returned condensed text rather than verbatim content, so some wording above is a summary. Check exact syntax against the module pages before relying on it.

## Back to index

[README](../README.md)
