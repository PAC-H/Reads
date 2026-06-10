# Swift Hybrid Postal Address Structuring — Databricks Deployment & Usage Guide (v2)

> **v2 — updated against the official *Swift Hybrid Postal Address Structuring Installation & User Guide*, release 1.0.0 (01 December 2025).**
>
> Sources: [Swift-SC/iso20022-address-structuring](https://github.com/Swift-SC/iso20022-address-structuring) (GitHub) · Swift.com download centre archives · official Installation & User Guide PDF.

---

## 0. What Changed in v2 (vs. the original deployment doc)

| # | Change | Why |
|---|---|---|
| 1 | **Pyfunc wrapper rewritten to use the documented `DataStructuring` API** (list-of-strings input + 4 config objects) instead of internal `AddressStructuringPipeline`/`DataFrameReader` classes | The official guide documents `DataStructuring` as the supported programming interface; internal classes may change without notice |
| 2 | Added **official distribution channels**: Swift.com download centre ships two independent archives (codebase + reference data/trained model) with different licensing | Affects sourcing strategy and legal review scope |
| 3 | Added **mandatory folder layout** (`resources/` must sit alongside the codebase contents) | Misplaced resources is the most likely cause of init failure |
| 4 | Added **exact reference dataset file lists** per raw folder | Removes guesswork from the build-resources job |
| 5 | Added **input contract**: MT 103 field 50/59 formatting, 35-char line limit, dummy account/name lines, romanisation guidance | Input formatting materially affects accuracy — must be enforced upstream of the endpoint |
| 6 | Added **full output schema** (business columns + debug columns) and `NO COUNTRY` semantics | Needed to define the serving response contract and downstream Delta schema |
| 7 | Added **post-deployment validation step** using the bundled benchmark datasets and Swift's published accuracy figures | Objective "is it set up correctly" gate for CI/CD |
| 8 | Added **model configuration parameters** (`town_minimal_population`, `force_countries`, `enable_extended_data`, `show_inferred_country`, …) | Tuning levers for accuracy vs. speed; relevant for ASB-specific corridors |
| 9 | Added **scaling guidance**: model is stateless and horizontally scalable; CPU-optimised; reference-data size trades speed vs. accuracy | Confirms endpoint sizing strategy and adds a speed-tuning lever |
| 10 | Added **lifecycle & support caveats**: provided as-is with no support; Swift expects the model to be obsolete by **end of 2027** | Plan decommissioning; don't build long-lived dependencies on it |
| 11 | Noted **version discrepancy**: official guide covers release 1.0.0; the GitHub repo is at 1.0.2 | Pin and record exactly which release you deploy |

---

## 1. What the Model Is

An **offline-runnable, lightweight NLP system** (deliberately *not* an LLM — a few million parameters) that infers structured **Town** and **Country** from unstructured legacy address content found in client databases and corporate payment files, supporting the ISO 20022 migration and the November 2026 CBPR+ structured/hybrid address requirement.

Three sequential components plus a post-processor:

| Stage | Component | Notes |
|---|---|---|
| 1 | **CRF + Transformer** (char-level tokens, BIO tagging, order-2 CRF) + a 3-layer **MLP country predictor** on a CLASS token | Trained on synthetic data generated from anonymised payment templates (Faker, OpenStreetMap, GeoNames) |
| 2 | **Fuzzy matcher** against the reference dataset | Max Levenshtein distance 1 (one typo tolerated) |
| 3 | **Postcode regex matcher** | Exact match against postcode reference data |
| 4 | **Rule-based bonus/malus post-processing** | Flags (e.g. `IS_IN_LAST_THIRD`, `IS_SHORT`, `MLP_STRONGLY_AGREES`) adjust scores; top-2 combinations output |

**Operational profile:** Python ≥ 3.12, PyTorch ≥ 2.8.0, ~4 GB RAM, CPU-optimised (GPU optional speed-up), fully offline at inference — no internet/API connectivity required. Stateless, so it scales horizontally without diminishing returns.

---

## 2. Sourcing & Licensing (updated)

Two **independently distributed** archives (Swift.com download centre; the codebase is also on GitHub):

| Archive | Contents | License |
|---|---|---|
| **Codebase** (`swift_address_structuring-<ver>.tar.gz`) | Model source code, owned by Swift | Permissive open-source-type license |
| **Reference data / trained model** (`swift_address_structuring_resources-<ver>.tar.gz`) | Trained weights, misc files, benchmark inputs, empty `raw/` folders | Based on open-source reference data; **each dataset carries its own license**; Swift claims no ownership |

Action items for ASB:

- Review the license and installation notice **in each archive** separately (code vs. data have different terms).
- GeoNames is CC-BY — record attribution in internal docs/NOTICES.
- The solution is provided **"as is", with no guarantee or support** — it is explicitly not a Swift product/service. Feedback only via Swift.Address.Structuring@swift.com.
- **Version pinning:** the official guide documents release **1.0.0**; the public GitHub repo is at **1.0.2**. Decide which release is your deployment baseline, record it in the model version tags, and diff release notes when they diverge.
- **Sunset planning:** Swift assumes the model will be **obsolete by end of 2027** (post-migration). Treat the endpoint as a transitional capability with a planned decommission date.

---

## 3. Required Folder Layout (mandatory)

The resources archive contents **must** live in a folder named exactly `resources`, sitting alongside the codebase contents:

```
project/
├── data_structuring/          ← codebase package
├── pyproject.toml
├── requirements.txt
├── resources/                 ← resources archive contents, merged here
│   ├── input/                 ← benchmark CSVs (gauntlet, wikipedia)
│   ├── misc/                  ← country mappings; country_specs.json generated here
│   ├── models/                ← CRF_with_MLP safetensors + config
│   ├── post_codes/            ← generated by postcode preprocessing
│   └── raw/
│       ├── geonames/          ← you download these (see §4)
│       ├── postcodes/         ← you download these
│       └── restCountries/     ← you download this
└── ...
```

When bundling into MLflow, replicate this layout inside the model artifacts and point the `ds_` env vars (or `DatabaseConfig`) at it.

---

## 4. Building the Reference Datasets (exact file lists)

No runtime downloads occur — all of this is a **one-time offline preparation** (plus periodic refresh). Run on a DBR 16.x cluster, writing outputs to a versioned UC Volume.

### 4.1 GeoNames town/country dataset → `resources/raw/geonames/`

Download from the GeoNames exports page, extract, and place:

- `allCountries.zip` (extracted)
- `countryInfo.txt`
- `alternateNamesV2.zip` (extracted)

Then run:

```bash
python data_structuring/preprocessing/preprocess_geonames_towns.py
python data_structuring/preprocessing/preprocess_geonames_countries.py
```

Output paths are read from the database configuration in `config.py`; generated files land in `resources/`.

### 4.2 GeoNames postal codes dataset → `resources/raw/postcodes/`

Download from the GeoNames **postal codes** exports page, extract, and place:

- `allCountries.zip` (extracted)
- `CA_full.csv.zip` (extracted)
- `GB_full.csv.zip` (extracted)
- `NL_full.csv.zip` (extracted)

Then run:

```bash
python data_structuring/preprocessing/preprocess_geonames_postcodes.py
```

Generates the postcode matching files in `resources/post_codes/`.

### 4.3 REST Countries dataset → `resources/raw/restCountries/`

Download `countriesV3.1.json`, place it in the folder, then run:

```bash
python data_structuring/preprocessing/preprocess_rest_countries.py
```

Generates `country_specs.json` in `resources/misc/` (used for matching website domains and phone numbers).

> **Refresh cadence:** GeoNames updates continuously; Swift notes benchmark accuracy can drift slightly with reference data updates. Pin the raw dumps in a Volume for reproducible rebuilds; refresh quarterly and re-run the §8 validation gate.

---

## 5. Input Contract (new — enforce upstream of the endpoint)

Accuracy depends heavily on input shape. The CRF is extensively trained on **MT 103 field 50/59** party structures.

**Format rules:**

1. Tabular input uses a column labelled **`address`**, one address string per row. Multi-line addresses use embedded newlines (escape them in JSON payloads — standard `\n` in strings is fine).
2. If town, postal code, and/or country are known, place them on the **last line, in that order**.
3. If account number and account holder name are known, put them on the **first and second lines** respectively. If unknown, use the dummy placeholders Swift used in training for consistency:

   ```
   XX012345678901234567
   MARIA KUMAR
   <address line>
   <address line, with TOWN POSTCODE COUNTRY last>
   ```

4. **Max 35 characters per address line** (per the MT standard). Split longer lines at the limit and continue on the next line — the guide ships a reference `split_every_35th_char` implementation worth porting into your ingestion step.
5. **Romanise non-Latin scripts** (Arabic, Chinese, Cyrillic, Japanese) before submission, e.g. with `anyascii`. The model is unreliable on raw non-Latin input.
6. If input data is **partially structured**, merge the known structured elements into the unstructured string per rules 2–3 rather than submitting the unstructured fragment alone.

Ideal input example:

```
1234567890
JOHN DOE
42 MAIN ST A 5TH AVE
NEW YORK, NY 10001, US
```

**Recommendation for ASB:** implement these rules as a deterministic pre-processing function (Spark/Python) in the ingestion layer, so both the endpoint and batch paths receive normalised input. Do not rely on callers to format correctly.

---

## 6. MLflow Pyfunc Wrapper (updated — documented API)

The officially documented programming interface is the **`DataStructuring`** class taking a **list of address strings** and four config objects. Use it instead of internal pipeline classes:

```python
import mlflow
import pandas as pd
from mlflow.models import infer_signature

class SwiftAddressModel(mlflow.pyfunc.PythonModel):
    def load_context(self, context):
        import os
        res = context.artifacts["resources"]
        # Ensure the package resolves the bundled resources folder.
        # The ds_-prefixed env vars (pydantic-settings) relocate all paths;
        # alternatively pass a customised DatabaseConfig below.
        os.environ["ds_prefix_folder_path"] = res

        from data_structuring.run import DataStructuring
        from data_structuring.config import (CRFConfig, FuzzyMatchConfig,
                                             PostProcessingConfig, DatabaseConfig)

        self.ds = DataStructuring(
            crf_config=CRFConfig(),
            fuzzy_match_config=FuzzyMatchConfig(),
            post_processing_config=PostProcessingConfig(),
            database_config=DatabaseConfig(),
        )
        from data_structuring.components.Runners import ResultPostProcessing
        self._post = ResultPostProcessing

    def predict(self, context, model_input, params=None):
        debug = bool(params.get("debug", False)) if params else False
        addresses = model_input["address"].tolist()

        results = self.ds.run(addresses, batch_size=1024, show_progress=False)

        # Flatten to the documented human-readable schema (1th_/2th_ columns).
        final_df, _ = self._post.save_list_as_human_readable_csv(
            results, file_name="/tmp/ds_output.csv", debug=debug)
        return final_df


mlflow.set_registry_uri("databricks-uc")

example = pd.DataFrame({"address": [
    "XX012345678901234567\nMARIA KUMAR\n42 MAIN ST A 5TH AVE\nNEW YORK, NY 10001, US"
]})

with mlflow.start_run(run_name="swift-addr-structuring"):
    mlflow.pyfunc.log_model(
        artifact_path="model",
        python_model=SwiftAddressModel(),
        artifacts={"resources": "/Volumes/<catalog>/<schema>/swift_addr/resources/<version>"},
        pip_requirements=["/Volumes/<catalog>/<schema>/libs/swift_address_structuring-1.0.2-py3-none-any.whl"],
        signature=infer_signature(example),
        registered_model_name="<catalog>.<schema>.swift_address_structuring",
    )
```

Notes:

- Log from a **Python 3.12** runtime (DBR 16.x+).
- `params={"debug": true}` toggles the explainability columns at request time; per the guide it does **not** affect runtime, only output width.
- Verify the exact import paths and the `save_list_as_human_readable_csv` signature against the release you pin — the guide documents release 1.0.0; check 1.0.2 for drift.
- Tag the model version with both the **code release** and the **resources snapshot** (e.g. `code=1.0.2`, `resources=2026-06`).

Serving endpoint creation (Terraform `databricks_model_serving`, Medium CPU, scale-to-zero per latency needs) is unchanged from v1. The guide confirms the model is **stateless** and **CPU-optimised**, so horizontal scaling via endpoint concurrency is safe; GPU is optional and unnecessary at typical volumes.

---

## 7. Output Schema (new — serving response / Delta contract)

### Business columns (always present)

| Column | Description |
|---|---|
| `address` | Original input string |
| `1th_best_country` | Top country prediction, as spelled in the input |
| `1th_best_country_confidence` | Confidence, string percent (e.g. `'98.25%'`) |
| `1th_best_country_resolved_code` | Resolved ISO 3166-1 alpha-2 code |
| `1th_inferred_country_resolved_code` | Country code inferred **from the town** prediction (if `show_inferred_country` enabled) |
| `1th_best_town` | Top town prediction, as written in the address |
| `1th_best_town_confidence` | Confidence, string percent |
| `1th_best_town_resolved` | Town name as spelled in the reference database |
| `2th_*` | Second-best combination, same fields |

Semantics to handle downstream:

- **`NO COUNTRY`** appears as the resolved code when a combination has a town but no explicit country; a low fixed confidence (e.g. `15.0%`) is assigned. The inferred-country column may still carry a useful code (derived from the town).
- The same candidate can repeat across 1st/2nd combinations with identical scores.
- Confidence values are **percent strings**, not floats — cast in your Delta schema (`DOUBLE` after stripping `%`).

### Debug columns (only with `debug=true`)

`detailed_country_matches`, `detailed_town_matches` (candidates with flags/scores), `crf_prediction_country`, `crf_prediction_town`, `crf_prediction_postal_code`, `crf_spans` (token-level tags), `country_head_prediction` + `country_head_confidence` (MLP), `ibans` (detected IBANs).

**Routing rule (from Swift best practices):** automate on high-confidence predictions; route low-confidence results to manual review. Periodically review debug output for recurring ambiguities — they usually indicate reference-data gaps for specific geographies.

---

## 8. Post-Deployment Validation (new — CI/CD gate)

The resources archive ships two benchmark datasets in `resources/input/` for confirming correct setup:

| Dataset | Records | Expected accuracy (country / town / combined) |
|---|---|---|
| `addresses_gauntlet.csv` (Swift handcrafted hard cases) | 864 | 0.853 / 0.785 / 0.694 |
| Wikipedia address formats (65 countries × 3 variants) | 195 | 0.833 / 0.594 / 0.517 |

Add a validation notebook/job to the registration pipeline: run both benchmark files through the registered model, compute accuracy against the label columns, and fail the deployment if results deviate materially from the table above (small deviations are expected when reference data has been refreshed). These benchmarks are deliberately harder than real payment data — production accuracy on regular payment party fields is expected to be significantly higher.

---

## 9. Model Configuration & Tuning (new)

Key parameters (set via the config objects at logging time, or exposed as pyfunc params if you need per-request control):

| Parameter | Effect | Trade-off |
|---|---|---|
| `enable_extended_data` | Adds the OpenStreetMap dataset for more candidates | Slower inference |
| `town_minimal_population` | Filters reference DB by town population | Speed ↑, recall ↓ for small towns |
| `force_countries` | Full reference DB for listed countries regardless of population threshold | Targeted recall (e.g. `["NZ", "AU", "CN", "IN"]` for ASB corridors) |
| `force_country_groupings` | Same, by grouping (e.g. Europe, EMEA) | As above, coarser |
| `show_inferred_country` | Emit inferred-country column when only a town is found | Extra signal for repair workflows |
| Flag bonus/malus weights | Post-processing score tuning | Advanced; defaults recommended |

**Performance lever:** if inference is too slow, reduce reference-data size via the population threshold — speed scales proportionally, at an accuracy cost. Prefer horizontal scaling (more endpoint concurrency) first, since the model is stateless.

**Reference-data substitution:** if specific geographies underperform, the reference data can be amended or replaced with better regional datasets, provided the GeoNames format is preserved — a cleaner fix than tuning weights.

---

## 10. Operational Checklist (v2)

- [ ] Legal review of **both** archive licenses (code: permissive; data: per-dataset third-party terms) + GeoNames CC-BY attribution
- [ ] Deployment baseline pinned (guide = 1.0.0; GitHub = 1.0.2) and recorded in model tags
- [ ] Raw dumps pinned in a Volume; resources built per §4 with the exact file lists
- [ ] Input normalisation implemented upstream: `address` column, MT 103 50/59 shaping, 35-char line splitting, dummy account/name lines, `anyascii` romanisation
- [ ] Output contract handled: percent-string confidences cast, `NO COUNTRY` semantics, 1th/2th candidate logic
- [ ] Confidence-threshold routing: auto-apply vs. manual review queue
- [ ] Benchmark validation gate wired into the registration pipeline (§8)
- [ ] Quarterly reference refresh job + re-validation
- [ ] Endpoint monitoring (latency, memory, cold starts); horizontal scaling preferred over reference-data trimming
- [ ] **Decommission plan**: Swift expects obsolescence by end of 2027 — set a review date
- [ ] No support expectation set with stakeholders (provided as-is); security issues via Swift's vulnerability process
