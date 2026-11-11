# Steam Promo Impact

**English** | [Português](README.pt-BR.md)

> **Do Steam promotions pay off for AAA RPG/Action publishers? For whom, when and how?**

Steam does not publish sales or revenue data. This project builds a data pipeline that collects public signals (prices and discounts, review counts, concurrent players, top-seller rankings and news/DLC events), models every promotion as an event, and **estimates its impact from the publisher's point of view**: sales uplift, estimated revenue against a counterfactual, and break-even.

> 🚧 **Work in progress.** Data collection is being built. See the [roadmap](#roadmap).

## Architecture

```
Steam APIs / IsThereAnyDeal / SteamSpy
  → Python ingestion (Databricks Lakeflow Jobs)  → raw JSON in a Unity Catalog Volume
  → Auto Loader                                  → bronze (Delta)
  → dbt                                          → silver / gold
  → Notebook and/or dashboard
```

## Tech stack

| Layer | Tools |
|---|---|
| Ingestion | Python 3.12, httpx, tenacity |
| Platform | Databricks Free Edition: Unity Catalog, Volumes, Delta Lake, Auto Loader, Lakeflow Jobs |
| Transformation | dbt Core + dbt-databricks |
| Engineering | uv, ruff, pre-commit, Databricks Asset Bundles, GitHub Actions |

## Roadmap

| Block | Scope | Status |
|---|---|---|
| A — Foundation | Repository, tooling, Databricks setup | 🔄 In progress |
| B — Know the data | API exploration, game cohort | ⏳ |
| C — Ingestion | Collectors, scheduling, historical backfill | ⏳ |
| D — Bronze | Auto Loader | ⏳ |
| E — dbt | Staging, price snapshot (SCD2), promotion events, CI | ⏳ |
| F — Analysis | Gold marts, hypothesis testing, final write-up | ⏳ |

Progress is tracked in the [issues](../../issues) and [milestones](../../milestones).

## Getting started

**Prerequisites:** [uv](https://docs.astral.sh/uv/), Git and a [Steam Web API key](https://steamcommunity.com/dev/apikey).

```bash
git clone https://github.com/arthurdurso/steam-promo-impact.git
cd steam-promo-impact

uv sync                        # creates .venv with Python 3.12 and dev tools
uv run pre-commit install      # enables the git hooks

cp .env.example .env           # then fill in your API keys
```

## Documentation

- [Product requirements (PRD)](docs/prd.md) (in Portuguese): scope, data sources, methodology, hypotheses and risks.

## A note on the data

All sales and revenue figures in this project are **estimates** built from public proxies, mainly review velocity and concurrent players. They are meant to compare promotions with each other, not to report real sales. The methodology and its limitations are documented in the PRD.
