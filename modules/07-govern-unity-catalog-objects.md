# Module 07: Govern Unity Catalog objects

Module link: https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/

Part of: [Secure and Govern Unity Catalog Objects in Azure Databricks](../paths/secure-and-govern-unity-catalog.md)

## Overview

- Units: 10 (introduction, 6 content units, exercise, knowledge check, summary)
- Level: intermediate. Role: data engineer.
- Duration: not stated on the page
- Prerequisites: basic Azure Databricks and Unity Catalog knowledge, and SQL and data engineering concepts
- Why it matters: this module covers ABAC, retention, lineage, audit logs, and sharing. Expect questions on the exact settings and limits.

## Learning objectives

1. Create and preserve table and column definitions.
2. Configure ABAC with tags and policies.
3. Apply data retention policies.
4. Set up and manage data lineage.
5. Configure audit logging.
6. Design a secure Delta Sharing strategy.

## Key points

### 1. Comments and tags (table and column definitions)

- **Comments** describe any securable object: catalogs, schemas, tables, views, and columns.
  - Add them inline with `COMMENT` in `CREATE TABLE`, or with `COMMENT ON TABLE` and `ALTER TABLE ... ALTER COLUMN ... COMMENT`.
  - Markdown works in comments.
  - AI-generated comments are suggestions. Review them. Do not use them for PII classification.
- **Permissions for comments:** ownership, or **MODIFY** and **SELECT** on the table. You also need USE CATALOG and USE SCHEMA. BROWSE lets users see comments.
- **Tags** are key-value pairs used for classification and search.

```sql
ALTER TABLE sales.customers.profiles
SET TAGS ('domain' = 'sales', 'data_classification' = 'internal');

ALTER TABLE sales.customers.profiles
ALTER COLUMN email
SET TAGS ('pii' = 'email', 'sensitivity' = 'high');

ALTER TABLE sales.customers.profiles
UNSET TAGS ('domain');
```

- **Governed tags:** an account admin defines the allowed keys and values. Only users with ASSIGN can apply them. They show a lock icon.
- **System tags:** predefined governed tags for classification, ownership, and lifecycle.
- **Limits:** 50 tags per object, 1,000 column tags per table, 255 characters for tag keys, and 1,000 characters for tag values.
- **Discovery:** Catalog Explorer (columns, sample data, details, history, insights), `SHOW`, and `DESCRIBE TABLE EXTENDED`. Search by tag needs at least BROWSE. View relationships shows an ERD for foreign keys.
- AI/BI Genie uses comments and tags, so good metadata improves Genie answers.

### 2. ABAC (attribute-based access control)

- ABAC applies policies to tagged objects. Adding the right tag brings a new table under existing policies.
- **Governed tags** are the foundation. Standard tags only need APPLY TAG. Governed tags need ASSIGN.
- Tags on a catalog or schema are inherited by everything inside. Tags on columns are not inherited.
- **Row filter policy:** a UDF returns TRUE for rows to show. The policy names the tag it matches.
- **Column mask policy:** a UDF returns the masked value. You can exclude groups with `EXCEPT`.

```sql
CREATE POLICY hide_eu_customers
ON CATALOG sales
ROW FILTER filter_non_eu
TO `analysts`
FOR TABLES
MATCH COLUMNS has_tag_value('pii', 'address') AS addr
USING COLUMNS (addr);

CREATE POLICY mask_sensitive_ssn
ON SCHEMA customers
COLUMN MASK mask_ssn
TO `all_users`
EXCEPT `compliance_team`
FOR TABLES
MATCH COLUMNS has_tag_value('pii', 'ssn') AS ssn_col
ON COLUMN ssn_col;
```

- **UDF rules:** keep them simple and deterministic. Avoid external calls. Reference only table columns. Test with at least 1 million rows. Do not call `is_account_group_member()` or `is_member()` inside the UDF. Keep access rules in the policy, not the UDF.
- **Identity:** policies are evaluated with the **session user's** identity, not the view or function owner.
- **Scope:** a catalog-level policy covers everything in the catalog. A schema-level policy covers the schema.
- **Limits:** 100 policies per catalog, 100 per schema, 50 per table, 10,000 per metastore, and 20 principals per policy (TO and EXCEPT combined).
- **Conflicts:** only one row filter per table and one mask per column. Conflicting policies block access with an error.
- **Runtime:** Databricks Runtime 16.4 or later, or serverless compute, is needed to query ABAC-protected tables. Users not covered by a policy can use any runtime.

### 3. Data retention

- Delta keeps history in two places. Know both defaults:
  - `delta.logRetentionDuration`: transaction log history, default **30 days**.
  - `delta.deletedFileRetentionDuration`: when VACUUM can delete unreferenced files, default **7 days**.
- To keep 30 days of history, set both properties. Longer retention costs more storage.

