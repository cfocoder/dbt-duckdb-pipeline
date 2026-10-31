# dbt + DuckDB Job Postings Pipeline

A simple ELT pipeline using [dbt](https://www.getdbt.com/) and [DuckDB](https://duckdb.org/) to analyze job posting data. The pipeline ingests raw job posting parquet files, cleans and transforms them, and produces analytics-ready tables. It's configured to run automatically via GitHub Actions and publishes the resulting DuckDB warehouse as a GitHub Release.

## Overview

This project processes scraped job posting data for data/analytics roles (search terms like Data Analyst, Data Engineer, Data Scientist) across a 12-month period (July 2025 - June 2026). It creates models for:
- Cleaned job postings
- Monthly job posting counts by role
- Top 10 job locations by posting volume
- Monthly summary with remote work percentage

## Project Structure

```
.
├── analytics/           # dbt project
│   ├── dbt_project.yml
│   ├── models/          # dbt models (SQL)
│   │   ├── sources.yml
│   │   ├── schema.yml   # Model and column descriptions + tests
│   │   ├── job_postings.sql
│   │   ├── jobs_per_month.sql
│   │   ├── monthly_summary.sql
│   │   └── top_locations.sql
│   ├── seeds/           # Seed data (if any)
│   └── target/          # Compiled artifacts (gitignored in practice)
├── ci/
│   └── profiles.yml     # CI profile pointing to data/prod.duckdb
├── data/
│   ├── dev.duckdb       # Local development database
│   ├── prod.duckdb      # Production database (built in CI)
│   └── raw/             # Raw parquet files (downloaded separately)
├── scripts/
│   └── download_data.py # Script to download dataset
├── .github/workflows/
│   └── dbt-build.yml    # CI pipeline
├── pyproject.toml
└── main.py
```

## Setup

### Prerequisites

- Python >= 3.13
- [uv](https://docs.astral.sh/uv/) (recommended) or pip

### Installation

1. Clone the repository:
```bash
git clone <repo-url>
cd dbt-duckdb-pipeline
```

2. Install dependencies:
```bash
uv sync
```

3. Download the raw dataset:
```bash
uv run python scripts/download_data.py
```
This will download parquet files from the GitHub release to `data/raw/`. The script skips files that already exist.

## Usage

### Local Development

The dbt project is in the `analytics/` directory. Run commands from there or specify the profile.

**Build all models and run tests:**
```bash
cd analytics
uv run dbt build
```

**Run models only:**
```bash
cd analytics
uv run dbt run
```

**Run tests:**
```bash
cd analytics
uv run dbt test
```

**Generate and serve docs:**
```bash
cd analytics
uv run dbt docs generate
uv run dbt docs serve
```

### Data Lineage

The models follow this lineage:
1. **Source**: `raw.job_postings` (external parquet files from `data/raw/raw_job_postings_*.parquet`)
2. **Staging/Cleaning**: `job_postings` (view) - filters out rows with errors, selects key columns
3. **Aggregations**:
   - `jobs_per_month` (table) - counts by month and role
   - `top_locations` (table) - top 10 locations by posting count
   - `monthly_summary` (table) - monthly totals with remote work percentage

## Models

| Model | Materialization | Description |
|---|---|---|
| `job_postings` | view | Cleaned job postings (filters null/error rows) |
| `jobs_per_month` | table | Monthly posting counts grouped by role (`search_term`) |
| `top_locations` | table | Top 10 locations by posting volume (excludes null locations) |
| `monthly_summary` | table | Monthly totals with remote posting counts and percentage |

## Data Tests

Tests are defined in `analytics/models/schema.yml`:
- `job_postings.job_title` - not null
- `top_locations.job_location` - not null, unique
- `monthly_summary.month` - not null, unique

## CI/CD

The GitHub Actions workflow (`.github/workflows/dbt-build.yml`) runs:
1. On push to `master`
2. Weekly on Mondays at 06:17 UTC (cron: `17 6 * * 1`)
3. Manually via workflow_dispatch

Steps:
- Checkout code, setup uv, install dependencies
- Download data with `scripts/download_data.py`
- Run `dbt build --profiles-dir ../ci` from the `analytics/` directory
- Create/update GitHub release named `warehouse` and upload `data/prod.duckdb`

The CI uses `ci/profiles.yml` which writes to `../data/prod.duckdb` (so `data/prod.duckdb` from repo root).

## Database

- **DuckDB** is used as the data warehouse
- Development DB: `data/dev.duckdb` (local)
- Production DB: `data/prod.duckdb` (built in CI, published as release asset)

## Notes

- Raw data is sourced from [lukebarousse/dbt_Analytics_Engineering_Course](https://github.com/lukebarousse/dbt_Analytics_Engineering_Course) (release `dataset-v1`)
- The `job_postings` source uses DuckDB's `read_parquet()` with a glob pattern to read all monthly files
- The pipeline processes ~12 months of job posting data across multiple search terms
