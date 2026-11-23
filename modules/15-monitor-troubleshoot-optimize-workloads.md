# Module 15: Monitor, troubleshoot and optimize workloads in Azure Databricks

Module link: https://learn.microsoft.com/en-us/training/modules/monitor-troubleshoot-optimize-workloads-azure-databricks/

Part of: [Deploy and Maintain Data Pipelines and Workloads with Azure Databricks](../paths/deploy-maintain-data-pipelines-workloads.md)

## Overview

- Units: introduction, cluster consumption, troubleshooting Lakeflow Jobs, troubleshooting Spark jobs and notebooks, caching, skew, spill and shuffle, log streaming to Azure Log Analytics, exercise, knowledge check, and summary
- Prerequisites: Lakeflow Jobs and Spark basics
- Why it matters: this is the operations module. The exam asks you to read the right metric, job state, or Spark UI signal, and to choose the fix that matches it

## Learning objectives

1. Monitor and manage cluster consumption with metrics, auto-termination, and budgets.
2. Troubleshoot and repair failed Lakeflow Jobs.
3. Diagnose Spark job failures and bottlenecks with the Spark UI and compute metrics.
4. Investigate and resolve caching, data skew, memory spill, and shuffle problems.
5. Stream Azure Databricks logs to Azure Log Analytics.

## Key points

### 1. Cluster consumption

- Two costs to avoid: idle clusters (left running overnight or over a weekend) and under-provisioned clusters (slow jobs). The aim is to match resources to the workload.
- **Compute metrics** are in **Compute**, then the cluster, then the **Metrics** tab. Three groups:
  - Hardware: CPU, memory, and network.
  - Spark: active tasks, failed and completed tasks, and shuffle read and write.
  - GPU: Databricks Runtime ML 13.3 or later.
- Metrics cover up to the past 30 days, and can be shown per node or aggregated.
- Serverless compute for notebooks and jobs uses query insights, not the metrics UI.
- **SQL warehouses**: the **Monitoring** tab shows live statistics, peak query count (spikes mean the warehouse cannot keep up), running clusters, and query history.
- **Auto-termination** stops a cluster after a period of inactivity (30 to 60 minutes is typical for development). A cluster counts as inactive only when all commands have finished. Auto-termination does not track DStream activity, so turn it off for DStream workloads or move to Structured Streaming.
- **Autoscaling** changes worker count between a minimum and a maximum. The unit says it can cut costs by 20 to 40 percent compared with a fixed size.
- **Budgets** send email alerts as spend approaches a limit. Creating budgets needs account admin rights. Workspace admins can view them.
- **Tags** flow from clusters and workspaces into billing records. Apply them from the start, because tags cannot be added to past usage.
- System tables: `system.billing.usage` for usage, and `system.compute.clusters` for cluster owners. Examples from the unit:
  - Daily DBUs for `STANDARD_ALL_PURPOSE_COMPUTE`, grouped by `usage_date`.
  - DBUs per job with `usage_metadata.job_id`.
  - DBUs by cluster owner, joining usage and clusters on `cluster_id`.
  - Usage by custom tag, using `custom_tags['project']`.
- For serverless, filter with `product_features.is_serverless = true`, use `identity_metadata.run_as` for the user or service principal, and use `SUM()` because one run can produce several billing rows.

### 2. Troubleshooting Lakeflow Jobs

- A run's status comes from its **leaf tasks** (tasks with no downstream dependency):
  - **Failed**: a leaf task did not succeed.
  - **Succeeded with failures**: some tasks failed, but all leaf tasks succeeded.
  - **Skipped**: the run exceeded maximum concurrent runs for the job or workspace.
  - **Timed Out**: the run exceeded its maximum duration.
  - **Canceled**: stopped by a user or an automated process.
