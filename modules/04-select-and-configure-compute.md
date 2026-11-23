# Module 04: Select and configure compute in Azure Databricks

Module link: https://learn.microsoft.com/en-us/training/modules/select-and-configure-compute/

Part of: [Set up and Configure an Azure Databricks Environment](../paths/set-up-and-configure-environment.md)

## Overview

- Units: 9 (introduction, 5 content units, exercise, knowledge check, summary)
- Prerequisites: basic Azure Databricks concepts, Unity Catalog and workspace management, and Spark and cluster basics
- Why it matters: compute choice drives cost, speed, and what your code can do. This module is heavily tested in practice.

## Learning objectives

1. Choose the right compute type for each workload.
2. Set node types, autoscaling, and termination.
3. Enable Photon and pick a Databricks Runtime version.
4. Set compute permissions and dedicated group access.
5. Install libraries at cluster and notebook scope.

## Key points

### 1. Choose a compute type

**Serverless compute**
- Fully managed. Runs in Databricks' Azure subscription. Starts in about 2 to 6 seconds.
- **Requires Unity Catalog.** Covers notebooks, jobs, Lakeflow pipelines, and SQL warehouses.
- Uses a versionless runtime, so upgrades happen automatically.
- Limits in notebooks: no Scala, no R, no JAR libraries, no RDD APIs. Custom Spark configuration is restricted.
- Performance modes for jobs and pipelines: **performance-optimized** (default, fast start) and **standard** (cheaper, can cut DBU use by up to 70%, starts in about 4 to 6 minutes). Notebooks only support performance mode.

**Classic compute**
- Runs in your own Azure subscription. Starts in about 3 to 7 minutes.
- You choose the runtime version. Runtime upgrades are manual.
- **Access modes:**
  - **Standard:** shared by multiple users, isolated by Lakeguard. Good for collaboration.
  - **Dedicated:** assigned to one user or group. Needed for RDD APIs, GPU workloads, R, and custom containers.
  - **Auto** (default in the UI): picks Standard, but picks Dedicated for ML runtimes or runtimes below 14.3.
  - **No isolation shared** (legacy): avoid it.
- **Cluster modes:** multi-node (driver plus workers, for production scale) or single-node (driver only, cannot scale out, good for small data and single-machine ML).

**SQL warehouses**
- **Serverless:** starts in 2 to 6 seconds. The default choice for SQL and BI.
- **Pro:** Photon and Predictive IO, but no Intelligent Workload Management. Starts in about 4 minutes. Use it for custom networking.
- **Classic:** Photon only, about 4 minutes to start, basic autoscaling.

**Instance pools**
- Keep idle VMs ready so clusters start in under a minute in many cases.
- You pay for idle VMs. Set a minimum number of idle instances and a maximum capacity.
- Less important now that serverless exists.

**Job compute**
- Clusters for automated jobs. They end when the job finishes.
- **Serverless job compute:** fastest and easiest.
- **Classic job compute:** more control, including spot instances (up to about 90% cheaper). Azure can reclaim spot VMs with 30 seconds' notice.
- The **Job Compute policy** gives a template for production job clusters.

**Decision rule from the module:** start with serverless. Move to classic only when you hit a limitation serverless does not support. Classic dedicated is needed for RDD, R, or GPU work. Pro SQL warehouses are for custom networking.

### 2. Performance settings

- **Resources:** total executor cores set parallelism. Total executor memory sets how much data stays in memory. Local storage holds shuffle and cache data.
- **Node types:**
  - Memory-optimized (E-series): large joins and aggregations.
  - Compute-optimized (F-series): CPU-heavy ETL.
  - Storage-optimized (L-series): caching and heavy I/O.
  - GPU-accelerated (NC and ND series): deep learning. Requires Databricks Runtime ML.
- Fewer, larger workers reduce shuffle traffic. More, smaller workers suit highly parallel batch jobs.
- **Flexible node types:** when a VM type is out of capacity (`CLOUD_PROVIDER_RESOURCE_STOCKOUT`), the cluster can fall back to similar types. Admins enable it in workspace Compute settings.
- **Autoscaling:** set min and max workers. Optimized autoscaling is the default. Use it when load varies. Use fixed size when load is steady.
- **Termination:** set an inactivity timeout. A 45-minute timeout is a good default for interactive clusters. Job compute ends after each run.
- **Spot instances:** use them for workers only. Always use on-demand for the driver.
- Check the Spark UI for shuffle spill to memory and disk.

### 3. Feature settings

- **Photon:** a native query engine for SQL and DataFrame work. Helps joins, aggregations, and scans on large data. Enabled by default on Runtime 9.1 LTS and later. Not supported on GPU clusters.
- **Runtime choice:** use the latest runtime for interactive development. Use an LTS runtime for production jobs.
- **Runtime ML:** pre-installed ML libraries and GPU drivers. Start here for ML work.
- The Spark version is tied to the runtime you choose.

### 4. Library installation

- **Compute-scoped libraries:** available to every notebook on the cluster. Reinstalled at each cluster start. Needs **CAN MANAGE** on the cluster.
- **Notebook-scoped libraries:** only for one notebook session. Use them to avoid conflicts.
- Supported types: Python wheels, Java JARs, and R packages.

