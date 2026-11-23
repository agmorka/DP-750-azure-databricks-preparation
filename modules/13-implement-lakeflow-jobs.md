# Module 13: Implement Lakeflow Jobs with Azure Databricks

Module link: https://learn.microsoft.com/en-us/training/modules/implement-lakeflow-jobs/

Part of: [Deploy and Maintain Data Pipelines and Workloads with Azure Databricks](../paths/deploy-maintain-data-pipelines-workloads.md)

## Overview

- Units: introduction, create a job, configure triggers, schedule a job, configure alerts, configure automatic restarts, exercise, knowledge check, and summary
- Prerequisites: Azure Databricks workspaces and Unity Catalog basics, plus notebooks
- Why it matters: the exam asks which trigger, schedule, alert, or retry setting fits a given workload, and what the run identity needs to access

## Learning objectives

1. Create and configure Lakeflow Jobs with tasks and compute.
2. Configure triggers: table updates and file arrivals.
3. Schedule jobs with intervals and cron expressions.
4. Configure job alerts and notifications.
5. Configure automatic restarts and retry policies.

## Key points

### 1. Job structure

- A job has one or more **tasks** arranged as a DAG. Each job needs a unique name, at least one task with logic, and compute to run it.
- Task types include notebook, Python script, Python wheel, SQL, pipeline, dbt, JAR, Spark Submit, and Run Job. If/else and For each tasks are also available.
- Code sources: workspace, Git provider (all tasks share one Git reference, and the run uses a commit snapshot), or a full URI for Python scripts in ADLS.
- **Compute**:
  - Serverless is the default for supported tasks.
  - Classic jobs compute lets you set the Spark version, instance types, and autoscaling.
  - SQL warehouses run SQL tasks (serverless or pro).
  - Tasks that share compute keep it running until all of them finish.
- In **Job details**, the Compute section shows each task's resources. **Configure** edits classic compute. **Swap** changes compute for all tasks that use a resource.
- Dependencies: **Depends on** plus a **Run if** condition. The six conditions are All succeeded, At least one succeeded, None failed, All done, At least one failed, and All failed. All done is for cleanup, and At least one failed is for error handling.
- Parameters: add key-value pairs in Job details. Notebooks read them with `dbutils.widgets.get`. Python scripts receive them as command-line arguments. Dynamic values such as `{{job.trigger.time.iso_date}}` work.
- **Tags** on a job filter and group jobs and propagate to job clusters for cost tracking.

### 2. Permissions and run identity

- Job permission levels:
  - **CAN VIEW**: see configuration, tasks, and run history.
  - **CAN RUN**: view, plus trigger runs.
  - **CAN MANAGE RUN**: run, plus cancel runs, view output, and restart failed runs.
  - **CAN MANAGE**: manage run, plus edit configuration, tasks, schedules, and permissions.
- The job creator and workspace admins get CAN MANAGE automatically. Set permissions in the **Permissions** tab.
- Jobs run as the job owner or a service principal, not as the user who triggered them.
- The **run identity** is the principal that Unity Catalog checks at runtime. Use a service principal for production.
- Required privileges: `SELECT` to read a table, `MODIFY` to write, `CREATE TABLE` and `USE SCHEMA` to create tables in a schema, and `READ VOLUME` or `WRITE VOLUME` for volumes.
- Without these privileges the job fails at runtime with an authorization error. Check access before scheduling.

### 3. Triggers

| Trigger | Behavior |
|---|---|
| Table update | Runs when monitored tables change |
| File arrival | Runs when new files appear in a Unity Catalog location |
| Continuous | Starts the next run right after the previous run ends, whether it succeeded or failed |

- **Table update** supports Unity Catalog Delta and Iceberg managed tables, external Delta tables, materialized views, streaming tables, and views on those tables.
  - **Any table** runs when any monitored table changes. **All tables** waits until every monitored table has changed.
  - **Minimum time between triggers** sets a gap after a run.
  - **Wait after last change** delays the run, and resets with each new update. It batches rapid changes.
  - Dynamic values: `{{job.trigger.table_update.updated_tables}}`, plus per-table `commit_timestamp` and `version`.
  - Enable **file events** on the external location for efficiency.
- **File arrival** monitors a path (a volume or external location), including subdirectories. The check runs about every minute on a best-effort basis.
  - Needs READ on the storage location and CAN MANAGE on the job.
  - Triggers only on new files. Overwriting a file with the same name does not trigger a run, so use unique names.
  - Without file events: 50 jobs with file arrival triggers per workspace, and 10,000 files per monitored location. With file events enabled, these limits do not apply.
- **Continuous** runs the job without stopping. A failed run still triggers the next one. On serverless, only bounded triggers such as `Trigger.AvailableNow` are supported. `Trigger.ProcessingTime` and `Trigger.Continuous` need classic jobs compute, or a continuous declarative pipeline for low latency.
- Pause and resume are in **Schedules & Triggers**. Pausing stops new runs and lets active runs finish. Configuration is kept.

### 4. Schedules

- **Simple** schedules run at a fixed interval in minutes, hours, days, or weeks. You cannot choose the first run time. The minimum interval is 10 seconds.
- **Advanced** schedules use cron and a time zone. The cron format in this module has six or seven fields: seconds, minutes, hours, day of month, month, day of week, and an optional year.
- Special characters: `*` (all), `?` (no specific value, for day of month or day of week), `-` (range), `,` (list), `/` (increment), `L` (last), and `#` (nth weekday).
- Examples from the unit:
  - `0 0 12 * * ?`: every day at noon.
  - `0 15 10 ? * MON-FRI`: weekdays at 10:15.
  - `0 0 8 1 * ?`: the first of each month at 8:00.
  - `0 30 6 ? * 6L`: the last Friday of each month at 6:30.
  - `0 0 */2 * * ?`: every 2 hours.
