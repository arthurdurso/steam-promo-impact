# Steam Promo Impact

[English](README.md) | **Português**

> **As promoções na Steam compensam para as publishers de jogos AAA de RPG/Ação? Para quem, quando e de que forma?**

A Steam não divulga dados de vendas nem de faturamento. Este projeto constrói um pipeline de dados que coleta sinais públicos (preços e descontos, número de reviews, jogadores simultâneos, ranking de mais vendidos e notícias/DLCs), modela cada promoção como um evento e **estima o impacto dela pela ótica da publisher**: aumento de vendas, receita estimada contra um contrafactual e ponto de equilíbrio.

> 🚧 **Em construção.** A coleta de dados está sendo desenvolvida. Veja o [roadmap](#roadmap).

## Arquitetura

```
APIs da Steam / IsThereAnyDeal / SteamSpy
  → Ingestão em Python (Databricks Lakeflow Jobs)  → JSON cru num Volume do Unity Catalog
  → Auto Loader                                    → bronze (Delta)
  → dbt                                            → silver / gold
  → Notebook e/ou dashboard
```

## Stack

| Camada | Ferramentas |
|---|---|
| Ingestão | Python 3.12, httpx, tenacity |
| Plataforma | Databricks Free Edition: Unity Catalog, Volumes, Delta Lake, Auto Loader, Lakeflow Jobs |
| Transformação | dbt Core + dbt-databricks |
| Engenharia | uv, ruff, pre-commit, Databricks Asset Bundles, GitHub Actions |

## Roadmap

| Bloco | Escopo | Status |
|---|---|---|
| A — Fundação | Repositório, ferramentas, setup do Databricks | 🔄 Em andamento |
| B — Conhecer os dados | Exploração das APIs, coorte de jogos | ⏳ |
| C — Ingestão | Coletores, agendamento, backfill histórico | ⏳ |
| D — Bronze | Auto Loader | ⏳ |
| E — dbt | Staging, snapshot de preço (SCD2), eventos de promoção, CI | ⏳ |
| F — Análise | Marts gold, teste das hipóteses, conclusões | ⏳ |

O progresso é acompanhado nas [issues](../../issues) e nos [milestones](../../milestones).

## Como rodar

**Pré-requisitos:** [uv](https://docs.astral.sh/uv/), Git e uma [API key da Steam](https://steamcommunity.com/dev/apikey).

```bash
git clone https://github.com/arthurdurso/steam-promo-impact.git
cd steam-promo-impact

uv sync                        # cria a .venv com Python 3.12 e as ferramentas de desenvolvimento
uv run pre-commit install      # ativa os hooks do Git

cp .env.example .env           # depois preencha as suas API keys
```

## Documentação

- [Documento de requisitos (PRD)](docs/prd.md): escopo, fontes de dados, metodologia, hipóteses e riscos.

## Sobre os dados

Todos os números de vendas e receita deste projeto são **estimativas** construídas a partir de sinais públicos, principalmente a velocidade de reviews e os jogadores simultâneos. Eles servem para comparar promoções entre si, não para informar vendas reais. A metodologia e as limitações estão documentadas no PRD.
