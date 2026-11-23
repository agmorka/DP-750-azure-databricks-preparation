# Module 06: Secure Unity Catalog objects

Module link: https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/

Part of: [Secure and Govern Unity Catalog Objects in Azure Databricks](../paths/secure-and-govern-unity-catalog.md)

## Overview

- Units: 11 (introduction, 8 content units, exercise, knowledge check, summary)
- Duration: not stated on the page
- Prerequisites: basic Azure Databricks and Unity Catalog knowledge, SQL, and Microsoft Entra ID and Azure security basics
- Why it matters: this module covers the permission model, row and column security, and how credentials are handled. These topics come up often on the exam.

## Learning objectives

1. Explain the query lifecycle in Unity Catalog and where security is checked.
2. Use inherited and explicit permissions.
3. Apply row filters and column masks.
4. Read secrets from Azure Key Vault in Databricks.
5. Authenticate with service principals.
6. Configure managed identities for storage access.

## Key points

### 1. Query lifecycle

1. A principal (user or service identity) submits a query from a cluster, job, SQL warehouse, or BI tool.
2. Unity Catalog checks the principal's permissions on every object. It logs the request in the audit log. Denied requests are blocked.
3. Unity Catalog uses the cloud credential linked to each object.
4. Unity Catalog issues a **scoped, temporary access token**. Compute never gets long-lived credentials.
5. Compute reads directly from ADLS Gen2 with the token.
6. Row filters and column masks are applied during execution.
7. The filtered result is returned.

- Use a managed identity with an access connector for ADLS. Service principals alone cannot reach storage behind network rules.
- If the storage firewall blocks the access connector or managed identity, access fails even with a valid token.
- **Hive metastore** (legacy, `hive_metastore`) works differently. It uses table ACLs and admin-managed credentials, mount points, or instance profiles. It has weaker governance and auditing. Unity Catalog automates credentials and applies policies consistently.

### 2. Access control: privileges and inheritance

- A grant says **who** (a principal) can do **what** (a privilege) on **which object**.
- Principals can be users, service principals, or groups. **Use groups.** Names with special characters need backticks.
- The built-in principal `account users` covers all users in the account. The workspace-local `users` group cannot receive grants.
- Unity Catalog uses Microsoft Entra ID users and groups. Workspace-local users and groups are not supported.
- Metastores are bound to one Azure region. A workspace can link to only one metastore at a time.

**Explicit model (most secure, more work):** to query a table, a principal needs
1. **USE CATALOG** on the catalog
2. **USE SCHEMA** on the schema
3. **SELECT** on the table

**Inherited from schema:** grant USE CATALOG, USE SCHEMA, and SELECT on the schema. The grantee can read all current and future tables in it.

**Inherited from catalog:** grant USE CATALOG and ALL PRIVILEGES (or SELECT) on the catalog. This is the most permissive option.

**Creating tables:** explicit needs USE CATALOG, USE SCHEMA, and CREATE TABLE on the schema. Catalog-level inheritance covers every schema in the catalog, including future ones.

```sql
GRANT SELECT ON TABLE sales TO `analysts`;
GRANT USE SCHEMA ON SCHEMA finance TO `analysts`;
GRANT USE CATALOG ON CATALOG corpdata TO `analysts`;
GRANT CREATE TABLE ON SCHEMA finance TO `analysts`;
REVOKE SELECT ON TABLE sales FROM `analysts`;
SHOW GRANTS ON TABLE sales.reporting.monthly_revenue;
```

- `SHOW GRANTS` on a table may not show inherited grants. Check each level.
- Privileges are checked along the hierarchy: metastore, catalog, schema, then object.
- Unity Catalog rejects privileges that don't apply to an object type. For example, MODIFY on a view.
- Other securable objects include functions, volumes, and external locations.

**Catalog Explorer:** Permissions tab on the table. It shows inherited privileges too.

### 3. Row filtering and column masking (row and column security)

- **Column mask:** a SQL function that returns the real value or a redacted value for each user. Attach it with `SET MASK`.
- **Row filter:** a SQL function that returns `true` for rows a user can see. Attach it with `SET ROW FILTER`.
- Functions are securable objects. Protect them, because weakening them weakens the whole model.
- Both use `is_account_group_member()` to check group membership.

```sql
CREATE OR REPLACE FUNCTION phone_mask(c_phone STRING)
  RETURN CASE WHEN is_account_group_member('metastore_admins')
    THEN c_phone
    ELSE 'REDACTED PHONE NUMBER'
  END;

ALTER TABLE customers
  ALTER COLUMN c_phone
  SET MASK phone_mask;

CREATE OR REPLACE FUNCTION nation_filter(c_nationkey INT)
  RETURN IF(is_account_group_member('admin'), true, c_nationkey = 21);

ALTER TABLE customers
SET ROW FILTER nation_filter ON (c_nationkey);
```

**Dynamic views** (the older approach):
- A view applies masks with CASE and filters with WHERE. Each audience usually needs its own view.
- Grant access with `GRANT SELECT ON VIEW ...`. The base table stays unchanged.
- Views are read-only.

**Choosing:**
- **Row and column security** (the approach taught in detail here): one table plus a few functions. Enforced at the source. Fewer objects to manage.
- **Dynamic views:** good when you need to combine tables, or when consumers should not see the structure underneath.
- The module says **attribute-based access control (ABAC)** is now the recommended approach for most cases. That conflicts with the "preferred" label on row and column security above. ABAC is covered in module 7. Know both, and check the current docs.

Common pitfalls: duplicating tables per audience, too many near-identical views, complex business logic in masks, unsecured functions, and over-masking.

