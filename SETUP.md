# SETUP — Data Engg Ops Notebooks bundle

## How to use this bundle in your own workspace

1. Clone the repository.
2. Authenticate the Databricks CLI against **your** workspace.
3. Set **`catalog`**, **`schema`**, and **`volume_name`** in `databricks.yml` (or pass `--var` at deploy time).
4. Run `databricks bundle deploy` then `databricks bundle run sf_account_wide_csv_lab`.

## Prerequisites

- Unity Catalog **USE CATALOG** on the target catalog.
- Ability to **CREATE SCHEMA**, **CREATE VOLUME**, **CREATE TABLE** (or equivalent grants).
- Workspace that supports **serverless** or default job compute for **notebook** tasks (region and policy dependent).

## Customize

| Setting | Where |
|---------|--------|
| Catalog / schema / volume | `databricks.yml` → `variables` |
| Row count per CSV file | Notebook 1 → `ROWS_PER_FILE` |
| Delta output file count | Notebook 2 → `DELTA_TARGET_FILES` |
| Workspace host (optional) | `databricks.local.yml` (create from `databricks.local.example.yml`) |

## Local overrides (not in git)

- **`databricks.local.yml`** is **gitignored**. Use it for personal `workspace.host` or per-developer variable overrides.
- Do **not** commit workspace URLs, tokens, or account-specific catalog names you consider sensitive—use `databricks.local.yml` or CI variables.

## Known limitations

- **Scale:** Default `ROWS_PER_FILE` is modest for a quick demo; raising toward **500k × 50 files** increases runtime and volume storage.
- **Manifest size:** The manifest lists every column name per file; the notebooks read the **full** JSON via `dbutils.fs.open` (not `head`) to avoid truncation.
- **Permissions:** The bundle does **not** grant IAM-style permissions to other users; adjust in the workspace after deploy if needed.
- **Compute:** If your workspace **disallows** serverless jobs, you may need to add an explicit job cluster in `resources/jobs.yml` (not included by default).

## Validate the bundle

```bash
databricks bundle validate -t dev -p YOUR_PROFILE
```

If validation fails, check CLI version and that your profile `host` matches the intended workspace.
