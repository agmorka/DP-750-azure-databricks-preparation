# DP-750 Preparation: Azure Databricks Data Engineer

## Purpose

This repository holds my study notes for the Microsoft DP-750 exam, which covers implementing data engineering solutions with Azure Databricks.

The notes follow the Microsoft Learn training content. For each learning path and module, I summarize what I need to know and link to the original source, so I can review quickly and check the details when needed.

## How the repo is organized

- `paths/`: one note per Microsoft Learn learning path, with its modules and prerequisites.
- `modules/`: one note per module, with key concepts, hands-on lab notes, and references.

## Index

Each learning path lists its modules. Status is shown next to each item.

### Path 1: Set up and Configure an Azure Databricks Environment

[Path overview](paths/set-up-and-configure-environment.md): the first path, covering workspaces, architecture, integrations, compute, and Unity Catalog. Overview done.

- [01. Explore Azure Databricks](modules/01-explore-azure-databricks.md): workspace setup, workloads, key concepts, and data governance. Notes done (knowledge check not reviewed).
- [02. Understand Azure Databricks architecture](modules/02-understand-azure-databricks-architecture.md): account hierarchy, control and compute planes, and storage. Notes done (knowledge check not reviewed).
- [03. Understand Azure Databricks integrations](modules/03-understand-azure-databricks-integrations.md): Fabric, Power BI, VS Code, Power Platform, Copilot Studio, Purview, and Foundry. Notes done (knowledge check not reviewed).
- [04. Select and configure compute in Azure Databricks](modules/04-select-and-configure-compute.md): compute types, performance, features, libraries, and permissions. Notes done (exercise steps and knowledge check not reviewed).
- [05. Create and organize objects in Unity Catalog](modules/05-create-and-organize-objects-in-unity-catalog.md): catalogs, schemas, tables, views, volumes, DDL, foreign catalogs, and Genie. Notes done (knowledge check not reviewed).

### Path 2: Secure and Govern Unity Catalog Objects in Azure Databricks

[Path overview](paths/secure-and-govern-unity-catalog.md): the second path, covering permissions, row filters, column masks, credentials, governance, and Delta Sharing. Overview done.

- [06. Secure Unity Catalog objects](modules/06-secure-unity-catalog-objects.md): privileges, row filters, column masks, Key Vault, service principals, and managed identities. Notes done (exercise and knowledge check not reviewed).
- [07. Govern Unity Catalog objects](modules/07-govern-unity-catalog-objects.md): comments and tags, ABAC, retention, lineage, audit logs, and Delta Sharing. Notes done (exercise and knowledge check not reviewed).

### Path 3: Prepare and Process Data with Azure Databricks

[Path overview](paths/prepare-and-process-data.md): the third path, covering data modeling, ingestion, transformation, and data quality. Overview done, all modules have notes.

- [08. Design and implement data modeling with Azure Databricks](modules/08-design-implement-data-modeling-unity-catalog.md): ingestion design, table formats, partitioning, SCD, granularity, and clustering. Notes done (exercise and knowledge check not reviewed).
- [09. Ingest data into Unity Catalog](modules/09-ingest-data-into-unity-catalog.md): Lakeflow Connect, notebooks, SQL loads, AUTO CDC, Structured Streaming, Auto Loader, and declarative pipelines. Notes done (exercise and knowledge check not reviewed).
- [10. Cleanse, transform, and load data into Unity Catalog](modules/10-cleanse-transform-load-data-into-unity-catalog.md): profiling, column types, duplicates and nulls, filters, joins, pivots, and load patterns. Notes done (exercise and knowledge check not reviewed).
- [11. Implement and manage data quality constraints with Azure Databricks](modules/11-implement-manage-data-quality-constraints.md): validation checks, data type checks, schema drift, and pipeline expectations. Notes done (exercise and knowledge check not reviewed).

### Path 4: Deploy and Maintain Data Pipelines and Workloads with Azure Databricks

[Path overview](paths/deploy-maintain-data-pipelines-workloads.md): the fourth path, covering pipeline design, Lakeflow Jobs, development lifecycle, and monitoring and optimization. Overview done, all modules have notes.

- [12. Design and implement data pipelines with Azure Databricks](modules/12-design-and-implement-data-pipelines.md): order of operations, notebooks vs Lakeflow pipelines, job task logic, error handling, and declarative pipelines. Notes done (exercise and knowledge check not reviewed).
- [13. Implement Lakeflow Jobs with Azure Databricks](modules/13-implement-lakeflow-jobs.md): job structure, permissions, triggers, cron schedules, alerts, and retries. Notes done (exercise and knowledge check not reviewed).
- [14. Implement development lifecycle processes in Azure Databricks](modules/14-implement-development-lifecycle-processes.md): Git folders, branches and pull requests, testing strategy, Declarative Automation Bundles, and CLI deployment. Notes done (exercise and knowledge check not reviewed).
- [15. Monitor, troubleshoot and optimize workloads in Azure Databricks](modules/15-monitor-troubleshoot-optimize-workloads.md): cluster consumption, job repair, Spark UI diagnostics, skew, spill, shuffle, and log streaming. Notes done (exercise and knowledge check not reviewed).

## Sources

All content comes from Microsoft Learn and related Microsoft documentation. Each note lists its references at the bottom.
