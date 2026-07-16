# Databricks → Snowflake Authentication with Entra ID Service Principals
## MLOps Production Deployment — Design & Implementation

| | |
|---|---|
| **Status** | Draft |
| **Owner** | Platform Engineering |
| **Snowflake security integration owner** | Snowflake Platform Team |
| **Last updated** | 2026-07-16 |

---

## 1. Purpose

This document defines how Databricks jobs deployed through our MLOps pipeline authenticate to Snowflake for **read and write** data access, using Microsoft Entra ID service principals (SPs) with Snowflake External OAuth.

It covers:

- The service principal model (CI/CD deploy SPs vs. runtime SPs) and the role each plays in data access.
- Why Lakehouse Federation cannot meet this requirement (read-only).
- Why the runtime SPs **must** use client secrets, and cannot use federated (secretless) credentials.
- Full implementation steps: Entra ID, Snowflake, Azure Key Vault, Databricks secret scopes, and runtime code.

MLOps pipeline internals (bundle structure, model promotion, testing gates) are out of scope; only the identity and data-access surface is described.

---

## 2. Service Principal Model — Summary

Each application ("app") that reads from or writes to Snowflake operates with **three** Entra ID service principals:

| SP | Example name | Authenticates to | Credential type | Purpose |
|---|---|---|---|---|
| CI/CD deploy SP | `sp-<app>-cicd` | Databricks (staging + prod workspaces) | **Federated credential (OIDC, secretless)** | Deploys the Databricks Asset Bundle (jobs, pipelines, configs) from the CI/CD pipeline. **Never touches Snowflake data.** |
| Staging runtime SP | `sp-<app>-dbx-stg` | Snowflake (via Entra External OAuth) | **Client secret** (stored in Key Vault) | `run_as` identity for jobs/tasks in the **staging** workspace; reads and writes staging Snowflake objects. |
| Production runtime SP | `sp-<app>-dbx-prd` | Snowflake (via Entra External OAuth) | **Client secret** (stored in Key Vault) | `run_as` identity for jobs/tasks in the **production** workspace; reads and writes production Snowflake objects. |

### 2.1 Role of the runtime SPs in data access

The runtime SP is the **only identity that touches Snowflake data** for a given app and environment. At job runtime, the task:

1. Reads the SP's client ID and client secret from a Key Vault–backed Databricks secret scope.
2. Performs an OAuth 2.0 client-credentials exchange against the Entra ID token endpoint, requesting a token for the Snowflake resource application.
3. Presents the resulting access token to Snowflake via the Spark–Snowflake connector (`sfAuthenticator = oauth`).
4. Snowflake validates the token against its External OAuth security integration, maps the token to a Snowflake service user, and authorizes the session under that user's role.

All Snowflake access is therefore attributable to a specific app + environment, governed by Snowflake RBAC on the mapped service user, and auditable end-to-end (Entra sign-in logs → Snowflake `LOGIN_HISTORY` / `QUERY_HISTORY`).

### 2.2 One SP per app, per environment — mandatory

**Do not share runtime SPs across applications.** Each app gets its own staging and production runtime SPs, scoped to exactly the Snowflake objects that app requires:

- **Least privilege.** Different apps need different databases, schemas, and read/write combinations. A shared SP accumulates the union of all apps' privileges.
- **Blast radius.** A leaked or misused secret compromises one app in one environment, not the whole estate.
- **Independent rotation.** Secrets can be rotated per app without coordinating downtime across teams.
- **Attribution.** Snowflake access history maps 1:1 to an app and environment — no ambiguity during incident response or access reviews.
- **Clean offboarding.** Decommissioning an app is deleting its SPs, secrets, and Snowflake user — nothing else is affected.

Where a single app has workloads with materially different access needs (e.g. a read-only inference path and a read-write feature-engineering path), split further into multiple runtime SPs per environment.

---

## 3. Why Not Lakehouse Federation

Lakehouse Federation (Unity Catalog foreign catalogs over a Snowflake connection) is the preferred pattern for **governed read access**, but it is **read-only by design** and cannot support this use case:

