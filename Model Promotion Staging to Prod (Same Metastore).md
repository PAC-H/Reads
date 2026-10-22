# Model Promotion: Staging to Prod (Same Metastore)

Oct 22, 2026&#32;

## Purpose and scope

A model version validated in the staging catalog is promoted to prod by a **pull from the prod workspace**: a prod-owned job copies the exact artifact with `copy_model_version`, and two approval gates sit in front of it. Staging never gets write access to anything in prod, and prod never retrains.

**In scope:** Unity Catalog models on a single shared metastore, a staging workspace and a prod workspace, classic job clusters, prod consumption through both a Model Serving endpoint and a batch inference job.

**Out of scope:** retraining in prod ("deploy code"), cross-metastore promotion (Delta Sharing or export/import), feature table promotion.

**Approval model (layered):**

1. **Gate 1, GitHub Environments:** a reviewer on the `prod-model-release` environment approves copying a specific staging version into prod.
2. **Gate 2, MLflow deployment job:** the new prod version is evaluated on prod data, and a model approver signs off before the `@champion` alias moves and serving changes.

**Placeholder names used throughout.** Swap these for real values.

| Thing | Placeholder |
| --- | --- |
| Staging / prod workspace | `ws-stg` / `ws-prod` |
| Staging / prod catalog | `stg_ml` / `prod_ml` |
| Schema / model | `models` / `churn_classifier` |
| CI deploy identity | `sp-ghes-deploy-prod` |
| Promotion and deployment run-as identity | `sp-model-release-prod` |
| Batch inference run-as identity | `sp-batch-inference-prod` |
| Model approvers | `grp-model-approvers` |
| Aliases | `@release_candidate` (staging), `@champion` and `@previous` (prod) |

## Architecture

&#91;embedded content: promotion flow · 3 zones, 2 gates\]

The artifact moves straight down the right-hand column, from the staging model to the prod model, through the read-only binding. Orchestration runs down the left, and nothing reaches serving or batch without passing both gates.

## Identities and permissions

`sp-model-release-prod` **owns** the prod registered model, because only the owner can set aliases. Version creation needs ownership or the newer `CREATE MODEL VERSION` privilege. The CI identity can trigger the job but holds no data grants.

| Identity | Role | Unity Catalog grants | Workspace permissions |
| --- | --- | --- | --- |
| `sp-ghes-deploy-prod` | GHES workflow identity (OIDC to Entra) | None | Deploys the bundle to `ws-prod`; `CAN_MANAGE_RUN` on the promotion job |
| `sp-model-release-prod` | `run_as` for the promotion job and the deployment job | **Owner** of `prod_ml.models.churn_classifier`; `USE CATALOG` / `USE SCHEMA` on `prod_ml` and `stg_ml`; `EXECUTE` on `stg_ml.models.churn_classifier` | `CAN_MANAGE` on the serving endpoint |
| `grp-model-approvers` | Humans who sign off Gate 2 | `APPLY TAG` and `EXECUTE` on the prod model | `CAN_VIEW` on the deployment job |
| `sp-batch-inference-prod` | `run_as` for batch scoring | `EXECUTE` on the prod model; read on input tables, write on output tables | n/a |
| Endpoint consumers | Apps calling the endpoint | None | `CAN_QUERY` on the serving endpoint |

**Why split CI and run-as.** If a GHES token or runner is compromised, the attacker can trigger the promotion job, which is gated and idempotent. They cannot read staging artifacts or move aliases directly.

**Why one owner SP for both jobs.** Copying creates a version and the deploy step sets the alias, and both need ownership. Splitting them across two SPs would need `CREATE MODEL VERSION` on one and ownership on the other. Confirm `CREATE MODEL VERSION` is available on your workspace release before choosing that split.

**Staging side.** `sp-model-release-prod` gets read-only access to the staging model only. It has no grants on staging tables or volumes, and no staging-workspace permissions.

## Network path