**Note:** The sample code in the source uses different group names (`metastore_admins`, `admin`, `admins`) and table names (`customers`, `customers_new`). Check names before reusing the examples.

### 4. Azure Key Vault secrets

- Azure Key Vault is the preferred store for secrets. Databricks-backed scopes are the alternative.
- A **secret scope** is a named group of secrets. Permissions are set per scope.
- **Key Vault-backed scope:** read-only in Databricks. Manage the secrets in Key Vault. Restart clusters after you change a secret, because clusters cache values at startup.
- **Prerequisites for the scope:**
  - The Key Vault must use the **Vault access policy** permission model. Azure RBAC is not supported. The module says Key Vault API version 2026-02-01 and later default to RBAC. I have not verified this, so check it before you create the scope.
  - Allow access from the networks you need, and allow trusted Microsoft services.
  - Get the DNS name and resource ID of the Key Vault.
- Create the scope at `https://<workspace>#secrets/createScope`. The URL is case-sensitive.
- Your account needs Key Vault Contributor, Contributor, or Owner role. Databricks gets Get and List permissions on the vault.
- Restricting MANAGE to the creator requires the Premium plan.
- Key Vault permissions apply to every secret in the vault. Use separate vaults to split access.

**Reading secrets**
```python
username = dbutils.secrets.get(scope="jdbc", key="username")
dbutils.secrets.list("jdbc")  # lists keys, not values
```
```sql
SELECT secret('jdbc', 'username');
-- try_secret() returns NULL when a secret is missing (unverified: confirm the function exists in the docs)
```

- Printed secret values show as `[REDACTED]`. Redaction is best-effort only. Anyone who can run code on the cluster can read the value.
- Secrets in Spark config: `spark.storage.key {{secrets/storage/account-key}}` (Public Preview).
- Secrets in environment variables are visible to anyone with CAN ATTACH TO. Use them only when everyone on the cluster should have access.

**Scope permissions**

| Permission | What it allows |
|---|---|
| READ | Read secret values |
| WRITE | Add or update secrets (Databricks-backed scopes only) |
| MANAGE | Read, and change permissions |

- Set permissions with the Databricks CLI: `databricks secrets put-acl <scope> <principal> READ`.
- Key Vault-backed scopes use Databricks ACLs only. WRITE does not apply.

### 5. Service principals

- A service principal is an identity for applications. It has an application (client) ID, a directory (tenant) ID, and a client secret.
- **Microsoft recommends managed identities** for Unity Catalog storage credentials when possible. Use service principals when managed identities are not an option or you need explicit control over credentials.
- With Unity Catalog, an admin creates a **storage credential** and an **external location**. Data engineers use them without seeing the credentials.
- Notebook-level config (for storage not in Unity Catalog) uses Spark `fs.azure.account.*` OAuth settings with a client secret read from a secret scope.
- Secrets must be rotated. Expired secrets break access.
- Give each service principal access only to the storage it needs.

### 6. Managed identities

- An **access connector for Azure Databricks** links a managed identity to your account. It can use a **system-assigned** identity (deleted with the connector) or **user-assigned** identities (persist and can be shared).
- Two uses: the **metastore root storage** (managed tables) and **external storage** (external tables and files).
- Roles on the storage account:
  - **Storage Blob Data Contributor** (required). Assign at account or container level.
  - **Storage Queue Data Contributor** (recommended). Enables file event notifications.
- Create a **storage credential** in Catalog Explorer, using the Azure Managed Identity type. Creating one requires **CREATE STORAGE CREDENTIAL** on the metastore.
- An **external location** joins a storage credential and a path (`abfss://...`). Several locations can share one credential. A credential can be read-only.
- **Workspace binding** limits external locations and credentials to certain workspaces.
- Chain: access connector → managed identity → storage credential → external location → table or files.
- Managed identities handle authentication only. Network access still needs a private endpoint or service endpoint.

## Exam focus

- The query lifecycle: check permissions, then issue a scoped token. Compute never gets long-lived credentials.
- Explicit vs. inherited privileges, and the three privileges needed to query a table explicitly.
- Row filters (`SET ROW FILTER`) and column masks (`SET MASK`). Row and column security is preferred over dynamic views.
- Key Vault-backed scopes: Vault access policy model, read-only in Databricks, restart clusters after changes.
- `dbutils.secrets.get`, `secret()`, `try_secret()`, and the `[REDACTED]` output.
- Managed identity chain and the required roles.
- Service principal vs. managed identity: managed identities are preferred.

## Knowledge check

Assessment: https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/10-knowledge-check

The module assessment did not load when I read the page, so I have no questions to record. Add them after you take the assessment.

## Quick review checklist

- [ ] Explain the query lifecycle in order
- [ ] Write the three privileges needed to query a table
- [ ] Write a column mask and a row filter, and attach them
- [ ] Explain when to use row and column security vs. dynamic views
- [ ] Create a Key Vault-backed scope from memory of the steps
- [ ] Explain the managed identity chain and its roles
- [ ] Complete the lab exercise

## References

- Module: https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/
- Knowledge check: https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/10-knowledge-check
- Query lifecycle: https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/2-understand-query-lifecycle
- Access control strategies: https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/3-implement-access-control-strategies
- Fine-grained access control: https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/4-understand-fine-grained-access-control
- Row filtering and column masking: https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/5-implement-row-filtering-column-masking
- Azure Key Vault secrets: https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/6-access-azure-key-vault
- Service principals: https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/7-authenticate-data-access-service-principals
- Managed identities: https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/8-authenticate-resource-access-managed-identities
- Summary: https://learn.microsoft.com/en-us/training/modules/secure-unity-catalog-objects/11-summary

## Back to index

[README](../README.md)
