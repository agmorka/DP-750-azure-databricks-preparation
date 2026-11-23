# Module 03: Understand Azure Databricks integrations

Module link: https://learn.microsoft.com/en-us/training/modules/understand-azure-databricks-integrations/

Part of: [Set up and Configure an Azure Databricks Environment](../paths/set-up-and-configure-environment.md)

## Overview

- Units: 10 (introduction, 7 integration units, knowledge check, summary)
- Prerequisites: basic Azure Databricks concepts, Unity Catalog and governance, and Delta Lake tables and SQL warehouses
- Why it matters: for the exam, you must know which integration uses which authentication method and what it limits

## Learning objectives

1. Use Microsoft Fabric for two-way data access.
2. Describe how Power BI connects to Azure Databricks.
3. Use the Databricks extension for VS Code.
4. Describe Power Platform integration.
5. Describe Copilot Studio integration.
6. Describe Purview governance integration.
7. Connect Microsoft Foundry agents to Genie spaces.

## Key points

### 1. Microsoft Fabric

- The integration works both ways. Fabric can **read** Unity Catalog data, and Azure Databricks can **write** to OneLake.
- **Fabric reads Unity Catalog:** a *Mirrored Azure Databricks Catalog* item creates OneLake shortcuts to Delta tables. No data is copied. Shortcut credentials refresh every hour.
- Security risk: Fabric checks access with the identity of the user who **configured the connection**. Unity Catalog policies do **not** apply to downstream Fabric users.
- The person configuring the connection needs **EXTERNAL USE SCHEMA** on the schemas.
- Limits: no views, materialized views, streaming tables, or Delta Sharing catalogs. Only Delta tables. Private endpoints and IP access lists are not supported. Lineage does not track Fabric operations.
- Mitigations: Automatic Identity Management with a OneLake Data Access Role, or Power BI DirectQuery for user-level row filters and masks.
- **Azure Databricks writes to OneLake:** use the ABFS driver with the OneLake endpoint (`abfss://<workspace>@onelake.dfs.fabric.microsoft.com/<lakehouse>.lakehouse/Files/`).
  - Authentication: Entra credential passthrough (premium workspace, not supported on serverless) or a service principal (works on serverless).
- Choosing: expose existing Databricks data to Fabric, or publish curated datasets to Fabric.

### 2. Power BI

Three connection methods:

1. **Power BI Desktop** connects to clusters or SQL warehouses. Use SQL warehouses with DirectQuery.
   - Setup: Partner Connect (fastest) or manual, using the Server Hostname and HTTP Path.
   - Modes: **Import** (fast, needs refreshes) or **DirectQuery** (live, needs running compute).
   - New connections use the ADBC driver by default since February 2026. Older ones use ODBC. (Verify this date before the exam.)
   - Authentication: personal access tokens (single user), Entra ID (SSO, per-user Unity Catalog checks), or service principals (automation).
   - Power BI Desktop needs Windows.
2. **Publish from Azure Databricks** to the Power BI service: in Catalog Explorer, choose **Publish to Power BI workspace**.
   - Needs Unity Catalog and a Premium Power BI license or Fabric capacity. The XMLA endpoint must be enabled with read and write.
   - Column comments carry over as descriptions. Foreign keys carry over, but only one active relationship is allowed between two tables.
   - Authentication: OAuth (per user, recommended for governance) or machine-to-machine OAuth for refreshes.
3. **Power BI tasks** in Databricks workflows: publish and refresh semantic models after upstream tasks finish. Needs a Unity Catalog connection to Power BI.

Power BI gateways are used when private network access is required.

Governance table to remember:
- Entra SSO: per-user permissions and per-user audit.
- Import or no SSO: everyone sees the same data.
- Service principal: permissions of the principal only.
- Personal access token: development only.

### 3. VS Code (Databricks extension)

- **Develop locally, run remotely:** run local `.py` files or notebooks on clusters or serverless compute.
- **Databricks Connect:** links the local Python debugger to remote compute. Set breakpoints and step through code.
- Run `pytest` tests on Databricks compute.
- **Declarative Automation Bundles** (formerly Databricks Asset Bundles): define Lakeflow Jobs, Lakeflow pipelines, and MLOps Stacks in YAML. Deploy to dev, staging, or prod from VS Code. Bundles live in Git.
- **One-way sync:** local edits upload to the workspace. Edits made in the workspace do not sync back. Use it for prototyping. Use bundles for production.

### 4. Power Platform

- The **Azure Databricks connector** gives Power Apps and Power Automate access to Unity Catalog data, and governance still applies.
- Prerequisites: a premium Power Apps license, an Entra ID account, and a SQL warehouse. Use the Server Hostname and HTTP Path.
- Authentication: **OAuth** (per user) or a **service principal**.
- Private networks: VNet peering or allow the AzureConnectors IP ranges.
- **Power Apps:**
  - Canvas apps use a direct connection. They support OAuth passthrough.
  - Model-driven apps use Dataverse virtual tables. They do **not** support OAuth passthrough.
  - Prefer direct connections for speed.