The prod job cluster must reach **both** catalogs' storage accounts. Expect the MLflow client on the cluster to download the source artifacts and upload them to the destination, using UC-vended short-lived credentials. Verify this against your MLflow version, because a server-side copy would change the rules below.

| Path | From | To | What must allow it |
| --- | --- | --- | --- |
| Read source artifact | `ws-prod` job cluster subnets (VNet-injected) | `stg_ml` storage account | A private endpoint for the staging storage account resolvable from the prod VNet, or a VNet rule for the prod subnets on the staging storage firewall |
| Write promoted version | `ws-prod` job cluster subnets | `prod_ml` storage account | Existing prod private endpoint or VNet rule |
| Serving reads artifact | Serverless Model Serving | `prod_ml` storage account | Prod workspace NCC private endpoint rule (or NCC stable subnets) on the prod storage firewall |
| Batch reads artifact | `ws-prod` batch job cluster | `prod_ml` storage account | Same as the promoted-version write |

**Prefer a private endpoint** for the staging storage account in the prod VNet over opening the staging firewall to prod subnets. It is one more DNS record, but it keeps the staging firewall closed, and it can be scoped to the `blob` and `dfs` sub-resources.

**Credential path.** The UC storage credential (access connector managed identity) for each catalog is unchanged. The binding and grants decide whether UC vends a credential, and the network rules decide whether the cluster can use it.

**CMK.** If the storage accounts use customer-managed keys, the access connector identities already hold key permissions. Nothing new is needed for cross-workspace reads, because the read still goes through the staging catalog's own credential.

## Implementation 1: Terraform (bindings, model, grants)

Terraform owns the boundary: catalog isolation and bindings, the prod registered model and its owner, and every cross-environment grant. DABs never grant anything.

**Catalog bindings.** Both catalogs are `ISOLATED`. Staging is read-write in `ws-stg` and read-only in `ws-prod`. Prod is bound to `ws-prod` only.

```hcl
resource "databricks_catalog" "stg_ml" {
  provider       = databricks.stg
  name           = "stg_ml"
  storage_root   = var.stg_ml_storage_root
  isolation_mode = "ISOLATED"
}

resource "databricks_workspace_binding" "stg_ml_in_stg" {
  provider       = databricks.stg
  securable_type = "catalog"
  securable_name = databricks_catalog.stg_ml.name
  workspace_id   = var.ws_stg_id
  binding_type   = "BINDING_TYPE_READ_WRITE"
}

resource "databricks_workspace_binding" "stg_ml_in_prod" {
  provider       = databricks.stg
  securable_type = "catalog"
  securable_name = databricks_catalog.stg_ml.name
  workspace_id   = var.ws_prod_id
  binding_type   = "BINDING_TYPE_READ_ONLY"
}

resource "databricks_catalog" "prod_ml" {
  provider       = databricks.prod
  name           = "prod_ml"
  storage_root   = var.prod_ml_storage_root
  isolation_mode = "ISOLATED"
}

resource "databricks_workspace_binding" "prod_ml_in_prod" {
  provider       = databricks.prod
  securable_type = "catalog"
  securable_name = databricks_catalog.prod_ml.name
  workspace_id   = var.ws_prod_id
  binding_type   = "BINDING_TYPE_READ_WRITE"
}
```

**Prod registered model.** Pre-create it so ownership is set in code, not by whoever happens to register the first version.

```hcl
resource "databricks_registered_model" "churn_prod" {
  provider     = databricks.prod
  catalog_name = databricks_catalog.prod_ml.name
  schema_name  = "models"
  name         = "churn_classifier"
  owner        = var.sp_model_release_prod_app_id
  comment      = "Versions arrive only by promotion from stg_ml.models.churn_classifier"
}
```

