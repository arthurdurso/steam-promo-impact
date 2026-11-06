# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

Pre-scaffold: the only file so far is [docs/prd.md](docs/prd.md). It is the source of truth for scope, data sources, methodology and phases. Read it before doing substantial work. No build/lint/test commands exist yet; add them here once the repository skeleton is created.

The user writes in Portuguese (pt-BR). Reply and write project docs in Portuguese. Code identifiers and dbt model names stay in English.

## What this project is

A portfolio data project: **"Do Steam promotions pay off for AAA RPG/Action publishers?"** Steam publishes no sales data, so the pipeline collects proxies (price/discount, daily review counts, hourly concurrent players, top-seller rank, news/DLC events), models each promotion as an event, and estimates uplift, revenue vs. a counterfactual and break-even (`1 / (1 - discount)`).

Required stack: **Databricks (Free Edition) + dbt**. Everything has to run serverless within Free Edition quotas, so prefer incremental processing everywhere.

## Architecture (decided)

```
Steam APIs / IsThereAnyDeal / SteamSpy
  → Python ingestion (Lakeflow Job) writes raw JSON only
  → UC Volume  /Volumes/steam/bronze/raw/<source>/dt=YYYY-MM-DD/...
  → Auto Loader (cloudFiles, trigger availableNow) → steam.bronze.<table>
  → dbt (dbt-databricks, runs as a Job task) → steam.silver / steam.gold
  → analysis in a notebook and/or AI/BI dashboard (still undecided)
```

- **Strict ELT:** ingestion scripts only fetch and persist the full raw payload. They do no parsing, filtering or business logic. All transformation lives in dbt. Keep the complete JSON so history can be reprocessed.
- Use **Unity Catalog Volumes**, not DBFS paths.
- Unity Catalog layout: catalog `steam`, schemas `bronze`/`silver`/`gold`, plus `ci` for CI runs. The landing volume is `steam.bronze.raw`, and Auto Loader checkpoints go under `_checkpoints/`.
- Re-running a day must be idempotent (no duplicates).
- Scope is fixed to US/USD prices and a fixed cohort of ~300 paid AAA RPG/Action games (criteria in PRD §4.1).

## dbt conventions

- `stg_<source>__<entity>` (e.g. `stg_steam__prices`, `stg_itad__price_history`), `snap_*`, `int_*`, `dim_*`, `fct_*`, `mart_*`.
- Price history is an SCD2 snapshot (`snap_game_price`). `int_promotions` derives promotion events from the price series (gaps-and-islands).
- The reviews→sales multiplier (Boxleiter method) is a `dbt var`, not a hardcoded constant, because the analysis runs sensitivity checks at several values.
- Packages: `dbt_utils`, `dbt_expectations`. Sources declare `freshness`.

## Tooling decisions

- Python dependency management: **uv** (not Poetry/pip). Lint: **ruff** (Python), **SQLFluff** (dbt SQL). Hooks: pre-commit.
- Deployment: **Databricks Asset Bundles** (`databricks.yml`). CI: GitHub Actions running lint + `dbt build` against the `ci` schema.
- Deliberately excluded: Airflow/Dagster, Kafka/streaming, Terraform, Docker. Don't introduce them without discussing it first.

## External API constraints

- Store API (`store.steampowered.com/api/appdetails`) is unofficial and rate-limited to ~200 requests per 5 minutes. Use retries with backoff and batch appids with `filters=price_overview`.
- Several endpoints (`ISteamChartsService`, `IStoreTopSellersService`, the review histogram) are undocumented and still need validation in Phase 1 (PRD §5).
- Never scrape SteamDB, which its terms of use forbid. Use the IsThereAnyDeal API for historical prices.
- The Steam API key goes in Databricks Secrets / env vars.

## Timeline pressure

Ingestion (Phase 1) is the top priority. Collection has to be running before the Steam Autumn Sale (late Nov 2026) to capture a pre-sale baseline. Concurrent players and top-seller data cannot be backfilled.