- **Power Automate:** run SQL, start and monitor jobs, cancel statements, and get run output. The connector has throttling limits, so batch the work.
- Not supported in US Government or China Cloud.
- Use the Power Platform **Business data policy** to control which connectors can share data with it.

### 5. Copilot Studio

- The Azure Databricks connector supports two patterns:
  - **Knowledge sources:** the agent queries tables through a SQL warehouse using natural language. Works best with clear column names and curated views.
  - **Genie spaces as tools:** the agent calls Genie over MCP. Needs the **Managed MCP Servers** preview. This is in Public Preview.
- Authentication: OAuth (per-user Unity Catalog checks, same Entra tenant required) or a service principal (no per-user row security).
- Needs a serverless or pro SQL warehouse. Classic warehouses are not supported for Genie.
- Power Platform throttling: 100 API calls per 60 seconds. (Numeric limit: verify before the exam.)
- Not supported in government clouds.

### 6. Microsoft Purview

- Purview reads **metadata** only. It does not read the data.
- Purview scans Unity Catalog and the Hive metastore.
  - **Unity Catalog:** uses a cloud-native Azure integration runtime through SQL warehouses. Supports **incremental** scans and captures **runtime lineage** from system tables.
  - **Hive metastore:** needs a **self-hosted integration runtime**. Each scan is a full scan. It captures static lineage from view definitions.
- Lineage limits: only transformations logged by Unity Catalog appear. External tools such as Azure Data Factory do not appear. Complex column lineage may be incomplete.
- Purview can scan both during a migration.

### 7. Microsoft Foundry

- Foundry agents connect to **Genie spaces** through the **Azure Databricks Genie connector**. The protocol is **MCP**.
- Foundry is the MCP client. Databricks exposes Genie spaces through a managed MCP server.
- Agents do not write SQL. Genie generates the SQL and enforces Unity Catalog permissions.
- Authentication: **OAuth identity passthrough**. Managed OAuth (Entra ID) is recommended.
- Requires the **Managed MCP Servers** preview.
- Limits: **5 questions per minute** (numeric limit: verify before the exam). No conversation history between MCP tool calls.
- Costs: Foundry usage plus serverless SQL compute for Genie queries.
- Status: Public Preview.

## Exam focus

- Fabric mirror vs. write to OneLake. Which one bypasses Unity Catalog for downstream users?
- Power BI: the three connection methods and the three authentication methods. Know which one enforces per-user permissions.
- Power Apps: canvas apps support OAuth passthrough. Model-driven apps (virtual tables) do not.
- Copilot Studio and Foundry: the difference between knowledge sources (SQL) and Genie (MCP).
- Purview: Unity Catalog uses a cloud-native runtime. Hive metastore needs a self-hosted runtime.
- Preview features: Genie integrations, Managed MCP Servers.
- Government clouds are not supported for Power Platform or Copilot Studio.

## Knowledge check

Assessment: https://learn.microsoft.com/en-us/training/modules/understand-azure-databricks-integrations/9-knowledge-check

The module assessment did not load when I read the page, so I have no questions to record. Add them after you take the assessment.

## Quick review checklist

- [ ] Explain Fabric mirroring vs. writing to OneLake, with the security trade-off
- [ ] Name the three Power BI connection methods and their authentication options
- [ ] Explain the difference between canvas and model-driven Power Apps
- [ ] Explain knowledge sources vs. Genie in Copilot Studio
- [ ] Explain how Purview scans Unity Catalog vs. Hive metastore
- [ ] Explain how Foundry connects to Genie over MCP

## References

- Module: https://learn.microsoft.com/en-us/training/modules/understand-azure-databricks-integrations/
- Knowledge check: https://learn.microsoft.com/en-us/training/modules/understand-azure-databricks-integrations/9-knowledge-check
- Introduction: https://learn.microsoft.com/en-us/training/modules/understand-azure-databricks-integrations/1-introduction
- Microsoft Fabric: https://learn.microsoft.com/en-us/training/modules/understand-azure-databricks-integrations/2-integration-databricks-microsoft-fabric
- Power BI: https://learn.microsoft.com/en-us/training/modules/understand-azure-databricks-integrations/3-integration-databricks-power-bi
- VS Code: https://learn.microsoft.com/en-us/training/modules/understand-azure-databricks-integrations/4-integration-databricks-vscode
- Power Platform: https://learn.microsoft.com/en-us/training/modules/understand-azure-databricks-integrations/5-integration-databricks-power-platform
- Copilot Studio: https://learn.microsoft.com/en-us/training/modules/understand-azure-databricks-integrations/6-integration-databricks-copilot-studio
- Microsoft Purview: https://learn.microsoft.com/en-us/training/modules/understand-azure-databricks-integrations/7-integration-databricks-microsoft-purview
- Microsoft Foundry: https://learn.microsoft.com/en-us/training/modules/understand-azure-databricks-integrations/8-integration-databricks-microsoft-foundry
- Summary: https://learn.microsoft.com/en-us/training/modules/understand-azure-databricks-integrations/10-summary

## Back to index

[README](../README.md)
