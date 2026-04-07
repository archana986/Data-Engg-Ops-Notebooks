# Multi-schema CSV → Delta

**What this is:** A deployable **Databricks Asset Bundle** plus two notebooks that simulate **many CSV exports with different schemas** (column counts, order, and header quirks), land them on a **Unity Catalog volume**, then **load them into one Delta table** using Spark patterns suited to heterogeneous files.

**Who it is for:** Data engineers loading **multiple CSVs that do not share one fixed layout** who want a **repeatable pattern** (and runnable demo) for a **canonical Delta** model.

## What problem this solves

| Challenge | How this asset helps |
|-----------|----------------------|
| Exports have **different column order** or **subset of columns** | Uses **`unionByName(allowMissingColumns=True)`** so alignment is by name, not position. |
| Headers do not always match logical field names | A **manifest** lists physical vs logical columns; the loader **renames** before union. |
| **Wide files** (hundreds of columns) make **schema inference** slow or wrong | CSV is read with **`inferSchema=false`** (strings), then projected to a fixed **500-column** model. |
| You need **catalog/schema** to vary by environment | **`databricks.yml` variables** (`catalog`, `schema`, `volume_name`) flow into the job and notebook **widgets**—no hardcoding in code for production paths. |
| **ANSI / serverless** arithmetic edge cases | Generator uses **pmod without abs(hash)** to avoid integer overflow on edge hash values. |

## PySpark choices for performance and stability

The speed story here is mostly **cheap reads**, a **stable schema**, a **sane plan shape**, and **controlled write fan-out** — not a single magic API.

| Technique | What it does | Why it helps |
|-----------|--------------|--------------|
| **`inferSchema=False` on CSV** | Every column is read as **string**. | Schema inference on **400–500 columns × many files** is slow and can guess wrong. One cheap, predictable read path. |
| **`unionByName(allowMissingColumns=True)`** | Aligns columns **by name**, not by position. | Files can have different column order and different subsets of fields without fragile “column 37 = ?” logic. |
| **Batched unions** (e.g. 10 files → union → repeat) | Build several smaller unions, then union those results. | Avoids a single giant lineage tree of many nested unions, which can stress the driver and slow analysis/optimization. |
| **One `select` to the canonical column list** | After union, project once; missing names become **null**. | Prunes the wide working set to exactly what Delta needs; keeps the write plan simple. |
| **`repartition(DELTA_TARGET_FILES)` before Delta write** | Control how many output files you write. | Avoids thousands of tiny Delta files (bad for later reads) or one huge file (bad for parallelism). |
| **Append (no MERGE)** | Insert-only write. | No join/shuffle for match/upsert; fastest write path if you do not need dedupe. |

Tune `DELTA_TARGET_FILES` and batch sizes in **notebook 2** (and row counts in notebook 1) for your workspace and file counts.

## What the manifest is for

The manifest (`manifest/file_manifest.json`) is **per-file metadata** generated alongside the CSVs. For each file it records things like:

- **`volume_path`**: where that CSV lives.
- **`columns`**: physical header names as they appear in the file (after quirks like `_BLANK_HDR_*`).
- **`logical_columns`**: the real Salesforce-style names those columns represent.

The loader uses this in **`read_and_normalize`**: **rename physical → logical** before **`unionByName`**, so every DataFrame speaks the same column names as your **canonical** schema.

Without that step, a CSV whose header is a placeholder (e.g. `_BLANK_HDR_12`) would never match the intended logical field (e.g. `Segment_Score_042__c`), and you would lose or mis-align data in the union.

**The manifest is not there to speed up I/O** — it makes header anomalies and intentional renames **correct and automatic**.

## Manifest + `unionByName` vs “read each file, different headers”

Spark still reads many paths; the difference is what happens **after** each read:

- **Without a manifest** (and with bad or placeholder headers), you either manually map each file’s headers to your model, or you accept default names (`_c0`, `_c1`, …) and then you cannot `unionByName` on real API field names without a lot of custom code per file.
- **With manifest + `unionByName`**, you normalize names once per file from the manifest, then one uniform union and one canonical projection — **the same code path for every file**.

**Different headers and order alone** are handled well by **`unionByName`** when the header text **equals** the logical API name. The **manifest** covers the case where the on-disk header is wrong or a placeholder but you still know the true field name — that improves **correctness** and **maintainability**, not raw read throughput.

**In one line:** PySpark speedups = string-only CSV + batched unions + single canonical projection + append + controlled repartition. The manifest maps weird physical headers to logical columns so the union and Delta table stay correct; it **complements** `unionByName`, it does not replace it.

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
cd multi-schema-csv-to-delta
databricks bundle deploy -t dev -p YOUR_PROFILE
databricks bundle run multi_schema_csv_to_delta -t dev -p YOUR_PROFILE
```

### Run notebooks interactively

Import or open the notebooks from your workspace (after deploy), set **widgets** to your catalog, schema, and volume, then run **generate** before **load**.

## Files in this folder

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
