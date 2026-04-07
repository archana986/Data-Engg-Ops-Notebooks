# Data-Engg-Ops-Notebooks

Customer-facing **Databricks** patterns and bundles for data engineering and platform work. Each top-level folder is a self-contained asset (bundle + notebooks + docs).

## Contents

| Folder | Summary |
|--------|---------|
| [`multi-schema-csv-to-delta/`](multi-schema-csv-to-delta/) | Load **many CSV files with different schemas** into **one canonical Delta** table: **`inferSchema=False`**, **`unionByName(allowMissingColumns=True)`**, batched unions, single canonical projection, append + repartition for write fan-out; **`file_manifest.json`** maps physical headers to logical names. See that folder’s **README** for the full design. |

## Using a bundle

Always `cd` into the bundle directory before `databricks bundle` commands:

```bash
cd multi-schema-csv-to-delta
databricks bundle validate -t dev -p YOUR_PROFILE
databricks bundle deploy -t dev -p YOUR_PROFILE
databricks bundle run multi_schema_csv_to_delta -t dev -p YOUR_PROFILE
```

Local workspace overrides: copy `databricks.local.example.yml` to **`databricks.local.yml`** inside that folder (gitignored).

## Author

[Archana Krishnamurthy](https://github.com/archana986)