- A **Disabled** task (in the DAG) is either disabled in settings or has a disabled upstream dependency.
- Find the cause in the **Runs** tab. The matrix view shows each task across runs. Hover over a red task for details, then open **Task run details** for logs.
- Patterns: one task failing repeatedly points to its code or configuration. Failures spread across tasks point to the cluster or a shared resource.
- **Diagnose Error** in the Jobs UI gives suggestions from Genie Code.
- Common fixes: **Edit task** (paths, parameters, timeouts), more compute or a different instance type, waiting for an external dependency, or raising **Maximum concurrent runs**.
- **Repair run** re-runs only the failed tasks and the tasks that depend on them. You can change task parameters for that repair only. Repair is available only for jobs with two or more tasks. With a shared job cluster, repair creates a new cluster version (for example `my_job_cluster_v1`) so you can compare settings.
- **Stop** ends an active run. **Cancel runs** and **Cancel all queued runs** handle several runs.
- **Restart run** on a continuous job cancels the active run, resets the backoff, and starts a new run immediately. Use stop when a job uses too many resources or writes bad data. Use restart after the fix.
- Ongoing monitoring: notifications, run history in the matrix view, and the `system.lakeflow` schema for account-wide job analysis.

### 3. Spark jobs and notebooks

- Three failure types:
  - **Code-related**: syntax errors, wrong transformations, schema mismatches.
  - **Resource bottlenecks**: out-of-memory errors, slow shuffles, repeated task failures.
  - **Environmental**: cluster start failures, network problems, unavailable sources, quota limits, or spot instance reclamation.
- **Spark UI** (from the cluster page): the Jobs Timeline shows failing jobs (red), long-running jobs, and execution gaps. A gap longer than a minute in the middle of a pipeline can mean an overloaded driver or a cluster problem.
- In the longest stage, check task metrics: **Input**, **Output**, **Shuffle Read**, and **Shuffle Write**. A stage with one task means too little parallelism. Uneven task durations suggest skew.
- Compute **Metrics** shows a server load distribution. A red driver with blue workers means the driver is overloaded.
- Bottleneck fixes:
  - Memory pressure or spill: bigger workers, fewer or smaller partitions, less data in memory.
  - CPU limits: Photon for compatible workloads, or more workers.
  - Network and shuffle: filter earlier, broadcast small tables, and repartition less.
- **Restart** a cluster to clear exhausted resources, broken executors, or stale images. Check the cluster's **Event log** first. A restart stops running jobs and clears Spark UI history. Restart from the UI (**Restart**) or with `databricks clusters restart <CLUSTER_ID>`.
- After a fix, confirm the next run returns to expected times with no new errors.

### 4. Caching, skew, spill, and shuffle

- Diagnostic tools: the **DAG** (overall data flow), the **Spark UI** (timelines and stage metrics), and the **query profile** for SQL warehouses (**Query History**, then **See query profile**). Start with the DAG, then drill into stages.
- **Disk cache** (formerly Delta cache) automatically caches Parquet files on workers. Check it with `spark.conf.get("spark.databricks.io.cache.enabled")`.
  - Under-caching: stages re-read the same data (high Input). Enable disk cache, and use SSD-backed workers.
  - Over-caching: cached data uses memory that processing needs. Spark cache (`.cache()` or `.persist()`) is manual, so release it with `df.unpersist()`.
- **Data skew**: some partitions hold far more data. In **Summary Metrics** for a stage, if the maximum duration is more than 50 percent above the 75th percentile, suspect skew.
  - Causes: uneven join or aggregation keys, null values landing in one partition, or time partitions where recent dates are much larger.
  - Fixes: check Adaptive Query Execution with `spark.conf.get("spark.databricks.optimizer.adaptive.enabled")`, salt skewed keys, or broadcast a small table.
- **Memory spill**: Spark writes to disk when memory runs short. Check **Shuffle Spill (Memory)** and **Shuffle Spill (Disk)** on the stage page. Any value above zero signals memory pressure.
  - Fixes: more partitions, shuffle partitions set to `auto` (`spark.conf.set("spark.sql.shuffle.partitions", "auto")`), more memory per core, or less data.
  - Spill often comes from skew. Fix skew first.
- **Shuffle**: data moves between nodes during joins, aggregations, and repartitioning. Check Shuffle Read and Write. In the DAG, shuffles are exchange nodes. In query profiles, they are **Exchange** nodes.
  - Reduce it by filtering early, broadcasting small tables (default threshold under 30 MB, and AQE can switch joins to broadcast at runtime), avoiding extra `repartition()` calls, and bucketing or partitioning tables that are joined on the same key.