- Unity Catalog vends **read credentials only** for foreign tables; it will not issue write credentials under any circumstance.
- Any write attempt fails, e.g.:

  ```
  [INVALID_PARAMETER_VALUE.CHILD_CREATION_FORBIDDEN_FOR_FOREIGN_SECURABLE]
  Lakehouse Federation only supports read operations for external data sources.
  Write operations, such as creating new schemas and tables, and modifying
  table contents, are not supported.
  ```

- This applies to both query federation (JDBC pushdown) and catalog federation (Iceberg/direct storage reads). Inserts, updates, deletes, and DDL must be performed in the source engine.

Because the MLOps pipeline must **write** results (scored outputs, features, model telemetry) back to Snowflake, federation is disqualified. The supported write path is the **Spark–Snowflake connector** bundled in the Databricks Runtime, which is also Databricks' documented recommendation when write access is required.

> **Note — complementary use.** Federation may still be used elsewhere for governed, read-only analyst access to the same Snowflake data. This document's pattern applies specifically to the pipeline read/write path.

---

## 4. Why Client Secrets Are Required (No Federated Credentials for Runtime SPs)

Secretless workload identity federation (WIF) is used **only** for the CI/CD deploy SPs. It is **not possible** for the runtime SPs, and a Key Vault–stored client secret is mandatory. The reasons:

1. **The federated exchange needs a platform-minted token.** An Entra federated credential replaces the client secret with a signed `client_assertion` JWT issued by a platform Entra has been configured to trust (GitHub Actions, Azure DevOps, AKS workload identity, Azure Functions, etc.). The workload platform must be able to mint that assertion.
2. **Databricks job compute cannot mint one.** Code running inside a Databricks job has no mechanism to obtain a signed identity token that Entra will accept as a `client_assertion` for an arbitrary app registration. Databricks' OIDC/WIF capability works in the *inbound* direction (external CI systems authenticating **to** Databricks) — it does not provide outbound assertions for Entra client-credential flows.
3. **Microsoft documents the limitation explicitly.** For Entra-backed service principals interacting with Snowflake, Microsoft's guidance states the SP cannot use the Databricks OIDC token endpoint and must authenticate directly against the Microsoft Entra token endpoint — which, without a platform-minted assertion, means a client secret (or certificate).
4. **No managed identity path.** Managed identities are bound to Azure resources; classic and serverless Databricks job compute does not expose a customer-controlled managed identity that could represent the runtime SP.

**Consequence:** each runtime SP has a client secret, which must be stored in Azure Key Vault, surfaced to jobs via a Key Vault–backed secret scope, never embedded in code or bundle files, and rotated on a defined cadence (see §9).

| | CI/CD deploy SP | Runtime SPs (stg / prd) |
|---|---|---|
| Authenticates to | Databricks workspaces | Entra token endpoint → Snowflake |
| Credential | Federated credential (OIDC) — no secret | Client secret in Key Vault |
| Why | GitHub Actions can mint the OIDC assertion | Databricks compute cannot mint an assertion |

---

## 5. Authentication Flow (Runtime)

```
┌──────────────────────────┐
│ Databricks job task      │
│ (run_as: sp-<app>-dbx-*) │
└─────────┬────────────────┘
          │ 1. dbutils.secrets.get() ── Key Vault–backed secret scope ──► Azure Key Vault
          │      (client ID + client secret)
          ▼
┌──────────────────────────┐
│ Entra ID token endpoint  │  2. POST /oauth2/v2.0/token
│ login.microsoftonline.com│     grant_type=client_credentials
└─────────┬────────────────┘     scope=api://<resource-app>/.default
          │ 3. access_token (JWT, aud=api://<resource-app>, sub=<SP object ID>)
          ▼
┌──────────────────────────┐
│ Snowflake                │  4. Spark connector: sfAuthenticator=oauth, sfToken=<JWT>
│ External OAuth           │  5. Validates issuer/audience/signature against
│ security integration     │     security integration; maps sub → service user
└──────────────────────────┘  6. Session runs as service user + role
```

---

## 6. Entra ID Setup

Two topology options are available. **A decision is pending** — both are documented below with trade-offs.