```sql
ALTER TABLE sales.customers.transactions
SET TBLPROPERTIES (
  'delta.logRetentionDuration' = 'interval 30 days',
  'delta.deletedFileRetentionDuration' = 'interval 30 days'
);
```

- **VACUUM** permanently removes unreferenced files older than the retention period. Time travel to those versions then fails.
- **Deletion vectors:** VACUUM alone does not remove rows marked as deleted. Run `REORG TABLE ... APPLY (PURGE)` first, then VACUUM.
- **Compliance deletes** (for example, "right to be forgotten"): use `DELETE`, or a `MERGE` driven by a control table. Then run maintenance to remove the files.
- Deletions reach materialized views after a refresh. Streaming tables use `skipChangeCommits` to skip upstream deletes.
- **Predictive optimization** runs VACUUM, OPTIMIZE, and ANALYZE for managed tables automatically. Enable it per schema with `ALTER SCHEMA ... ENABLE PREDICTIVE OPTIMIZATION`. It requires the Premium plan. It is on by default for accounts created on or after November 11, 2024.
- Set a longer `delta.deletedFileRetentionDuration` before enabling predictive optimization. The 7-day default may be too short for your compliance needs.
- **Azure lifecycle policies** move blobs to cool or archive tiers. They work at the storage layer. Delta retention works at the table layer. Use both.
- For archive tiers: set `delta.timeUntilArchived` (Public Preview, Runtime 13.3 LTS or later, Azure Archive tier only). Filter archived data out with views, and limit OPTIMIZE and VACUUM with predicates.

### 4. Lineage

- **Runtime lineage** is captured automatically, in all languages, down to the column level. It covers notebooks, jobs, and dashboards.
- Lineage is captured only on **Unity Catalog compute** (shared, single user, or serverless) and SQL warehouses with Unity Catalog. It is not captured for non-UC clusters or for queries that use file paths.
- Lineage is shared across all workspaces attached to the metastore.
- **Retention:** Catalog Explorer and the lineage API keep data indefinitely for events after September 1, 2024. The lineage system tables keep one year.
- View it in Catalog Explorer: Lineage tab, then See Lineage Graph. Select a column to see its upstream and downstream columns. Jobs and dashboards appear under the Lineage tab.
- **Ownership:** every object has an owner. Only the owner or a metastore admin can transfer ownership. The old owner loses owner rights.
- To see lineage, users need BROWSE on the parent catalog. Objects they cannot access appear as masked nodes.
- **Table history:** `DESCRIBE HISTORY` shows version, timestamp, operation, and user.
- **System tables:** `system.access.table_lineage` and `system.access.column_lineage`.
- **Limits:** lineage is lost for renamed objects. Path-based tables show only the path. Global temporary views and `system.information_schema` are not captured. Some UDFs hide column mappings.

### 5. Audit logging

- Audit logging is configured at the **account level** and covers all workspaces.
- The audit table is `system.access.audit`. It is in **Public Preview**. An account admin must enable system tables first.
- Key columns: `event_time`, `event_date`, `user_identity`, `workspace_id`, `service_name`, `action_name`, `request_params`, `response`, and `source_ip_address`.
- Account-level events use workspace ID `0`.
- **Retention in the system table:** up to one year. Plan longer retention separately.
- Services tracked include Unity Catalog, notebooks, clusters, jobs, Databricks SQL, accounts, billable usage, and Delta Sharing.
- Example queries in the unit cover: who accessed a table, permission changes (`updatePermissions`), and failed logins.
- **Verbose audit logs** (workspace admin setting) add notebook, job, and SQL command text. Command text can contain sensitive data.
- **Log delivery:** Azure diagnostic settings can send logs to Log Analytics (about 15 minutes of latency), Storage (archival), or Event Hubs (streaming to a SIEM).
- The system table is for ad hoc investigation. External delivery is for continuous monitoring.

### 6. Delta Sharing (OpenSharing)

- The unit is titled "Design Secure OpenSharing Strategy". Delta Sharing was renamed OpenSharing. Both names appear in the exam material, so learn both.
- Sharing gives live access to data without copying it, and keeps Unity Catalog controls in place.

**Choose a protocol**

| Factor | Databricks-to-Databricks | Open sharing |
|---|---|---|
| Recipient | Databricks with Unity Catalog | Any platform |
| Authentication | Platform-managed | Token or OIDC |
| Shareable assets | Tables, views, volumes, notebook files, models | Tables only |
| Token management | Not needed | Needed |

- Prefer Databricks-to-Databricks when the recipient has a UC workspace. The recipient gives you a sharing identifier (cloud, region, and metastore UUID).