### 5. Log streaming to Azure Log Analytics

- Log streaming uses Azure diagnostic settings to send Databricks audit and diagnostic events to a Log Analytics workspace, usually within 15 minutes. It needs the Premium plan.
- Platform administrators set up diagnostic settings in the Azure portal. Data engineers use the logs.
- Log Analytics tables: `DatabricksClusters`, `DatabricksJobs`, `DatabricksNotebook`, `DatabricksSQL`, `DatabricksTables`, `DatabricksWorkspace`, and `DatabricksSecrets`.
- Standard columns: `TimeGenerated`, `OperationName`, `Identity`, `SourceIPAddress`, `RequestParams`, and `Response`.
- For job problems, start with `DatabricksJobs`. For security reviews, use `DatabricksSecrets` and `DatabricksTables`.
- Query with KQL in the **Logs** blade. Examples from the unit: recent job events (`ago(24h)`), failed jobs grouped by hour (`summarize ... by bin(TimeGenerated, 1h)`), and table create, delete, or update events.
- Alerts: from a KQL query, select **New alert rule**, set the condition, and add an action group (email, SMS, the Azure mobile app, or ITSM tools). Examples from the unit: more than five job failures in an hour, and table creates or deletes outside working hours.
- The unit says `system.access.audit` is the recommended route for audit queries inside Azure Databricks. Use log streaming for cross-service monitoring and Azure Monitor alerts.

## Quick review checklist

- [ ] I know the metrics categories (hardware, Spark, GPU) and the 30-day window.
- [ ] I know what auto-termination ignores and why DStreams need special handling.
- [ ] I can name the system tables used for cost queries and the key columns (`usage_quantity`, `usage_metadata`, `custom_tags`).
- [ ] I know the five job end states and how the leaf tasks decide the run status.
- [ ] I can explain repair run: it re-runs failed tasks and their dependents only.
- [ ] I know when to stop a run and when to restart a continuous job.
- [ ] I know the three failure types and the Spark UI signals for each.
- [ ] I can identify skew (max more than 50 percent above the 75th percentile) and spill (non-zero spill metrics).
- [ ] I know that fixing skew often fixes spill.
- [ ] I know the default broadcast threshold (under 30 MB) and the Exchange nodes in query profiles.
- [ ] I know the Log Analytics tables for jobs, tables, and secrets, and that log streaming needs the Premium plan.

## Knowledge check and exercise

The exercise and knowledge check were not captured in these notes. The module summary also did not load. Review them on the module page before the exam.

## References

- Module: https://learn.microsoft.com/en-us/training/modules/monitor-troubleshoot-optimize-workloads-azure-databricks/
- Unit 2, cluster consumption: https://learn.microsoft.com/en-us/training/modules/monitor-troubleshoot-optimize-workloads-azure-databricks/2-monitor-manage-cluster-consumption
- Unit 3, troubleshoot and repair Lakeflow Jobs: https://learn.microsoft.com/en-us/training/modules/monitor-troubleshoot-optimize-workloads-azure-databricks/3-troubleshoot-repair-lakeflow-jobs
- Unit 4, troubleshoot Spark jobs and notebooks: https://learn.microsoft.com/en-us/training/modules/monitor-troubleshoot-optimize-workloads-azure-databricks/4-troubleshoot-repair-spark-jobs-notebooks
- Unit 5, caching, skew, spill, shuffle: https://learn.microsoft.com/en-us/training/modules/monitor-troubleshoot-optimize-workloads-azure-databricks/5-resolve-cache-skew-spill-shuffle
- Unit 6, log streaming: https://learn.microsoft.com/en-us/training/modules/monitor-troubleshoot-optimize-workloads-azure-databricks/6-implement-log-streaming-azure-analytics

Source note: several unit pages returned condensed text rather than verbatim content, so some wording above is a summary. Check exact syntax against the module pages before relying on it.

## Back to index

[README](../README.md)