### 6.1 Option A — One shared Snowflake resource app, per-app client SPs

A single app registration represents "Snowflake" as a resource in the tenant. All runtime SPs are client apps that request tokens for it.

```
Resource app:  app-snowflake-resource        (one per tenant/Snowflake account)
Client SPs:    sp-app1-dbx-stg, sp-app1-dbx-prd,
               sp-app2-dbx-stg, sp-app2-dbx-prd, ...
```

**Pros**

- One Snowflake security integration audience to manage; new apps require no Snowflake Platform Team change on the integration itself (only a new service user + role grants).
- Simpler Entra estate; app roles on the resource app can model Snowflake roles centrally.

**Cons**

- The resource app becomes shared infrastructure — a config error affects every consumer.
- All tokens share one audience; per-app isolation relies entirely on the `sub` → service-user mapping and Snowflake RBAC.

### 6.2 Option B — Separate resource app per application

Each app gets its own resource app registration and audience.

**Pros**

- Hard audience-level isolation between apps; a compromised or misconfigured app cannot present a token that even *parses* as valid for another app's integration scope.
- Per-app lifecycle: decommissioning removes the whole Entra footprint.

**Cons**

- Snowflake's security integration must list every audience (`EXTERNAL_OAUTH_AUDIENCE_LIST` grows per app), or one integration per app must be created — either way, every app onboarding requires a Snowflake Platform Team change.
- More Entra objects to govern.

> **Recommendation to discuss:** Option A with strict `sub`-based user mapping is the common pattern and operationally lighter; Option B suits regulated boundaries where audience-level separation is required. The implementation below is written for Option A; Option B differs only in the number of resource apps and the audience list.

### 6.3 Steps (Option A shown)

**6.3.1 Resource app (once):**

```bash
# Create the resource app registration representing Snowflake
az ad app create --display-name "app-snowflake-resource"

# Note the appId; set the Application ID URI
az ad app update --id <RESOURCE_APP_ID> \
  --identifier-uris "api://<RESOURCE_APP_ID>"
```

In the portal (App registrations → app-snowflake-resource → App roles), create an app role per Snowflake session role you intend to expose, e.g.:

| Display name | Value | Allowed member types |
|---|---|---|
| `session:role-any` | `session:role-any` | Applications |