**Authentication and access**
- Open sharing uses bearer tokens or OIDC federation. Tokens last up to one year. OIDC removes long-lived credentials.
- **IP access lists** restrict open-sharing recipients. Each recipient allows up to 100 addresses or ranges. Other requests are denied and logged.
- Use delegated privileges (`CREATE SHARE`, `CREATE RECIPIENT`, `USE SHARE`, `USE RECIPIENT`, `SET SHARE PERMISSION`) for data stewards, not metastore admin rights.

**Create and manage shares**

```sql
CREATE SHARE IF NOT EXISTS partner_analytics COMMENT 'Sales analytics data for partner integration';

ALTER SHARE partner_analytics ADD TABLE catalog.schema.sales_summary;
ALTER SHARE partner_analytics ADD TABLE catalog.schema.transactions WITH HISTORY;
ALTER SHARE partner_analytics ADD SCHEMA catalog.analytics_schema;  -- includes future tables
ALTER SHARE partner_analytics ADD VOLUME catalog.schema.documentation;  -- Databricks-to-Databricks only
ALTER SHARE partner_analytics ADD VIEW catalog.schema.aggregated_sales;

CREATE RECIPIENT IF NOT EXISTS contoso_analytics
USING ID 'azure:westeurope:a1b2c3d4-e5f6-7890-abcd-ef1234567890'
COMMENT 'Contoso data analytics team';

CREATE RECIPIENT IF NOT EXISTS external_partner COMMENT 'External analytics platform integration';
DESCRIBE RECIPIENT external_partner;  -- returns the activation link

GRANT SELECT ON SHARE partner_analytics TO RECIPIENT contoso_analytics;
SHOW GRANT ON SHARE partner_analytics;
REVOKE SELECT ON SHARE partner_analytics FROM RECIPIENT former_partner;
DROP RECIPIENT former_partner;
```

- A schema share includes future tables. Use it carefully.
- The activation link downloads the credential file only once. Send it over a secure channel.
- Notebook files are added in Catalog Explorer, not with SQL.

**Fine-grained sharing with views**
- Set recipient properties: `ALTER RECIPIENT ... SET PROPERTIES`.
- Use `CURRENT_RECIPIENT()` in a view to filter rows or mask columns per recipient.
- Partition filtering can use `CURRENT_RECIPIENT()` in `ALTER SHARE ... PARTITION`.
- Providers cannot query these views directly. Test by sharing with yourself.

**Monitoring**
- Sharing events are in `system.access.audit`. They are not in Azure diagnostic settings.
- Track configuration changes (`createShare`, `createRecipient`, `updateShare`, `updateSharePermissions`) and data access (`deltaSharingQueriedTable`, `generateTemporaryTableCredential`).
- Review monthly: recipients, share contents, and token expiry.

**Security practices**
- Document each sharing agreement first.
- Share only what is needed.
- Rotate tokens early. Set the old token's expiry to `0` to end it right away. A recipient can hold at most two tokens.
- Review when a partner's organization changes.

## Exam focus

- Default retention: log 30 days, deleted files 7 days. VACUUM deletes after the file retention period.
- Deletion vectors need REORG ... PURGE before VACUUM.
- Governed tags need ASSIGN. Standard tags need APPLY TAG. Tags on columns are not inherited.
- ABAC: one row filter per table, one mask per column. Runtime 16.4 or later, or serverless.
- ABAC UDFs run with the session user's identity. Do not call `is_account_group_member()` inside them.
- Lineage needs Unity Catalog compute. Lineage system tables keep one year. Catalog Explorer keeps it indefinitely (after September 1, 2024).
- Audit logs live in `system.access.audit`, for one year. The table is in Public Preview.
- Databricks-to-Databricks sharing supports more asset types. Open sharing needs tokens or OIDC, and IP access lists.
- Sharing audit events are in the audit system table.

## Knowledge check

The module assessment did not load when I read the page, so I have no questions to record. Add them after you take the assessment.

## Quick review checklist

- [ ] Know the default retention settings and how VACUUM uses them
- [ ] Explain governed tags vs. standard tags
- [ ] Write an ABAC row filter policy and a column mask policy
- [ ] Explain lineage capture requirements and retention
- [ ] Find a table's access history in the audit system table
- [ ] Explain Databricks-to-Databricks vs. open sharing
- [ ] Complete the lab exercise

## Notes on claims

- Retention defaults: the unit says 30 days for log retention and 7 days for deleted-file retention. Check these against current Databricks documentation before the exam.

## References

- Module: https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/
- Table definitions: https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/2-preserve-table-column-definitions
- ABAC: https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/3-configure-attribute-based-access-control-tags-policies
- Data retention: https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/4-apply-data-retention-policies
- Lineage: https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/5-manage-data-lineage-tracking
- Audit logging: https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/6-configure-audit-logging
- Delta Sharing (OpenSharing): https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/7-secure-delta-sharing-strategy
- Summary: https://learn.microsoft.com/en-us/training/modules/govern-unity-catalog-objects/10-summary

## Back to index

[README](../README.md)
