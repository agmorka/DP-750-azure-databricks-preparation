# Module 01: Explore Azure Databricks

Module link: https://learn.microsoft.com/en-us/training/modules/explore-azure-databricks/

Part of: [Set up and Configure an Azure Databricks Environment](../paths/set-up-and-configure-environment.md)

## Overview

- Level: intermediate
- Role: data analyst (the page also lists data engineers and data scientists)
- Length: 8 units. The page metadata lists about 68 minutes in total.
- Prerequisite: basic data analytics knowledge. Microsoft suggests the Azure Data Fundamentals certification first.

Azure Databricks is a cloud service for data analytics. It runs on Apache Spark.

## Learning objectives

1. Create an Azure Databricks workspace.
2. Name the main workloads.
3. Use Unity Catalog and Microsoft Purview for governance.
4. Explain the key concepts of an Azure Databricks solution.

## Key points

### 1. Introduction: the platform idea

- Azure Databricks brings data engineering, data science, and machine learning into one workspace.
- It connects to many sources, including Azure SQL, Amazon S3, Google Cloud Storage, SAP, and Oracle.
- A lakehouse mixes the strengths of a data lake and a data warehouse. It keeps one source of truth.
- The medallion architecture refines data in layers as it moves through processing.
- The two key technologies are Delta Lake (storage with ACID transactions and schema enforcement) and Unity Catalog (governance).

### 2. Get started: workspace setup

- Create a workspace in your Azure subscription. You can use the Azure portal, ARM, Bicep, Terraform, PowerShell, or the Azure CLI.
- You choose a name, a region, and a pricing tier.
- The pricing tiers are Premium and Trial (a 14-day free trial of Premium). The Standard tier is no longer available for new workspaces since April 1, 2026.
- The workspace types are Serverless (recommended for most cases) and Hybrid (also called Classic). Hybrid is for custom networking or on-premises connections.
- The Free Edition is for students and educators. It has daily usage limits and no classic compute.
- The workspace UI has a sidebar with sections such as Workspace, Recents, Catalog, Jobs & Pipelines, Compute, and Marketplace. The **+ New** button creates objects and compute.
- Genie Code (formerly Databricks Assistant) helps write, explain, and fix code. Its Agent mode can run multi-step data tasks.

### 3. Workloads

- **Data engineering:** Spark-based processing and pipelines. Python, R, Scala, and SQL are supported.
- **Machine learning:** MLflow manages the ML lifecycle. TensorFlow, PyTorch, and scikit-learn are supported.
- **SQL:** SQL warehouses, a SQL editor, and dashboards. SQL warehouses need Premium.
- The overview also names LLM work, BI and data warehousing, and streaming.

### 4. Key concepts

- **Workspace:** a shared environment for notebooks, clusters, jobs, and dashboards. It is linked to Unity Catalog.
- **Notebooks:** mix code, charts, and Markdown. You can switch language with magic commands.
- **Two planes:** Microsoft manages the control plane. The compute plane runs in your own Azure subscription.
- **Clusters:** a driver node coordinates work. Worker nodes do the processing. Clusters can scale automatically.
- **Compute types:** serverless (managed and fast to start), classic (full control), and SQL warehouses (serverless, pro, or classic).
- **Databricks Runtime:** a tuned Spark build. Know the lifecycle stages: Legacy, Deprecated, End of Support, and End of Life. Restart a cluster to apply maintenance.
- **Lakeflow Jobs:** scheduled or event-driven workflows made of tasks.
- **Delta Lake:** adds ACID transactions, time travel, and rollback. Delta is the default table format.
- **Databricks SQL:** data warehousing with ANSI SQL. Premium only.
- **MLflow:** tracks experiments and models, and moves models to production.

### 5. Data governance: Unity Catalog and Microsoft Purview

- **Unity Catalog** manages access, discovery, lineage, audit logs, and quality monitoring. There is usually one metastore per region.
- Names follow the pattern `catalog.schema.object`.
- **Managed tables** are fully governed by Unity Catalog and always use Delta.
- **External tables** are governed by Unity Catalog, but the storage is managed outside it. They can use Delta, CSV, JSON, or Parquet.
- Access is granted with SQL, for example `GRANT CREATE TABLE ON SCHEMA mycatalog.myschema TO finance-team`. Control can go down to rows and columns.
- Unity Catalog is usually enabled by default when you create a workspace.
- **Microsoft Purview** discovers and classifies data across clouds and SaaS. Once integrated with Unity Catalog, it scans metadata, finds sensitive data, and shows lineage. Scans can run on demand or on a schedule. It can also scan the Hive metastore.

## Hands-on exercise: Explore Azure Databricks (45 minutes)

The exercise is a lab called CityMoves Transit, a fictional transport authority. You will:

1. Explore the workspace sidebar. Do not create anything yet.
2. Ask Genie Code a question about what you can do as a data engineer.
3. Create a Unity Catalog volume named `lab_data` in the `adb-dp750` catalog, in the `default` schema.
4. Upload `routes.csv` to that volume. You do not query it in this lab.
5. Import the lab notebook `01-explore-azure-databricks.ipynb` into a workspace folder.
6. Select **Serverless** compute before running any cell.
7. Work through the notebook's Python, SQL magic commands, and Markdown exercises.

Prerequisites: a Premium workspace (created in the setup path), an Azure subscription where you have admin access, and no prior Databricks experience.

Lab instructions: https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/Instructions/Labs/01-explore-azure-databricks.html

## Knowledge check

The questions did not load when I read the module, so this section is empty. Add them here after reviewing the module assessment.

## Quick review checklist

- [ ] Know the workspace tiers and types, and that Premium is needed for SQL warehouses
- [ ] Know the compute options and SQL warehouse types
- [ ] Know Delta Lake's main features
- [ ] Know the `catalog.schema.object` hierarchy and the difference between managed and external tables
- [ ] Know what Purview adds on top of Unity Catalog
- [ ] Complete the lab exercise

## References

- Module: https://learn.microsoft.com/en-us/training/modules/explore-azure-databricks/
- Module summary: https://learn.microsoft.com/en-us/training/modules/explore-azure-databricks/08-summary
- Workspace creation and UI: https://learn.microsoft.com/en-us/training/modules/explore-azure-databricks/02-azure-databricks
- Workloads: https://learn.microsoft.com/en-us/training/modules/explore-azure-databricks/03-workloads
- Key concepts: https://learn.microsoft.com/en-us/training/modules/explore-azure-databricks/04-key-concepts
- Data governance: https://learn.microsoft.com/en-us/training/modules/explore-azure-databricks/05-data-governance-using-unity-catalog-and-microsoft-purview
- Lab 01 instructions: https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/Instructions/Labs/01-explore-azure-databricks.html
- Azure Databricks documentation: https://learn.microsoft.com/en-us/azure/databricks/
- Unity Catalog documentation: https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/
- Microsoft Purview documentation: https://learn.microsoft.com/en-us/purview/

## Back to index

[README](../README.md)