*(Exact app-role strategy to be finalized alongside §8 role design. `EXTERNAL_OAUTH_ANY_ROLE_MODE = 'ENABLE'` on the Snowflake side, shown in §7, lets the Snowflake user's granted roles govern access rather than token claims — the simplest and recommended starting point.)*

**6.3.2 Runtime client SPs (per app, per environment):**

```bash
# Repeat for -stg and -prd
az ad app create --display-name "sp-<app>-dbx-prd"
az ad sp create --id <CLIENT_APP_ID>

# Create the client secret (capture the value once; it is not retrievable later)
az ad app credential reset --id <CLIENT_APP_ID> \
  --display-name "snowflake-oauth" \
  --years 1
```

Record for each SP:

- **Application (client) ID** — used in the token request.
- **Enterprise application Object ID** (App registrations → *Managed application in local directory* → Object ID) — this is the `sub` claim in client-credential tokens and is what Snowflake maps to a service user. Referred to below as `<SP_OBJECT_ID>`.

**6.3.3 Authorize the client against the resource app:** assign the client SP the app role on the resource app (API permissions → Add permission → *app-snowflake-resource* → Application permissions → grant admin consent), or via `az ad app permission add` / `az rest` app-role assignment.

**6.3.4 CI/CD deploy SP (per app, once — secretless):** add a federated credential instead of a secret:

```bash
az ad app federated-credential create --id <CICD_APP_ID> --parameters '{
  "name": "github-actions-<app>-prd",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<org>/<repo>:environment:production",
  "audiences": ["api://AzureADTokenExchange"]
}'
```

This SP is granted deploy permissions in the Databricks workspaces only. It receives **no** Snowflake resource permissions and **no** Key Vault access.

---

## 7. Snowflake Setup

> **Ownership:** the security integration below is **created and owned by the Snowflake Platform Team** (requires `ACCOUNTADMIN`). It is account-level, created **once**, and shared by all apps under Option A. App teams request service users and role grants against it; they do not modify the integration.

### 7.1 External OAuth security integration *(Snowflake Platform Team)*

```sql
CREATE SECURITY INTEGRATION entra_external_oauth_databricks
  TYPE = EXTERNAL_OAUTH
  ENABLED = TRUE
  EXTERNAL_OAUTH_TYPE = AZURE
  EXTERNAL_OAUTH_ISSUER = 'https://login.microsoftonline.com/<TENANT_ID>/v2.0'
  EXTERNAL_OAUTH_JWS_KEYS_URL =
    'https://login.microsoftonline.com/<TENANT_ID>/discovery/v2.0/keys'
  EXTERNAL_OAUTH_AUDIENCE_LIST = ('api://<RESOURCE_APP_ID>')
  EXTERNAL_OAUTH_TOKEN_USER_MAPPING_CLAIM = 'sub'
  EXTERNAL_OAUTH_SNOWFLAKE_USER_MAPPING_ATTRIBUTE = 'LOGIN_NAME'
  EXTERNAL_OAUTH_ANY_ROLE_MODE = 'ENABLE';
```

Notes:

- `sub` in a client-credentials token is the client SP's **enterprise app Object ID**; mapping it to `LOGIN_NAME` binds each Entra SP to exactly one Snowflake user.
- `ANY_ROLE_MODE = 'ENABLE'` delegates authorization to Snowflake RBAC (the service user's granted roles) rather than parsing role scopes out of the token. Access control lives in one place: Snowflake.
- Under **Option B**, `EXTERNAL_OAUTH_AUDIENCE_LIST` carries one entry per app resource app (or one integration per app is created).

### 7.2 Service users (per app, per environment)

```sql
-- Staging
CREATE USER svc_<app>_stg
  LOGIN_NAME    = '<SP_OBJECT_ID_STG>'          -- enterprise app Object ID, case-sensitive
  TYPE          = SERVICE                        -- no password, no interactive login
  DEFAULT_ROLE  = <APP_STG_ROLE>                 -- see §8
  DEFAULT_WAREHOUSE = <STG_WAREHOUSE>
  COMMENT       = 'Runtime SP for <app>, Databricks staging workspace';

-- Production
CREATE USER svc_<app>_prd
  LOGIN_NAME    = '<SP_OBJECT_ID_PRD>'
  TYPE          = SERVICE
  DEFAULT_ROLE  = <APP_PRD_ROLE>
  DEFAULT_WAREHOUSE = <PRD_WAREHOUSE>
  COMMENT       = 'Runtime SP for <app>, Databricks production workspace';
```

Optionally, pin each user to External OAuth only:

```sql
ALTER USER svc_<app>_prd SET
  DEFAULT_SECONDARY_ROLES = ();
-- Authentication policy restricting to OAUTH can be applied per account standards.
```

### 7.3 Roles and privileges — **TBD**

> **⬜ To be completed.** This section will define, per application and environment: the functional role(s) granted to each service user; database/schema/table grants (read vs. write); warehouse usage grants; and future-grants strategy. Placeholder:

```sql
-- CREATE ROLE <APP_PRD_ROLE>;
-- GRANT USAGE ON DATABASE ...          TO ROLE <APP_PRD_ROLE>;
-- GRANT USAGE ON SCHEMA ...            TO ROLE <APP_PRD_ROLE>;
-- GRANT SELECT ON ALL TABLES IN ...    TO ROLE <APP_PRD_ROLE>;
-- GRANT INSERT, UPDATE ON ...          TO ROLE <APP_PRD_ROLE>;
-- GRANT USAGE ON WAREHOUSE ...         TO ROLE <APP_PRD_ROLE>;
-- GRANT ROLE <APP_PRD_ROLE> TO USER svc_<app>_prd;
```

### 7.4 Validation *(Snowflake Platform Team / app team)*

```sql
-- Confirm the integration parses a sample token correctly (run with a real token)
SELECT SYSTEM$VERIFY_EXTERNAL_OAUTH_TOKEN('<access_token_jwt>');

-- Confirm login attribution after first connection
SELECT event_timestamp, user_name, first_authentication_factor
FROM snowflake.account_usage.login_history
WHERE user_name = 'SVC_<APP>_PRD'
ORDER BY event_timestamp DESC
LIMIT 10;
```

---

## 8. Azure Key Vault + Databricks Secret Scope

Runtime SP secrets are stored in Azure Key Vault and exposed to jobs through a **Key Vault–backed secret scope**. Secrets never appear in bundle files, job parameters, or notebook source.

### 8.1 Key Vault layout

One Key Vault per environment (aligned to the workspace), e.g. `kv-mlops-stg`, `kv-mlops-prd`. Per app, store:

| Secret name | Value |
|---|---|
| `sp-<app>-client-id` | Runtime SP application (client) ID |
| `sp-<app>-client-secret` | Runtime SP client secret |
| `sf-account-url` | `<account_identifier>.snowflakecomputing.com` (shared) |
| `entra-tenant-id` | Tenant ID (shared) |
| `sf-resource-app-id` | Resource app ID for the OAuth scope (shared) |

```bash
az keyvault secret set --vault-name kv-mlops-prd \
  --name "sp-<app>-client-id"     --value "<CLIENT_APP_ID>"
az keyvault secret set --vault-name kv-mlops-prd \
  --name "sp-<app>-client-secret" --value "<CLIENT_SECRET>"
```

### 8.2 Grant the Databricks workspace access to Key Vault

Key Vault–backed scopes are read by the **AzureDatabricks** first-party enterprise application, not by individual users or the runtime SP. With RBAC-enabled Key Vault:

```bash
az role assignment create \
  --assignee "2ff814a6-3304-4ab8-85cb-cd0e6f879c1d" \   # AzureDatabricks first-party app
  --role "Key Vault Secrets User" \
  --scope "/subscriptions/<SUB>/resourceGroups/<RG>/providers/Microsoft.KeyVault/vaults/kv-mlops-prd"
```

*(Access-policy vaults: grant Get/List on secrets to the same application.)*

### 8.3 Create the Key Vault–backed secret scope

```bash
databricks secrets create-scope \
  --json '{
    "scope": "snowflake-<app>",
    "scope_backend_type": "AZURE_KEYVAULT",
    "backend_azure_keyvault": {
      "resource_id": "/subscriptions/<SUB>/resourceGroups/<RG>/providers/Microsoft.KeyVault/vaults/kv-mlops-prd",
      "dns_name": "https://kv-mlops-prd.vault.azure.net/"
    }
  }'
```

Constraints worth noting:

- Creating a Key Vault–backed scope requires an **Entra ID user identity** (not a Databricks-native SP token) with permission on both the workspace and the vault — treat scope creation as a platform provisioning step.
- Key Vault–backed scopes are **read-only** from Databricks: secrets are written in Key Vault (`az keyvault secret set`), never via the Databricks Secrets API.

### 8.4 Lock the scope to the runtime SP

Grant `READ` on the scope to the app's runtime SP only, and remove default broad access:

```bash
databricks secrets put-acl snowflake-<app> <RUNTIME_SP_APPLICATION_ID> READ
# Review with:
databricks secrets list-acls snowflake-<app>
```

The CI/CD deploy SP gets **no ACL** on this scope — it deploys job definitions but can never read data-access credentials.

---

## 9. Databricks Runtime Implementation

### 9.1 Asset bundle — run_as (illustrative, minimal)

The bundle deployed by the CI/CD SP sets the runtime SP as the job execution identity per target:

```yaml
# databricks.yml (excerpt)
targets:
  staging:
    workspace:
      host: https://<staging-workspace>.azuredatabricks.net
    run_as:
      service_principal_name: <STG_RUNTIME_SP_APPLICATION_ID>
  prod:
    workspace:
      host: https://<prod-workspace>.azuredatabricks.net
    run_as:
      service_principal_name: <PRD_RUNTIME_SP_APPLICATION_ID>
```

### 9.2 Token acquisition + Spark–Snowflake connector (read & write)

```python
import requests

SCOPE = "snowflake-<app>"   # Key Vault-backed secret scope

# --- 1. Pull identity material from the secret scope ---
tenant_id     = dbutils.secrets.get(SCOPE, "entra-tenant-id")
client_id     = dbutils.secrets.get(SCOPE, "sp-<app>-client-id")
client_secret = dbutils.secrets.get(SCOPE, "sp-<app>-client-secret")
resource_app  = dbutils.secrets.get(SCOPE, "sf-resource-app-id")
sf_url        = dbutils.secrets.get(SCOPE, "sf-account-url")

# --- 2. Client-credentials exchange with Entra ID ---
def get_snowflake_oauth_token() -> str:
    resp = requests.post(
        f"https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token",
        data={
            "grant_type":    "client_credentials",
            "client_id":     client_id,
            "client_secret": client_secret,
            "scope":         f"api://{resource_app}/.default",
        },
        timeout=30,
    )
    resp.raise_for_status()
    return resp.json()["access_token"]

# --- 3. Connector options ---
def sf_options() -> dict:
    return {
        "sfUrl":           sf_url,
        "sfUser":          "svc_<app>_prd",     # must match the mapped LOGIN_NAME's user
        "sfAuthenticator": "oauth",
        "sfToken":         get_snowflake_oauth_token(),  # fetch fresh per read/write
        "sfDatabase":      "<DATABASE>",
        "sfSchema":        "<SCHEMA>",
        "sfWarehouse":     "<WAREHOUSE>",
        "sfRole":          "<APP_PRD_ROLE>",
    }

# --- 4. Read (table or pushdown query) ---
df = (spark.read
      .format("snowflake")
      .options(**sf_options())
      .option("query", "SELECT * FROM SOURCE_TABLE WHERE LOAD_DT = CURRENT_DATE()")
      .load())

# ... scoring / transformation (MLOps logic out of scope) ...

# --- 5. Write ---
(result_df.write
    .format("snowflake")
    .options(**sf_options())
    .option("dbtable", "TARGET_TABLE")
    .mode("append")            # or "overwrite"
    .save())
```

Implementation notes:

- **Token lifetime.** Entra access tokens live ~60–90 minutes. Fetch a fresh token per read/write operation (as above) rather than once per job; long tasks holding a stale token will fail mid-flight.
- **`sfUser` must align** with the Snowflake user whose `LOGIN_NAME` equals the SP's Object ID; a mismatch surfaces as a misleading "Incorrect username or password" error.
- **Write staging path.** The connector stages write data through cloud storage. In our controlled-egress network, confirm compute → stage → Snowflake connectivity is permitted before first production write.
- **Secret redaction.** `dbutils.secrets.get` values are redacted in notebook/job output; never log or print the token or secret.
- **No secrets in bundle files.** Only the scope name and secret keys are referenced in code; values resolve at runtime.

---

## 10. Secret Rotation & Operations

| Item | Practice |
|---|---|
| Client secret lifetime | ≤ 12 months (align to enterprise credential standard); calendar-tracked per SP |
| Rotation procedure | Add new secret version in Entra → update Key Vault secret value → jobs pick it up on next run (Key Vault–backed scope resolves latest version) → delete old Entra secret |
| Rotation downtime | Zero, if the new Entra secret is created before the Key Vault value is updated |
| Monitoring | Entra sign-in logs (client-credential grants per SP); Snowflake `LOGIN_HISTORY` / `QUERY_HISTORY` per service user; alert on failed OAuth validations |
| Access reviews | Runtime SP ↔ Snowflake role grants reviewed with §7.3 role design cadence |

---

## 11. Decision Log / Open Items

| # | Item | Status |
|---|---|---|
| 1 | Entra topology: shared resource app (Option A) vs. per-app resource apps (Option B) | **Open — pending decision (§6)** |
| 2 | Snowflake roles & privileges per app/environment | **Open — §7.3 to be completed** |
| 3 | Security integration creation | Owned by Snowflake Platform Team (§7.1) |
| 4 | Runtime SP credential type | **Decided:** client secret in Key Vault; federated credentials not technically possible from Databricks compute (§4) |
| 5 | Lakehouse Federation for this workload | **Decided:** rejected — read-only (§3) |
