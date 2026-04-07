# Data Engg Ops Notebooks — Wide CSV to Delta (Salesforce-style lab)

**What this is:** A small, deployable **Databricks Asset Bundle** plus two notebooks that simulate **many wide CSV exports** (different column counts, order, and header quirks), land them on a **Unity Catalog volume**, then **load them quickly into one Delta table** using Spark patterns suited to messy exports.

**Who it is for:** Data engineers and platform teams who need a **repeatable pattern** (and a runnable demo) for landing **heterogeneous CSV** files into a **canonical Delta** model.

## What problem this solves

| Challenge | How this asset helps |
|-----------|----------------------|
| Exports have **different column order** or **subset of columns** | Uses **`unionByName(allowMissingColumns=True)`** so alignment is by name, not position. |
| Headers do not always match logical field names | A **manifest** lists physical vs logical columns; the loader **renames** before union. |
| **Wide files** (hundreds of columns) make **schema inference** slow or wrong | CSV is read with **`inferSchema=false`** (strings), then projected to a fixed **500-column** model. |
| You need **catalog/schema** to vary by environment | **`databricks.yml` variables** (`catalog`, `schema`, `volume_name`) flow into the job and notebook **widgets**—no hardcoding in code for production paths. |
| **ANSI / serverless** arithmetic edge cases | Generator uses **pmod without abs(hash)** to avoid integer overflow on edge hash values. |

## What you get

- **Notebook 1:** Creates schema and volume (if needed), writes CSV files and `manifest/file_manifest.json` under `/Volumes/<catalog>/<schema>/<volume>/`.
- **Notebook 2:** Reads the manifest, loads all CSVs, unions, aligns to the canonical column list, **appends** to Delta table `<catalog>.<schema>.salesforce_account`.
- **Job:** Runs notebook 1 then notebook 2 in order (serverless-compatible: **no classic cluster** defined in YAML).

## How to run

### Prerequisites

- Databricks workspace with **Unity Catalog** and permission to create **schema**, **volume**, and **tables** in your catalog.
- **Databricks CLI** v0.200+ with auth: `databricks auth login --host https://<your-workspace>.cloud.databricks.com`
- Optional: copy `databricks.local.example.yml` → **`databricks.local.yml`** (gitignored) to set `workspace.host` or variable overrides for your machine.

### Configure catalog and schema

Edit **`databricks.yml`** `variables` defaults, or override at deploy time:

```bash
databricks bundle deploy -t dev --var catalog=YOUR_CATALOG --var schema=YOUR_SCHEMA --var volume_name=YOUR_VOLUME
```

### Deploy and run the job

```bash
cd Data-Engg-Ops-Notebooks
databricks bundle deploy -t dev -p YOUR_PROFILE
databricks bundle run sf_account_wide_csv_lab -t dev -p YOUR_PROFILE
```

### Run notebooks interactively

Import or open the notebooks from your workspace (after deploy), set **widgets** to your catalog, schema, and volume, then run **generate** before **load**.

## Files in this repo

| Path | Use |
|------|-----|
| `databricks.yml` | Bundle name, variables (`catalog`, `schema`, `volume_name`), targets |
| `databricks.local.example.yml` | Template for **local-only** overrides; copy to `databricks.local.yml` (not committed) |
| `resources/jobs.yml` | Two-task job wiring and `base_parameters` |
| `src/notebooks/01_*.ipynb` | Generate CSVs + manifest |
| `src/notebooks/02_*.ipynb` | Load to Delta |
| `SETUP.md` | Prerequisites, customization, and known limits |

## References

- [Databricks Asset Bundles](https://docs.databricks.com/dev-tools/bundles/index.html)
- [Unity Catalog volumes](https://docs.databricks.com/en/volumes/index.html)

## Author

Published by [Archana Krishnamurthy](https://github.com/archana986) for reusable data-engineering patterns.
