# Decision log

Lightweight ADRs: what was decided, why, and what it costs. Newest at the bottom of each section.

## Tooling

### D1. dbt Core 1.12 (not dbt platform, not dbt v2)
- **Decision:** run dbt Core 1.12 locally with the BigQuery adapter.
- **Why not dbt platform (Cloud):** it hides `profiles.yml`, targets and CI, which are exactly what this repo needs to show.
- **Why not v2:** v2 went GA on 2026-09-14. Most production projects today still run v1, the dbt Analytics Engineering certification targets 1.11, and authoring (SQL, YAML, Jinja, tests, snapshots) is the same in both. Compatibility with v2 is checked with `--use-v2-parser` before release.
- **Trade-off:** a future migration to v2 is still needed, but it mostly touches the runtime, not the models.
- **Gotcha:** `pip install dbt` now installs v2 (package `dbt`). v1 is installed as `dbt-core` + `dbt-bigquery`.

### D2. Python 3.13 in a uv-managed venv
- **Decision:** `uv venv --python 3.13 .venv`.
- **Why:** `dbt-bigquery 1.12.1` declares support up to Python 3.13 (PyPI classifiers). The machine default is 3.14, which is unsupported by the adapter.

### D3. Exact version pins in `requirements.txt`
- **Decision:** `dbt-core==1.12.6`, `dbt-bigquery==1.12.1`.
- **Why:** reproducible installs across my machine and CI. Core and adapter must share the same **minor** (1.12); their patch versions are independent.
- **Trade-off:** patch fixes don't arrive automatically; upgrading is a deliberate PR. Transitive dependencies still float. A lock file (`uv pip compile`) would close that gap.
- **How it's validated:** `uv pip install --dry-run -r requirements.txt` must report no changes.

## BigQuery

### D4. Billing enabled instead of the sandbox
- **Why:** the sandbox doesn't support DML (incremental `MERGE` and snapshots fail) and expires tables after 60 days.
- **Cost guard:** budget alert on the billing account (alerts only, it never blocks spend) plus `maximum_bytes_billed` in the profile (blocks the query).

### D5. All datasets in `US`
- **Why:** the public source `bigquery-public-data.thelook_ecommerce` lives in `US`, and BigQuery can't join across locations.

### D6. `maximum_bytes_billed` = 512 MiB per query
- **Context:** the four source tables total ~45 MB (users 19.8, order_items 13.6, orders 6.7, products 4.3).
- **Why 512 MiB:** ~11x the full dataset, so any legitimate query (even a full refresh) fits, while an accidental scan of a large table fails before it runs and costs nothing. Within the 1 TiB/month free tier that is 2,048 capped queries.
- **What it doesn't catch:** expensive compute with few bytes read (e.g., a cartesian join). That's what `job_execution_timeout_seconds: 300` is for.

### D7. OAuth via Application Default Credentials for local dev
- **Decision:** `method: oauth` using `gcloud auth application-default login`.
- **Why:** no long-lived key file on disk or near the repo. Service account keys are a leak risk; CI will use keyless auth instead.
- **Where:** `profiles.yml` lives in `~/.dbt/`, outside the repo.

## Data

### D8. Source: `thelook_ecommerce`, treated as live data
- **Why this dataset:** clear grains, real foreign keys for `relationships` tests, timestamps for incremental models.
- **Verified 2026-10-26:** all four tables were last modified the same day at ~02:52 UTC, so the dataset is regenerated daily (rewritten, not just appended). Whether historical rows change between days is checked during profiling and drives the incremental and snapshot design.
- **Known weakness:** it's synthetic and common in tutorials. Differentiators: decisions driven by profiling findings, cost guards, CI, SCD2, and a real external source (D9).

### D9. Real external source via Airflow: BCB PTAX exchange rate (planned)
- **Decision:** an Airflow DAG ingests the daily USD/BRL PTAX rate from the Banco Central public API into `raw_bcb.ptax`; dbt converts revenue to BRL.
- **Why:** a real, free, daily source that keeps roles clean (Airflow ingests, dbt transforms) and exercises idempotent loads, backfill, source freshness, and an as-of join for days without a rate (weekends, holidays).

## Repository

### D10. Secrets never reach the repo: three layers, `.gitignore` is the last one
- **Decision:** credentials are kept out of the repo by three layers, in this order of importance:
  1. **No key files exist.** Local dev uses OAuth/ADC (D7); CI will use keyless auth. A file that doesn't exist can't leak.
  2. **`.gitignore` as a safety net** in case someone creates one anyway: the GCP Console key name for this project (`thelook-dbt-512821-*.json`), generic key names (`*key*.json`, `*credentials*.json`), every `.env` variant (`.env*`) except the documented template (`!.env.example`), and the local `profiles.yml` anchored to the repo root (`/profiles.yml`) so `ci/profiles.yml` can still be committed.
  3. **Review before commit:** `git status` and `git diff --cached` on every commit.
- **Trade-off:** the wildcard patterns are deliberately broad and can ignore a legitimate file (e.g., `monkey.json` matches `*key*.json`). That false positive is cheap and visible: the file is missing from `git status`. A false negative leaks a key silently.
- **How it's validated:** `git check-ignore --no-index -v <paths>` checks paths without creating any file. The test has **positive cases** (key names and `.env.*` files that must be ignored) and **negative cases** (`.env.example`, `dbt_project.yml`, `package.json` and a non-key file with the project prefix, which must stay tracked). Testing only the positive cases would let an overly broad or name-specific rule pass.
- **Next:** run the same check as a script on every pull request in CI, so a change to `.gitignore` that weakens it fails the build.