Installation methods:
1. **Package repositories:** PyPI (Python), Maven (Java and Scala), CRAN (R). Pin versions in production. Without a version, you get the latest release.
2. **Workspace files or Unity Catalog volumes:** for internal or pinned packages. Workspace files have a 500 MB limit. Volumes are governed by Unity Catalog. `requirements.txt` works from these locations on Runtime 15.0 and later.
3. **Init scripts:** run at cluster startup. Use them only for system-level setup. Databricks does not recommend them for libraries. A non-zero exit code stops the cluster from starting. Store them in volumes on Runtime 13.3 LTS and later.

**Standard access mode and the allowlist:**
- A metastore admin must add Maven coordinates, JAR paths, and init scripts to the allowlist.
- Allowlisting uses prefix matching. Add a trailing slash to avoid matching too much.
- The allowlist permits use. The installer still needs data access, such as **READ VOLUME**.

Use package repositories for public packages. Use files or volumes for internal packages. Use init scripts last.

### 5. Access and permissions

**Compute permission levels**

| Level | What it allows |
|---|---|
| NO PERMISSIONS | Default. Users cannot see or attach the compute. |
| CAN ATTACH TO | Attach notebooks, view the Spark UI and metrics. |
| CAN RESTART | Everything above, plus start, stop, and restart. |
| CAN MANAGE | Full control: edit config, install libraries, change permissions. |

- Set permissions in the workspace UI (Compute, then Permissions). Changes apply immediately.
- Prefer groups over individual users.
- By default, only CAN MANAGE users can view driver logs, because logs can expose secrets. You can change this with `spark.databricks.acl.needAdminPermissionToViewLogs`, but that is a security risk.

**Entitlements**
- By default, only workspace admins can create large clusters.
- The **Allow cluster creation** entitlement lets a user or group create clusters without being an admin.

**Dedicated group access**
- Requires Unity Catalog, Runtime 15.4 or later, and CAN MANAGE on a workspace folder for the group.
- Choose **Advanced**, then **Dedicated** access mode, then the group.
- Objects created on a group cluster are owned by the group.
- Folder pattern: `/Workspace/Groups/<groupName>`.

**Best practices:** least privilege, group-based access, restricted CAN MANAGE for production, secret scopes instead of hardcoded secrets, and regular audits through the `system.access.audit` table.

## Exercise: Select and configure compute (45 minutes)

Goal: create an all-purpose cluster, install a library at cluster scope and at notebook scope, and use the Faker library with PySpark to create and analyze synthetic patient admission records.

Prerequisite: an Azure subscription where you have admin access.

The lab instructions are hosted here: https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/Instructions/Labs/02-select-and-configure-compute.html

The lab page did not load in full when I read the module, so I have not recorded the exact steps. Add them after you open the lab.

## Exam focus

- Serverless requires Unity Catalog. It does not support RDD, R, Scala notebooks, or JAR libraries in notebooks.
- Dedicated access mode is needed for RDD, R, GPU, and ML runtimes. Auto picks Dedicated for ML runtimes and for runtimes below 14.3.
- Standard performance mode is cheaper but starts slower. Performance mode is the default.
- Photon is off for GPU clusters.
- Spot instances on workers only, never on the driver.
- The four compute permission levels and what each adds.
- Driver logs are restricted to CAN MANAGE by default.
- Dedicated group access needs Runtime 15.4 or later and Unity Catalog.
- Standard access mode needs allowlist approval for JARs and Maven coordinates.
- Init scripts stop the cluster if they exit with a non-zero code.

## Knowledge check

The module assessment did not load when I read the page, so I have no questions to record. Add them after you take the assessment.

## Quick review checklist

- [ ] Compare serverless, classic, SQL warehouse, and job compute
- [ ] Know the access modes and when to use Dedicated
- [ ] Know the autoscaling and termination options
- [ ] Know the four permission levels
- [ ] Know the three library installation methods and their limits
- [ ] Know the allowlist rules for standard access mode
- [ ] Complete the lab exercise

## Notes on claims

- The summary says Photon can double query performance. That is Microsoft's stated expectation, not a guaranteed result. Test it on your own workloads.

## References

- Module: https://learn.microsoft.com/en-us/training/modules/select-and-configure-compute/
- Introduction: https://learn.microsoft.com/en-us/training/modules/select-and-configure-compute/1-introduction
- Choose a compute type: https://learn.microsoft.com/en-us/training/modules/select-and-configure-compute/2-choose-appropriate-compute-type
- Configure compute performance: https://learn.microsoft.com/en-us/training/modules/select-and-configure-compute/3-configure-compute-performance
- Configure compute features: https://learn.microsoft.com/en-us/training/modules/select-and-configure-compute/4-configure-compute-feature-settings
- Install libraries: https://learn.microsoft.com/en-us/training/modules/select-and-configure-compute/5-install-libraries-for-compute
- Configure compute access: https://learn.microsoft.com/en-us/training/modules/select-and-configure-compute/6-configure-compute-access-perms
- Exercise: https://learn.microsoft.com/en-us/training/modules/select-and-configure-compute/7-exercise-select-configure-compute
- Summary: https://learn.microsoft.com/en-us/training/modules/select-and-configure-compute/9-summary
- Lab 02 instructions: https://microsoftlearning.github.io/DP-750T00-Implement-Data-Engineering-Solutions-using-Azure-Databricks/Instructions/Labs/02-select-and-configure-compute.html

## Back to index

[README](../README.md)
