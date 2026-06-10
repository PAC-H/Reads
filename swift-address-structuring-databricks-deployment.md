# Swift ISO 20022 Address Structuring Model — Databricks Deployment & Usage Guide

> **Source:** [Swift-SC/iso20022-address-structuring](https://github.com/Swift-SC/iso20022-address-structuring) · companion repo [iso20022-address-structuring-resources](https://github.com/Swift-SC/iso20022-address-structuring-resources)
>
> **Purpose:** Deploy Swift's open-source address structuring model as a Databricks Model Serving endpoint (real-time) and as a Spark UDF (batch), to support the **November 2026 CBPR+ deadline** after which cross-border Swift payments must carry structured or hybrid postal addresses.

---

## 1. What the Model Is

The model infers structured **Town** and **Country** from unstructured/legacy address text, returning confidence scores and explainability fields to support both automation and expert review.

It is a **four-stage pipeline**, not a single model artifact:

| Stage | Component | Technology |
|---|---|---|
| 1 | TransformerCRF token tagger (town/country spans) + country prediction head | PyTorch, char tokenizer, safetensors weights (~7 MB) |
| 2 | Fuzzy matching against town/country alias dictionaries | RapidFuzz, jellyfish |
| 3 | Postcode matching (per-country regex/dictionaries) | GeoNames postcode data |
| 4 | Rule-based post-processing and candidate reconciliation | Python |

**Key characteristics:**

- Entrypoint: `AddressStructuringPipeline` class; `DataFrameReader` accepts polars DataFrames directly.
- All paths configurable via environment variables with the `ds_` prefix (pydantic-settings), e.g. `ds_prefix_folder_path`, `ds_model_weights_path`.
- **No runtime downloads.** Everything is loaded from local disk at pipeline init; missing files cause init failure.
- Max input length: **224 characters** per address — validate/truncate upstream.
- Optimised for English/transliterated input; unreliable on Arabic, Chinese, Cyrillic, Japanese scripts. Validate low-confidence predictions.
- Requires **Python ≥ 3.12** and ~**4 GB RAM** (reference data is held in memory).

### Reference data caveat

The companion resources repo ships **only** the model weights, model config, and sample inputs. The GeoNames-derived datasets (towns parquet, alias JSONs, postcode dictionaries) are **not redistributed** — the `raw/` folders are empty placeholders. You must download the GeoNames dumps and run the included preprocessing scripts yourself (one-time, then periodic refresh).

---

## 2. Prerequisites

- Databricks workspace with **Unity Catalog** and **Model Serving** enabled.
- Cluster on **DBR 16.x or later** (Python 3.12 — hard requirement; logging from a 3.11 runtime will produce a broken serving container).
- A UC catalog/schema for the model, plus a **UC Volume** for libraries and reference data, e.g.:
  - `/Volumes/<catalog>/<schema>/libs/` — wheel
  - `/Volumes/<catalog>/<schema>/swift_addr/raw/` — pinned raw GeoNames dumps
  - `/Volumes/<catalog>/<schema>/swift_addr/resources/<version>/` — generated reference data + weights
- Permissions: `CREATE MODEL` on the schema, write on the Volumes, `CAN MANAGE` on serving endpoints.
- **Licensing review:** the repo uses Swift's own license (LICENSE.pdf/LICENSE.txt, not a standard OSS license) — review before internal hosting. GeoNames data is **CC-BY** (attribution required).

---

## 3. Phase 1 — Build the Reference Datasets (one-time + scheduled refresh)

Run on a DBR 16.x cluster (or a job). Outputs are written to a **versioned** Volume path so model versions pin to a specific reference-data snapshot.

```python
# Notebook: build_swift_addr_resources
import subprocess, datetime

VERSION = datetime.date.today().strftime("%Y-%m")
RAW     = "/Volumes/<catalog>/<schema>/swift_addr/raw"
OUT     = f"/Volumes/<catalog>/<schema>/swift_addr/resources/{VERSION}"

# 1. Clone code + resources repos
subprocess.run(["git", "clone", "--depth", "1",
    "https://github.com/Swift-SC/iso20022-address-structuring.git", "/tmp/code"], check=True)
subprocess.run(["git", "clone", "--depth", "1",
    "https://github.com/Swift-SC/iso20022-address-structuring-resources.git", "/tmp/res"], check=True)

# 2. Download raw dumps (pin copies into RAW for reproducible rebuilds)
#    - https://download.geonames.org/export/dump/allCountries.zip
#    - https://download.geonames.org/export/dump/alternateNamesV2.zip
#    - https://download.geonames.org/export/dump/countryInfo.txt
#    - https://download.geonames.org/export/zip/allCountries.zip   (postcodes)
#    - restcountries countriesV3.1.json
#    Follow the exact file list in the repo README "Installing the reference datasets".

# 3. Run preprocessing scripts from the code repo (paths per README)
#    preprocess_geonames_towns.py / _countries.py / _postcodes.py / preprocess_rest_countries.py
#    These generate the parquet/JSON files the pipeline reads at init.

# 4. Assemble final resources folder
#    OUT/
#      models/CRF_with_MLP_EPOCH_1.safetensors      (from resources repo)
#      models/CRF_with_MLP_EPOCH_1.config.json      (from resources repo)
#      <generated towns parquet, alias JSONs, postcode dicts, misc files>
```

**Refresh cadence:** GeoNames updates continuously. A quarterly job that regenerates `resources/<new-version>` and triggers re-registration keeps town/country data current.

---

## 4. Phase 2 — Build and Stage the Wheel

The repo has a complete `pyproject.toml` (package `swift_address_structuring`, v1.0.2). The wheel contains **code only** — no weights, no reference data.

```bash
git clone https://github.com/Swift-SC/iso20022-address-structuring.git
cd iso20022-address-structuring
pip install build
python -m build --wheel
# → dist/swift_address_structuring-1.0.2-py3-none-any.whl  (~210 KB, pure Python)

databricks fs cp dist/swift_address_structuring-1.0.2-py3-none-any.whl \
  dbfs:/Volumes/<catalog>/<schema>/libs/
```

The wheel declares all runtime dependencies (torch 2.8.0, polars 1.34.0, RapidFuzz, safetensors, …) in its metadata, so it can be the **single** pip requirement when logging the model. It also installs an `iso_run` CLI entrypoint, useful for smoke tests on a cluster.

> For supply-chain hygiene, mirror the repo into your internal Git (e.g. GHES) and build/publish the wheel from a source you control.

---

## 5. Phase 3 — Wrap, Log, and Register with MLflow

```python
import mlflow
import pandas as pd
import polars as pl
from mlflow.models import infer_signature

class SwiftAddressModel(mlflow.pyfunc.PythonModel):
    def load_context(self, context):
        import os
        res = context.artifacts["resources"]
        # Point the pipeline at bundled resources BEFORE importing the package
        os.environ["ds_prefix_folder_path"] = res
        os.environ["ds_model_weights_path"] = f"{res}/models/CRF_with_MLP_EPOCH_1.safetensors"
        os.environ["ds_model_config_path"]  = f"{res}/models/CRF_with_MLP_EPOCH_1.config.json"

        from data_structuring.pipeline import AddressStructuringPipeline
        self.pipeline = AddressStructuringPipeline()   # loads weights + reference DBs once

    def predict(self, context, model_input, params=None):
        from data_structuring.components.readers.dataframe_reader import DataFrameReader
        pdf = pl.from_pandas(model_input)
        reader = DataFrameReader(
            pdf,
            data_column_name="address",
            suggested_country_column="suggested_country",
            force_suggested_country_column="force_suggested_country",
        )
        results = self.pipeline.run(reader, batch_size=256)
        # Trim verbose CRF tensors from the serving payload
        return [r.model_dump(exclude={"crf_result": {"emissions_per_tag", "log_probas_per_tag"}})
                for r in results]


mlflow.set_registry_uri("databricks-uc")

example = pd.DataFrame({"address": ["10 DOWNING STREET LONDON SW1A 2AA UNITED KINGDOM"]})
RESOURCES = "/Volumes/<catalog>/<schema>/swift_addr/resources/<version>"
WHEEL     = "/Volumes/<catalog>/<schema>/libs/swift_address_structuring-1.0.2-py3-none-any.whl"

with mlflow.start_run(run_name="swift-addr-structuring"):
    mlflow.pyfunc.log_model(
        artifact_path="model",
        python_model=SwiftAddressModel(),
        artifacts={"resources": RESOURCES},          # copied into the model package
        pip_requirements=[WHEEL],                    # wheel metadata pulls the rest
        signature=infer_signature(example),
        registered_model_name="<catalog>.<schema>.swift_address_structuring",
    )
```

Notes:

- `artifacts={"resources": ...}` copies the reference data **into** the logged model, so the serving container is **hermetic** — no Volume mount or network egress needed at inference time.
- Log from a **Python 3.12** runtime so the captured environment matches the package requirement.
- Tag the model version with the resource snapshot (e.g. `resources_version=2026-06`) for traceability.

---

## 6. Phase 4 — Create the Serving Endpoint

### Terraform

```hcl
resource "databricks_model_serving" "swift_addr" {
  name = "swift-address-structuring"

  config {
    served_entities {
      entity_name           = "<catalog>.<schema>.swift_address_structuring"
      entity_version        = "1"
      workload_size         = "Medium"   # ≥4 GB working set; Small is likely too tight
      workload_type         = "CPU"
      scale_to_zero_enabled = true       # set false if cold-start latency matters
    }

    traffic_config {
      routes {
        served_model_name   = "swift_address_structuring-1"
        traffic_percentage  = 100
      }
    }
  }
}
```

### Sizing & cold start

- Reference data + torch weights load in `load_context` → expect **minutes** of cold start when scaling from zero. For real-time payment-repair flows, keep `scale_to_zero_enabled = false`.
- Start with **Medium CPU**; right-size from endpoint metrics. GPU is unnecessary — the transformer is small and char-level.

---

## 7. Querying the Endpoint

```bash
curl -X POST "https://<workspace-url>/serving-endpoints/swift-address-structuring/invocations" \
  -H "Authorization: Bearer $DATABRICKS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "dataframe_records": [
      {"address": "10 DOWNING STREET LONDON SW1A 2AA UNITED KINGDOM"},
      {"address": "1 QUEEN ST AUCKLAND 1010 NZ", "suggested_country": "NZ"}
    ]
  }'
```

Input columns:

| Column | Required | Meaning |
|---|---|---|
| `address` | yes | Unstructured address text (≤ 224 chars) |
| `suggested_country` | no | Hint; biases candidate scoring |
| `force_suggested_country` | no | Hard constraint on country resolution |

Output (per record): resolved `town`, `country`, confidence scores, candidate/resolution options, and explainability fields from each pipeline stage.

```python
# Python client
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()
resp = w.serving_endpoints.query(
    name="swift-address-structuring",
    dataframe_records=[{"address": "1 QUEEN ST AUCKLAND 1010 NZ"}],
)
```

---

## 8. Batch Inference Alternative (recommended for bulk cleansing)

The model's primary use cases — cleaning client reference databases and repairing corporate-originated payment files — are **batch** workloads. For Delta-table-scale runs, load the registered model as a Spark UDF in a job instead of hammering the REST endpoint:

```python
import mlflow
from pyspark.sql import functions as F

mlflow.set_registry_uri("databricks-uc")

udf = mlflow.pyfunc.spark_udf(
    spark,
    model_uri="models:/<catalog>.<schema>.swift_address_structuring/1",
    result_type="string",     # adjust to your flattened output schema
)

df = spark.table("<catalog>.<schema>.customer_addresses")
out = df.withColumn("structured", udf(F.struct("address")))
out.write.mode("overwrite").saveAsTable("<catalog>.<schema>.customer_addresses_structured")
```

Rule of thumb: **endpoint** for interactive/real-time flows (payment repair UI, onboarding validation); **spark_udf job** for backfills and periodic database cleansing.

---

## 9. Operational Checklist

- [ ] Legal review of Swift license terms before internal hosting
- [ ] GeoNames CC-BY attribution recorded (e.g. in NOTICES / internal docs)
- [ ] Raw GeoNames dumps pinned in a Volume (reproducible rebuilds)
- [ ] Quarterly resource-refresh job + model re-registration
- [ ] Input validation upstream: length ≤ 224 chars, script/charset checks
- [ ] Low-confidence predictions routed to manual review, not auto-applied
- [ ] Endpoint monitoring: latency, memory, cold-start frequency
- [ ] Model version tagged with resources snapshot version
- [ ] Wheel built from internally mirrored source (supply-chain)

---

## 10. Quick Reference

| Item | Value |
|---|---|
| Package / version | `swift_address_structuring` 1.0.2 |
| Python | ≥ 3.12 (DBR 16.x+) |
| Key deps | torch 2.8.0, polars 1.34.0, safetensors 0.6.2, RapidFuzz 3.13.0 |
| Weights | `CRF_with_MLP_EPOCH_1.safetensors` (~7 MB, from resources repo) |
| Reference data | Generated from GeoNames dumps via repo preprocessing scripts |
| RAM | ≥ 4 GB (Medium CPU workload recommended) |
| Max address length | 224 characters |
| Runtime downloads | None — fully offline/hermetic at inference |
| CLI smoke test | `iso_run --input_data_path=... --verbose` |