**Grants.** Use the singular `databricks_grant` for these additions. The plural `databricks_grants` is authoritative and would wipe grants owned elsewhere (for example, by the staging team's own Terraform).

```hcl
locals {
  release_sp = var.sp_model_release_prod_app_id
  batch_sp   = var.sp_batch_inference_prod_app_id
  approvers  = "grp-model-approvers"
}

# Staging: read-only path to the one model
resource "databricks_grant" "stg_catalog_release" {
  provider   = databricks.stg
  catalog    = "stg_ml"
  principal  = local.release_sp
  privileges = ["USE_CATALOG"]
}

resource "databricks_grant" "stg_schema_release" {
  provider   = databricks.stg
  schema     = "stg_ml.models"
  principal  = local.release_sp
  privileges = ["USE_SCHEMA"]
}

resource "databricks_grant" "stg_model_release" {
  provider   = databricks.stg
  model      = "stg_ml.models.churn_classifier"
  principal  = local.release_sp
  privileges = ["EXECUTE"]
}

# Prod: traversal for everyone who touches the model
resource "databricks_grant" "prod_catalog" {
  for_each   = toset([local.release_sp, local.batch_sp, local.approvers])
  provider   = databricks.prod
  catalog    = "prod_ml"
  principal  = each.value
  privileges = ["USE_CATALOG"]
}

resource "databricks_grant" "prod_schema" {
  for_each   = toset([local.release_sp, local.batch_sp, local.approvers])
  provider   = databricks.prod
  schema     = "prod_ml.models"
  principal  = each.value
  privileges = ["USE_SCHEMA"]
}

# Prod model: approvers tag, batch executes; release SP is owner
resource "databricks_grant" "prod_model_approvers" {
  provider   = databricks.prod
  model      = databricks_registered_model.churn_prod.id
  principal  = local.approvers
  privileges = ["EXECUTE", "APPLY_TAG"]
}

resource "databricks_grant" "prod_model_batch" {
  provider   = databricks.prod
  model      = databricks_registered_model.churn_prod.id
  principal  = local.batch_sp
  privileges = ["EXECUTE"]
}
```

Notes:

- The staging model must exist before `stg_model_release` applies. If that ordering is awkward, grant `EXECUTE` on the `stg_ml.models` schema instead. That covers every model and function in the schema, which is broader.
- `databricks_workspace_binding` replaces the older `databricks_catalog_workspace_binding`. Check which one your provider version supports.
- Check that `owner` on `databricks_registered_model` is supported in your provider version. If it isn't, set it once with `ALTER MODEL prod_ml.models.churn_classifier OWNER TO ...`.
- The binding gives `ws-prod` reach into `stg_ml`, but grants still decide who can use it. Audit `stg_ml` for broad grants (`account users`, workspace groups) before binding, or prod users inherit read access to staging.

## Implementation 2: DABs bundle and job code

One prod bundle, `churn-model-release`, holds the promotion job and the deployment job, both running as `sp-model-release-prod`. The serving endpoint is created and updated **by the deploy task, not by the bundle**. Batch scoring lives in its own bundle so it can run as its own SP.

**Why the endpoint is not a bundle resource.** An endpoint declared in the bundle is created by the CI identity, which would then need `EXECUTE` on the prod model. It would also drift every time the deploy task moves the version. Managing it from the deploy task keeps the CI identity grant-free and leaves one writer of the served version.

### Bundle layout

```text
churn-model-release/
  databricks.yml
  resources/
    promote_job.yml
    deployment_job.yml
  src/
    promote.py
    evaluate.py
    approval.py        # Databricks sample approval notebook
    deploy.py
```

### databricks.yml

```yaml
bundle:
  name: churn-model-release

include:
  - resources/*.yml

variables:
  stg_model:
    default: stg_ml.models.churn_classifier
  prod_model:
    default: prod_ml.models.churn_classifier
  release_sp:
    description: Application ID of sp-model-release-prod
  ci_sp:
    description: Application ID of sp-ghes-deploy-prod
  endpoint_name:
    default: churn-classifier-prod

targets:
  prod:
    mode: production
    workspace:
      host: https://<ws-prod>.azuredatabricks.net
      root_path: /Workspace/Shared/.bundle/${bundle.name}/${bundle.target}
    run_as:
      service_principal_name: ${var.release_sp}
```

### resources/promote\_job.yml

```yaml
resources:
  jobs:
    promote_model:
      name: promote-churn-classifier
      max_concurrent_runs: 1
      parameters:
        - { name: source_version, default: "" }
        - { name: git_sha, default: "" }
        - { name: gh_run_url, default: "" }
        - { name: gate1_approver, default: "" }
      job_clusters:
        - job_cluster_key: single_node
          new_cluster:
            spark_version: 16.4.x-cpu-ml-scala2.12
            node_type_id: Standard_D4ds_v5
            num_workers: 0
            data_security_mode: SINGLE_USER
            spark_conf:
              spark.databricks.cluster.profile: singleNode
              spark.master: "local[*]"
            custom_tags:
              ResourceClass: SingleNode
      tasks:
        - task_key: promote
          job_cluster_key: single_node
          spark_python_task:
            python_file: ../src/promote.py
            parameters:
              - --source-model=${var.stg_model}
              - --source-version={{job.parameters.source_version}}
              - --target-model=${var.prod_model}
              - --git-sha={{job.parameters.git_sha}}
              - --gh-run-url={{job.parameters.gh_run_url}}
              - --gate1-approver={{job.parameters.gate1_approver}}
      permissions:
        - service_principal_name: ${var.ci_sp}
          level: CAN_MANAGE_RUN
```

### resources/deployment\_job.yml

A deployment job must take exactly two job parameters, `model_name` and `model_version`. It fires automatically when the promotion job creates a new prod version.

```yaml
resources:
  jobs:
    churn_deployment:
      name: churn-classifier-deployment
      max_concurrent_runs: 1
      parameters:
        - { name: model_name, default: "${var.prod_model}" }
        - { name: model_version, default: "" }
      job_clusters:
        - job_cluster_key: single_node
          new_cluster: { }   # same single-node spec as promote_job.yml
      tasks:
        - task_key: evaluation
          job_cluster_key: single_node
          spark_python_task:
            python_file: ../src/evaluate.py
            parameters:
              - --model-name={{job.parameters.model_name}}
              - --model-version={{job.parameters.model_version}}
        - task_key: Approval_Check
          depends_on: [{ task_key: evaluation }]
          job_cluster_key: single_node
          notebook_task:
            notebook_path: ../src/approval.py
            base_parameters:
              model_name: "{{job.parameters.model_name}}"
              model_version: "{{job.parameters.model_version}}"
              approval_tag_name: Approval_Check
        - task_key: deploy
          depends_on: [{ task_key: Approval_Check }]
          job_cluster_key: single_node
          spark_python_task:
            python_file: ../src/deploy.py
            parameters:
              - --model-name={{job.parameters.model_name}}
              - --model-version={{job.parameters.model_version}}
              - --endpoint-name=${var.endpoint_name}
      permissions:
        - group_name: grp-model-approvers
          level: CAN_VIEW
```

**Linking the job to the model (one-time).** Connect the deployment job to `prod_ml.models.churn_classifier` from the model page in Catalog Explorer, or run `MlflowClient().update_registered_model(name, deployment_job_id=<job id>)` as the model owner. The CI identity can't do this, because it isn't the owner.

### src/promote.py

This script copies one approved staging version into prod. It is idempotent, and it refuses anything that isn't validated or that has drifted since Gate 1.

```python
import argparse, sys, time
import mlflow
from mlflow import MlflowClient

p = argparse.ArgumentParser()
for arg in ["source-model", "source-version", "target-model",
            "git-sha", "gh-run-url", "gate1-approver"]:
    p.add_argument(f"--{arg}", required=True)
a = p.parse_args()

mlflow.set_registry_uri("databricks-uc")
client = MlflowClient()
src_uri = f"models:/{a.source_model}/{a.source_version}"
src = client.get_model_version(a.source_model, a.source_version)

# Guard 1: staging validation must have passed
if src.tags.get("validation_status") != "passed":
    raise RuntimeError(f"{src_uri} is not marked validation_status=passed")

# Guard 2: the version Gate 1 approved is still the release candidate
rc = client.get_model_version_by_alias(a.source_model, "release_candidate")
if rc.version != str(a.source_version):
    raise RuntimeError(
        f"@release_candidate is v{rc.version}, but Gate 1 approved v{a.source_version}")

# Idempotency: a re-run must not create a second prod copy
for mv in client.search_model_versions(f"name='{a.target_model}'"):
    if mv.tags.get("promoted_from") == src_uri:
        print(f"{src_uri} already promoted as v{mv.version}; nothing to do")
        sys.exit(0)

new = client.copy_model_version(src_uri, a.target_model)

for _ in range(60):
    if client.get_model_version(a.target_model, new.version).status == "READY":
        break
    time.sleep(10)
else:
    raise TimeoutError(f"{a.target_model} v{new.version} not READY after 10 min")

provenance = {
    "promoted_from": src_uri,
    "source_run_id": src.run_id or "",
    "git_sha": a.git_sha,
    "gh_run_url": a.gh_run_url,
    "gate1_approver": a.gate1_approver,
}
for k, v in provenance.items():
    client.set_model_version_tag(a.target_model, new.version, k, v)

print(f"Promoted {src_uri} -> {a.target_model} v{new.version}")
```

### src/evaluate.py

This script rejects versions that didn't arrive through promotion, then scores the version on a prod holdout set against the current champion. The release SP needs `SELECT` on the holdout table; add that grant to the Terraform.

```python
import argparse, time
import mlflow
from mlflow import MlflowClient
from mlflow.exceptions import MlflowException
from sklearn.metrics import roc_auc_score

p = argparse.ArgumentParser()
p.add_argument("--model-name", required=True)
p.add_argument("--model-version", required=True)
a = p.parse_args()

mlflow.set_registry_uri("databricks-uc")
client = MlflowClient()
MIN_AUC, MAX_DROP = 0.75, 0.01            # agree these with model risk

# The deployment job can fire before promote.py finishes tagging
for _ in range(12):
    mv = client.get_model_version(a.model_name, a.model_version)
    if "promoted_from" in mv.tags:
        break
    time.sleep(10)
else:
    raise RuntimeError("Version has no promoted_from tag: not created by the promotion job")

df = spark.table("prod_ml.eval.churn_holdout").toPandas()
X, y = df.drop(columns=["label"]), df["label"]

def auc(uri):
    return roc_auc_score(y, mlflow.pyfunc.load_model(uri).predict(X))

candidate = auc(f"models:/{a.model_name}/{a.model_version}")
try:
    champion = auc(f"models:/{a.model_name}@champion")
except MlflowException:
    champion = None                      # first release

client.set_model_version_tag(a.model_name, a.model_version, "eval_auc", f"{candidate:.4f}")
if candidate < MIN_AUC or (champion is not None and candidate < champion - MAX_DROP):
    raise RuntimeError(f"AUC {candidate:.4f} fails gate (champion {champion})")
```

### src/approval.py

Use the [Databricks sample approval notebook](https://docs.databricks.com/notebooks/source/mlflow/deployment-jobs/approval.html) unchanged. It fails unless the version carries the tag `Approval_Check = approved`. A member of `grp-model-approvers` approves from the model version page, which sets the tag, and the run then continues from the approval task. Confirm the exact approve-and-resume flow in your workspace.

### src/deploy.py

This script moves serving first and the alias second. If the endpoint update fails, `@champion` hasn't moved, so batch and serving never disagree.

```python
import argparse
from mlflow import MlflowClient
from mlflow.exceptions import MlflowException
from databricks.sdk import WorkspaceClient
from databricks.sdk.service.serving import EndpointCoreConfigInput, ServedEntityInput

p = argparse.ArgumentParser()
for arg in ["model-name", "model-version", "endpoint-name"]:
    p.add_argument(f"--{arg}", required=True)
a = p.parse_args()

client = MlflowClient(registry_uri="databricks-uc")
w = WorkspaceClient()

# 1. Keep a rollback pointer
try:
    current = client.get_model_version_by_alias(a.model_name, "champion")
    client.set_registered_model_alias(a.model_name, "previous", current.version)
except MlflowException:
    pass                                  # first release, nothing to keep

# 2. Point serving at the exact version (endpoints take a version, not an alias)
entity = ServedEntityInput(
    name="churn",
    entity_name=a.model_name,
    entity_version=a.model_version,
    workload_size="Small",
    scale_to_zero_enabled=False,
)
existing = {e.name for e in w.serving_endpoints.list()}
if a.endpoint_name in existing:
    w.serving_endpoints.update_config_and_wait(name=a.endpoint_name, served_entities=[entity])
else:
    w.serving_endpoints.create_and_wait(
        name=a.endpoint_name, config=EndpointCoreConfigInput(served_entities=[entity]))

# 3. Batch follows the alias on its next run
client.set_registered_model_alias(a.model_name, "champion", a.model_version)
client.set_model_version_tag(a.model_name, a.model_version, "deployed_to", a.endpoint_name)
```

**Rollback** uses the same script with `--model-version` set to the `@previous` version. You can run it from the deployment job's deploy task with a repaired run, or as a one-off run by the release SP.

### Batch scoring (separate bundle)

Batch scoring is its own bundle with `run_as: sp-batch-inference-prod`. It always resolves the alias at run time, so promotion needs no batch redeploy.

```python
import mlflow
mlflow.set_registry_uri("databricks-uc")
score = mlflow.pyfunc.spark_udf(spark, "models:/prod_ml.models.churn_classifier@champion")
(spark.table("prod_ml.features.churn_daily")
      .withColumn("churn_score", score())
      .write.mode("overwrite").saveAsTable("prod_ml.scores.churn_daily"))
```

Log the resolved version number with each batch run, using `get_model_version_by_alias`. Then every scored row can be traced to an exact version.

## Implementation 3: GitHub Actions (Gate 1)

Two workflows, each with its own environment and Entra federated credential:

- **`deploy-bundle.yml`** runs on merge to `main` and deploys the bundle code.
- **`promote-model.yml`** is a manual dispatch. It promotes a model version and is Gate 1.

The person dispatching names the staging version explicitly. CI never reads staging, so it needs no UC grant. `promote.py` then refuses the run if that version isn't the current `@release_candidate`.

### Entra and environment setup

| Setting | Value |
| --- | --- |
| Federated credential issuer | `https://<ghes-host>/_services/token` |
| Subject for promotion | `repo:<org>/<repo>:environment:prod-model-release` |
| Subject for bundle deploy | `repo:<org>/<repo>:environment:prod-bundle-deploy` |
| `prod-model-release` reviewers | Model release approvers team; enable "prevent self-review" if your GHES version has it |
| Deployment branches | `main` only, on both environments |

Binding the subject to the environment means a token for `sp-ghes-deploy-prod` exists only after a reviewer has approved that job.

### .github/workflows/promote-model.yml

```yaml
name: promote-model-to-prod

on:
  workflow_dispatch:
    inputs:
      source_version:
        description: "stg_ml.models.churn_classifier version (must be @release_candidate)"
        required: true
        type: string

permissions:
  id-token: write
  contents: read
  actions: read

concurrency:
  group: promote-churn-classifier
  cancel-in-progress: false

jobs:
  promote:
    runs-on: [self-hosted, aks-prod]
    environment: prod-model-release          # Gate 1: reviewers approve here
    env:
      SOURCE_VERSION: ${{ inputs.source_version }}
      DATABRICKS_HOST: ${{ vars.DATABRICKS_HOST_PROD }}
      DATABRICKS_AUTH_TYPE: azure-cli
      BUNDLE_VAR_release_sp: ${{ vars.SP_MODEL_RELEASE_PROD_APP_ID }}
      BUNDLE_VAR_ci_sp: ${{ vars.SP_GHES_DEPLOY_PROD_APP_ID }}
    steps:
      - name: Validate input
        run: |
          [[ "$SOURCE_VERSION" =~ ^[0-9]+$ ]] || { echo "source_version must be an integer"; exit 1; }
          echo "Promoting stg_ml.models.churn_classifier v$SOURCE_VERSION" >> "$GITHUB_STEP_SUMMARY"

      - name: Record Gate 1 approver
        id: gate1
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          approver=$(curl -fsSL -H "Authorization: Bearer $GH_TOKEN" \
            "$GITHUB_API_URL/repos/$GITHUB_REPOSITORY/actions/runs/$GITHUB_RUN_ID/approvals" \
            | jq -r '[.[] | select(.state == "approved") | .user.login] | last')
          [[ -n "$approver" && "$approver" != "null" ]] || { echo "No approval found"; exit 1; }
          echo "login=$approver" >> "$GITHUB_OUTPUT"

      - uses: actions/checkout@<pinned-sha>

      - uses: azure/login@<pinned-sha>
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID_DEPLOY_PROD }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          allow-no-subscriptions: true

      - uses: databricks/setup-cli@<pinned-sha>

      - name: Run promotion job in ws-prod
        working-directory: churn-model-release
        env:
          GATE1_APPROVER: ${{ steps.gate1.outputs.login }}
        run: |
          run_url="$GITHUB_SERVER_URL/$GITHUB_REPOSITORY/actions/runs/$GITHUB_RUN_ID"
          databricks bundle run promote_model -t prod \
            --params "source_version=$SOURCE_VERSION,git_sha=$GITHUB_SHA,gh_run_url=$run_url,gate1_approver=$GATE1_APPROVER"
```

Design points in the workflow:

- **No `${{ }}` inside `run:`.** Inputs go through `env:` and are checked as an integer, which closes the script-injection path.
- **Approver captured from the API, not `github.actor`.** The actor is whoever dispatched the run. The approvals endpoint names the reviewer, and that login is written onto the prod model version.
- **Actions pinned by SHA** and mirrored to GHES through your usual actions sync.
- **`bundle run` waits for the job** and fails the workflow if `promote.py` raises, so a refused promotion shows red in GitHub.

### .github/workflows/deploy-bundle.yml (outline)

```yaml
on:
  push:
    branches: [main]
    paths: ["churn-model-release/**"]
jobs:
  deploy:
    runs-on: [self-hosted, aks-prod]
    environment: prod-bundle-deploy
    # same azure/login + setup-cli steps as above
    steps:
      - run: databricks bundle validate -t prod && databricks bundle deploy -t prod
        working-directory: churn-model-release
```

## Controls, risks and mitigations

The weakest point is Gate 2. The release SP owns the prod model, so it can set any tag, including `Approval_Check`. Only reviewed job code stops it from approving itself. Everything else below has a code-level guard.

| Risk | Where | Mitigation | Residual |
| --- | --- | --- | --- |
| Release SP self-approves Gate 2 | Owner can set any tag | Job code changes only through reviewed `main` and the `prod-bundle-deploy` environment. Alert on `Approval_Check` tag writes by anyone outside `grp-model-approvers` | Medium; check whether a governed tag policy can restrict who sets that key |
| CI identity runs or edits the deployment job | Bundle deployer gets `CAN_MANAGE` on what it deploys | Approval task still blocks without the human tag. Bundle deploy is gated by its own environment review | Low |
| Alias moves between Gate 1 approval and copy | `@release_candidate` in staging | Guard 2 in `promote.py` compares the approved version to the live alias | Low |
| Version created outside promotion | Admin or owner registers directly in prod | `evaluate.py` refuses versions without `promoted_from`; audit `createModelVersion`-type events | Low; approvers with `APPLY TAG` could forge the tag, and audit catches it |
| Broad staging grants leak into prod | `READ_ONLY` binding exposes `stg_ml` to `ws-prod` | Audit `stg_ml` grants before binding; no grants to `account users` or workspace-wide groups | Low once audited |
| Serving and batch disagree | Partial failure in `deploy.py` | Serving moves first, alias second; `@previous` kept for rollback | Low |
| Promoted model can't build in serving | Locked-down PyPI egress | Pin `requirements` at training time; route serving builds through the approved package mirror; smoke test in a staging endpoint first | Medium until the mirror path is proven |
| Lineage gap in prod | Source run lives in the `ws-stg` experiment | `source_run_id`, `promoted_from`, `git_sha` and `gh_run_url` tags on every prod version; grant approvers read on the staging experiment if they need the run | Accepted |
| Re-run creates duplicate prod versions | Workflow retried | Idempotency check on `promoted_from` | Low |

**Audit evidence per release** (enough for change management):

- GHES run: dispatcher, Gate 1 reviewer, commit SHA.
- Prod model version tags: `promoted_from`, `source_run_id`, `git_sha`, `gh_run_url`, `gate1_approver`, `eval_auc`, `Approval_Check`, `deployed_to`.
- Deployment job run history on the model version page.
- `system.access.audit` events for model version creation, tag changes and alias changes on `prod_ml.models.churn_classifier`. Confirm the exact `action_name` values in your audit table before writing the alert.

## Open questions and things to validate

Validate each of these in a sandbox before the first real promotion:

- [ ] Does `copy_model_version` copy client-side on our MLflow version? This decides whether the prod cluster needs network reach to staging storage.
- [ ] Does the deployment job fire for versions created by `copy_model_version`, and does it pass `model_version` correctly?
- [ ] What is the exact approve-and-resume flow for the approval task: button on the version page, then repair run?
- [ ] Is `CREATE MODEL VERSION` available on our workspaces? If so, consider splitting the copy SP from the alias-owning SP.
- [ ] Can a governed tag policy restrict who sets `Approval_Check` on model versions?
- [ ] Does our Terraform provider version support `databricks_workspace_binding`, `databricks_grant` with `model`, and `owner` on `databricks_registered_model`?
- [ ] Does our GHES version support the run approvals API and "prevent self-review" on environments?
- [ ] Is the serving endpoint build path to the package mirror proven under the locked-down egress?
- [ ] Model risk: who owns the `MIN_AUC` and `MAX_DROP` thresholds, and are they set per model?
- [ ] Audit: what are the `action_name` values for version creation, tag set and alias set in `system.access.audit`?

## References

- [Upgrade ML workflows to target models in Unity Catalog](https://docs.databricks.com/aws/en/machine-learning/manage-model-lifecycle/upgrade-workflows): required privileges, `copy_model_version` promotion example
- [Upgrade ML workflows (Azure Databricks)](https://learn.microsoft.com/en-gb/azure/databricks/machine-learning/manage-model-lifecycle/upgrade-workflows)
- [MLflow 3 deployment jobs](https://learn.microsoft.com/en-in/Azure/databricks/mlflow/deployment-job)
- [Deployment job approval notebook](https://docs.databricks.com/notebooks/source/mlflow/deployment-jobs/approval.html)
- [Limit catalog access to specific workspaces](https://learn.microsoft.com/en-us/azure/databricks/catalogs/binding)
- [Databricks SDK: Registered Models API](https://databricks-sdk-py.readthedocs.io/en/stable/workspace/catalog/registered_models.html): owner-only alias and metadata updates
- [Community: serving a UC model by alias with DABs](https://community.databricks.com/t5/machine-learning/custom-model-serving-using-databricks-asset-bundles/m-p/112886/highlight/true): endpoints take a version, so resolve the alias first
- [Community: UC model deployment across workspaces](https://community.databricks.com/t5/machine-learning/uc-model-deployment-across-data-bricks-instances/td-p/141615)