- Time zone and DST: an hourly job can skip or shift around DST changes. Use UTC when intervals must be consistent.
- **Maximum concurrent runs** limits how many runs of one job can overlap. Set it to 1 for jobs that write to the same tables.
- When the limit is reached, choose **Queue** (wait, the usual choice for pipelines), **Cancel**, or **Skip**. Queued runs use no compute while they wait.
- Production notes: the scheduler is reliable but not low-latency, leave buffer time between dependent jobs, and watch run history for delays.

### 5. Alerts and notifications

- Events: **Start**, **Success**, **Failure**, **Duration warning** (run exceeds a threshold), and **Streaming backlog** (backlog above a threshold for 10 minutes).
- A job that ends in "Succeeded with failures" counts as a success.
- Destinations: email addresses, or system destinations (Slack, Microsoft Teams, PagerDuty, HTTP webhook). A workspace administrator creates system destinations under **Admin Settings > Notifications**. Each system destination needs credentials. Limit: three system destinations per event type, per job or task.
- **Job-level** notifications: **Job details > Job notifications > Edit notifications**. They are not sent when individual tasks fail and retry.
- **Task-level** notifications: set on the task panel. They fire for each task run and show which step failed.
- **Mute notifications until the last retry** stops alerts for each retry. You still get an alert if the task finally fails. The alerts unit says tasks retry three times by default. That conflicts with the restarts unit and the configure-task page, which say most configurations do not retry by default. Treat the alerts unit's claim as wrong here (see section 6).
- Noise reduction: mute skipped and canceled runs (job-level only), set duration thresholds from historical run times, and document why each alert exists.
- Webhook payloads are JSON with an `event_type` (for example `jobs.on_failure`), a workspace ID, and run and job IDs.
- If Azure Data Factory orchestrates the jobs, use ADF alerts for orchestration metrics and Databricks notifications for job events.

### 6. Automatic restarts and retries

- Default: most jobs do not retry. You must configure retries explicitly.
- Exceptions: serverless jobs auto-optimize retries by default. For continuous jobs, the default task retry mode is **On failure**, which retries failed tasks inside the job. Exponential backoff applies to repeated failures (see below).
- Jobs that must run exactly once, such as non-idempotent writes, need auto-optimization turned off.
- **Task retries**: open the task in **Tasks**, select **+ Add** next to **Retries**, then set the count and interval. The interval is in milliseconds, and it is measured from the start of the failed run to the start of the retry.
- **Continuous jobs**: add a Continuous trigger, then set **Task retry mode**. **On failure** retries failed tasks inside the job (default for new continuous jobs). **Never** retries only at job level.
- Exponential backoff: after consecutive failures, the wait grows up to a system maximum. A successful run, or a period without failures, resets it.
- Continuous jobs do not support task dependencies or task-level retry policies in the usual way. Use task retry mode.
- **Timeouts**: under **Metric thresholds**, choose **Run duration**. The **Warning** value sends a notification. The **Timeout** value ends the task. A timed-out task gets the status "Timed Out" and goes through the retry policy. The timeout applies to each attempt.
- Recommended limits: 1 to 3 retries, with intervals of 30 seconds to 5 minutes for transient failures. Longer intervals suit resource problems.
- Best practices: make tasks idempotent, track retry rates for infrastructure or data problems, and document the chosen settings.

## Quick review checklist

- [ ] I can name the six Run if conditions and when to use All done.
- [ ] I know the four job permission levels and what each allows.
- [ ] I know why production jobs should run as a service principal.
- [ ] I can list the privileges a run identity needs for read, write, create, and volume access.
- [ ] I know the three trigger types and their limits (50 jobs and 10,000 files without file events).
- [ ] I know that overwriting a file with the same name does not start a file arrival trigger.
- [ ] I know which trigger types are not supported on serverless.
- [ ] I can read a six-field cron expression, including `L` and `#`.
- [ ] I know that simple schedules cannot set the first run time, and the 10-second minimum.
- [ ] I know the three concurrency behaviors and why Queue is the usual choice.
- [ ] I know the difference between job-level and task-level notifications.
- [ ] I know the default task retry mode for continuous jobs (On failure), and that timeouts apply per attempt.

## Knowledge check and exercise

The exercise and knowledge check were not captured in these notes. Review them on the module page before the exam.

## References

- Module: https://learn.microsoft.com/en-us/training/modules/implement-lakeflow-jobs/
- Unit 2, create a job: https://learn.microsoft.com/en-us/training/modules/implement-lakeflow-jobs/2-create-lakeflow-job
- Unit 3, configure triggers: https://learn.microsoft.com/en-us/training/modules/implement-lakeflow-jobs/3-configure-job-triggers
- Unit 4, schedule a job: https://learn.microsoft.com/en-us/training/modules/implement-lakeflow-jobs/4-schedule-job
- Unit 5, configure alerts: https://learn.microsoft.com/en-us/training/modules/implement-lakeflow-jobs/5-configure-job-alerts
- Unit 6, automatic restarts: https://learn.microsoft.com/en-us/training/modules/implement-lakeflow-jobs/6-configure-job-automatic-restarts
- Unit 9, summary: https://learn.microsoft.com/en-us/training/modules/implement-lakeflow-jobs/9-summary

Source note: several unit pages returned condensed text rather than verbatim content, so some wording above is a summary. Check exact syntax against the module pages before relying on it.

## Back to index

[README](../README.md)
