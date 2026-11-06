# PRD — Steam Promo Impact: as promoções compensam para as empresas?

| | |
|---|---|
| **Autor** | Arthur Durso |
| **Status** | Rascunho v1 |
| **Criado em** | 2026-11-06 |
| **Tipo** | Projeto de portfólio (Engenharia / Analytics Engineering) |
| **Stack obrigatória** | Databricks + dbt |

---

## 1. Contexto

Promoções são uma das principais alavancas comerciais na Steam. Publishers de jogos AAA decidem com frequência **quando**, **quanto** e **com que frequência** dar desconto. Mesmo assim, não existe dado público que mostre se essas decisões compensam financeiramente.

A Steam **não divulga vendas nem faturamento**. Ela expõe sinais indiretos: preço, número de reviews, jogadores simultâneos e rankings. Este projeto constrói um pipeline de dados que coleta esses sinais, modela as promoções como eventos e **estima o impacto de cada promoção pela ótica da empresa**.

## 2. Pergunta central

> **"As promoções na Steam compensam financeiramente para as empresas de jogos AAA de RPG/Ação? Para quem, quando e de que forma?"**

Toda análise e todo modelo do projeto devem servir a essa pergunta.

### Perguntas derivadas
1. O volume de vendas estimado durante a promoção cresce o suficiente para compensar o desconto (ponto de equilíbrio)?
2. Existe **ressaca pós-promoção**, ou seja, as vendas caem depois porque a promoção só antecipou compras?
3. Promoções sazonais da Steam (Summer, Autumn, Winter Sale) rendem mais ou menos que promoções próprias da publisher?
4. Os jogadores trazidos pela promoção **permanecem** jogando?
5. Como a profundidade do desconto evolui com o **tempo desde o lançamento**?
6. O lançamento de **DLC** ou de uma **sequência** muda o efeito da promoção no jogo base/antecessor?
7. Quais publishers têm a estratégia de desconto mais eficiente?

## 3. Objetivos

### Objetivos do produto (analíticos)
- Detectar automaticamente cada promoção (início, fim, profundidade, tipo).
- Estimar o uplift de vendas e a receita de cada promoção contra um baseline/contrafactual.
- Comparar estratégias de desconto entre publishers.

### Objetivos de portfólio
- Demonstrar um pipeline **ELT completo e agendado** em Databricks (arquitetura medallion + Unity Catalog).
- Demonstrar **modelagem com dbt**: snapshots (SCD2), modelos incrementais, testes, documentação e linhagem.
- Demonstrar boas práticas de engenharia: versionamento, CI, infraestrutura como código, gestão de segredos.
- Demonstrar **maturidade analítica**: hipóteses claras, uso honesto de proxies e limitações documentadas.

## 4. Escopo

### 4.1 Universo de jogos: RPG / Ação AAA
Coorte **fixa** de jogos, revisada em lotes (novos lançamentos entram periodicamente).

**Critérios de inclusão (a validar com dados na Fase 1):**
| Critério | Regra proposta |
|---|---|
| Gênero | Pelo menos uma tag entre as 5 principais (SteamSpy): `RPG`, `Action RPG`, `Action`, `Open World`, `Souls-like`, `Action-Adventure` |
| Porte AAA | Preço de lançamento ≥ US$ 39,99 **ou** publisher em lista curada de grandes publishers |
| Relevância | ≥ 2.000 reviews no total |
| Recorte temporal | Lançado a partir de 2016 |
| Modelo de negócio | Jogo **pago** (free-to-play excluído) |

- **Tamanho-alvo:** ~300 jogos analisados (coleta de ~350 como margem de segurança).
- **Região:** apenas **EUA / USD**. Preço regional fica fora do escopo.
- **Exemplos esperados:** Baldur's Gate 3, Elden Ring, Cyberpunk 2077, The Witcher 3, Monster Hunter Wilds, Red Dead Redemption 2, Black Myth: Wukong, Kingdom Come: Deliverance II, Clair Obscur: Expedition 33, Hogwarts Legacy.

**Por que RPG/Ação AAA:** CCU alto (sinal forte), preço alto (o desconto tem impacto financeiro relevante), publishers com estratégias bem distintas (descontos agressivos x descontos raros) e eventos claros de DLC e sequências.

**Atenção:** jogos AAA dão desconto com **menos frequência** que indies. Para garantir eventos de promoção suficientes, o projeto combina a coleta contínua com a **reconstrução do histórico** (preços via IsThereAnyDeal e reviews com data). Ver seção 5.

### 4.2 Fora do escopo
- Vendas e faturamento **reais** (não existem publicamente; o projeto trabalha com **estimativas**).
- Wishlists, bundles e vendas de DLC.
- Outras lojas (Epic, GOG) e consoles.
- Preços regionais, impostos e reembolsos.
- Análise de sentimento do texto das reviews (candidata a **fase bônus**).

## 5. Fontes de dados

| Dado | Fonte | Frequência | Histórico retroativo? |
|---|---|---|---|
| Lista de jogos mais jogados | Steam Web API (`ISteamChartsService`)\* | Semanal | ❌ |
| Preço e desconto | Store API — `appdetails?filters=price_overview` (aceita vários appids por chamada)\* | Diária | ❌ → complementado via **IsThereAnyDeal API** |
| Total de reviews (positivas/negativas) | Reviews API — `appreviews/{appid}?json=1` (só o `query_summary`) | Diária | ✅ parcial (histograma de reviews\* / datas das reviews) |
| Jogadores simultâneos (CCU) | Steam Web API — `ISteamUserStats/GetNumberOfCurrentPlayers` | De hora em hora | ❌ |
| Ranking de mais vendidos | Steam Web API (`IStoreTopSellersService`)\* | Semanal | ❌ |
| Metadados (publisher, dev, gênero, tags, lançamento) | Store API + SteamSpy | Semanal | — |
| Notícias / patch notes / DLC | Steam Web API — `ISteamNews/GetNewsForApp` | Diária | ✅ |

\* Endpoints não oficiais ou pouco documentados: **validar na Fase 1**.

**Restrições:**
- Store API: limite de ~200 requisições a cada 5 minutos → retry com backoff obrigatório.
- Steam Web API: exige API key gratuita (guardada em Databricks Secrets).
- **Não fazer scraping do SteamDB**, que é proibido pelos termos de uso.

## 6. Metodologia de estimativa

### 6.1 Proxies de vendas
| Proxy | Uso | Limitação |
|---|---|---|
| **Velocidade de reviews** (novas reviews/dia) | Principal proxy de unidades vendidas (método Boxleiter: 1 review ≈ N vendas) | O multiplicador varia por gênero/ano; há atraso entre a compra e a review |
| **CCU (pico diário)** | Engajamento e entrada de novos jogadores | Inclui jogadores antigos que voltaram |
| **Posição no ranking de mais vendidos** | Validação qualitativa | Só mostra a posição, sem volume |

O multiplicador de reviews será um **parâmetro** (`dbt var`) com **análise de sensibilidade** (ex.: 20 / 30 / 50).

### 6.2 Ponto de equilíbrio
Com desconto `d`, o uplift mínimo de unidades para empatar a receita é:

```
uplift_break_even = 1 / (1 - d)
```

| Desconto | Uplift necessário |
|---|---|
| 20% | 1,25x |
| 50% | 2x |
| 75% | 4x |
| 90% | 10x |

### 6.3 Receita estimada
```
receita_estimada = unidades_estimadas × preço_com_desconto × (1 - taxa_steam)
```
- `taxa_steam` = 30% (premissa conservadora; a Steam aplica faixas de 30% / 25% / 20% conforme o faturamento do jogo, o que fica documentado como limitação).
- **Contrafactual:** receita que o jogo teria no mesmo período a preço cheio, baseada no baseline pré-promoção.

### 6.4 Janelas de análise por promoção
| Janela | Definição |
|---|---|
| Baseline | 14 dias antes do início (sem desconto) |
| Promoção | Do início ao fim do desconto |
| Pós-promoção | 14 dias após o fim |

### 6.5 Controle de sazonalidade
Durante as promoções sazonais todos os jogos descontam e o tráfego da loja inteira sobe. Para isolar o efeito do desconto:
- **v1:** comparação baseline x promoção x pós, separada por tipo de promoção.
- **v2 (opcional):** grupo de controle com jogos do mesmo gênero **sem** desconto no mesmo período (diferenças-em-diferenças).

## 7. Hipóteses

| # | Hipótese |
|---|---|
| H1 | Descontos acima de 75% raramente atingem o ponto de equilíbrio no período da promoção |
| H2 | Promoções sazonais geram mais uplift absoluto, mas menos uplift por ponto de desconto que promoções próprias |
| H3 | Existe ressaca pós-promoção mensurável nos 14 dias seguintes |
| H4 | Jogos com descontos raros e rasos mantêm um uplift maior a cada promoção ("fadiga de desconto" nos demais) |
| H5 | Promoções do jogo base coincidindo com lançamento de DLC rendem mais uplift |
| H6 | O lançamento de uma sequência aumenta o efeito da promoção do jogo antecessor |

## 8. Arquitetura

```
┌──────────────┐   ┌──────────────────────── Databricks Free Edition ─────────────────────────┐
│ Steam APIs   │   │                                                                           │
│ IsThereAny   │──▶│ Python (Lakeflow Job) ──▶ Volume (JSON cru) ──▶ Auto Loader ──▶ Bronze    │
│ Deal         │   │                                                                  │        │
│ SteamSpy     │   │                                       dbt (task no Job)          ▼        │
└──────────────┘   │                                Silver ──▶ Gold ──▶ Notebook / Dashboard   │
                   │                         (Unity Catalog: steam.bronze / silver / gold)     │
                   └───────────────────────────────────────────────────────────────────────────┘
                          ▲                                       ▲
                     GitHub (código) ── GitHub Actions (CI: lint + dbt build)
```

### 8.1 Unity Catalog
| Objeto | Nome |
|---|---|
| Catalog | `steam` |
| Schemas | `steam.bronze`, `steam.silver`, `steam.gold` (+ `steam.ci` para CI) |
| Volume (área de pouso) | `steam.bronze.raw` |

### 8.2 Organização do Volume
```
/Volumes/steam/bronze/raw/
├── prices/dt=YYYY-MM-DD/...
├── reviews_summary/dt=YYYY-MM-DD/...
├── ccu/dt=YYYY-MM-DD/hour=HH/...
├── top_sellers/dt=YYYY-MM-DD/...
├── metadata/dt=YYYY-MM-DD/...
├── news/dt=YYYY-MM-DD/...
├── price_history_itad/dt=YYYY-MM-DD/...
└── _checkpoints/
```

### 8.3 Princípios
- **ELT:** a ingestão só busca e grava o **JSON cru completo**; toda transformação acontece no dbt.
- **Coletar amplo, modelar enxuto:** o payload completo fica guardado para permitir reprocessamento.
- **Idempotência:** reexecutar um dia não duplica dados.
- **Incrementalidade:** Auto Loader e modelos dbt incrementais, para caber nas cotas do Free Edition.

## 9. Modelagem dbt

| Camada | Modelo | Descrição |
|---|---|---|
| Source | `bronze.*` | Tabelas carregadas pelo Auto Loader, com `freshness` |
| Staging | `stg_steam__prices`, `stg_steam__reviews_summary`, `stg_steam__ccu`, `stg_steam__top_sellers`, `stg_steam__apps`, `stg_steam__news`, `stg_itad__price_history` | Limpeza, tipagem, renomeação |
| Snapshot | `snap_game_price` | SCD2: cada mudança de preço vira uma linha |
| Intermediate | `int_promotions` | Detecta eventos de promoção a partir da série de preços (gaps-and-islands) |
| Intermediate | `int_game_daily` | 1 linha por jogo/dia: preço, desconto, novas reviews, pico de CCU |
| Intermediate | `int_promotion_windows` | Baseline / promoção / pós por evento |
| Gold | `dim_game`, `dim_publisher`, `dim_date` | Dimensões |
| Gold | `fct_promotion_impact` | Uplift, receita estimada, contrafactual, break-even, ressaca, por promoção |
| Gold | `mart_publisher_pricing_strategy` | Frequência, profundidade e eficiência dos descontos por publisher |

### Qualidade de dados
- Testes genéricos: `unique`, `not_null`, `relationships`, `accepted_values`.
- `dbt_expectations`: faixas válidas (desconto entre 0 e 100, CCU ≥ 0) e séries diárias sem buracos.
- `source freshness`: alerta se a coleta parar.

## 10. Stack

| Camada | Tecnologia |
|---|---|
| Ingestão | Python 3.11+, httpx/requests, tenacity, pydantic (opcional) |
| Plataforma | Databricks Free Edition: Unity Catalog, Volumes, Delta Lake, Auto Loader, Lakeflow Jobs, SQL Warehouse serverless, Secrets |
| Transformação | dbt Core + `dbt-databricks`, `dbt_utils`, `dbt_expectations` |
| Análise / visualização | Notebook Databricks (pandas/statsmodels) e/ou AI/BI Dashboard — **decisão em aberto** |
| Engenharia | Git + GitHub, Databricks Asset Bundles, GitHub Actions, SQLFluff, ruff, uv, pre-commit |

**Deliberadamente fora:** Airflow/Dagster (o Lakeflow Jobs basta), Kafka/streaming (batch atende), Terraform (Asset Bundles cobrem o escopo), Docker (sem serviço próprio para containerizar).

## 11. Requisitos não funcionais
- Rodar inteiramente no **Databricks Free Edition** (serverless, respeitando as cotas).
- **Nenhum segredo no código**: API keys apenas em Databricks Secrets / variáveis de ambiente.
- Ingestão resiliente a falhas de API (retry, backoff, log de erros).
- Pipeline reprocessável a partir do bronze.

## 12. Fases e cronograma

As datas das promoções sazonais são aproximadas; confirmar no calendário oficial da Steam.

| Fase | Período | Entregas |
|---|---|---|
| **0. Setup** | Semana de 2026-11-09 | Repositório, conta Databricks, API keys, catalog/schemas/volume |
| **1. Ingestão** ⚠️ prioridade | Até ~2026-11-20 | Scripts de coleta + Jobs agendados + bronze. Validação dos endpoints. Definição final da coorte. Carga do histórico (ITAD + reviews) |
| **2. Fundação dbt** | Nov–Dez 2026 | Staging, snapshot, `int_promotions`, `int_game_daily`, testes, CI |
| **3. Análises** | Jan 2027 | Marts gold, verificação das hipóteses, escolha entre notebook e dashboard |
| **4. Vitrine** | Jan–Fev 2027 | README final, dashboard/notebook, `dbt docs`, prints e descobertas |

**Eventos-alvo:** Autumn Sale (fim de nov/2026) e Winter Sale (fim de dez/2026). A coleta precisa estar no ar **antes** da Autumn Sale para capturar o baseline.

## 13. Entregáveis
1. Repositório no GitHub com README contando a história: pergunta, arquitetura, metodologia, limitações e resultados.
2. Projeto dbt documentado, com linhagem e testes.
3. Pipeline agendado e funcionando no Databricks.
4. Análise final (notebook e/ou dashboard) com 3 a 5 descobertas claras.
5. CI rodando lint + `dbt build` a cada PR.

## 14. Riscos e mitigações

| Risco | Impacto | Mitigação |
|---|---|---|
| Poucos eventos de promoção em jogos AAA | Conclusões fracas | Reconstruir histórico via ITAD + reviews; manter a coorte em ~300 jogos |
| Endpoints não oficiais mudarem ou caírem | Coleta interrompida | Validar na Fase 1, monitorar com `source freshness`, guardar o JSON cru |
| Multiplicador de reviews impreciso | Estimativas de vendas erradas | Parâmetro com análise de sensibilidade; foco em **variação relativa**, não em valores absolutos |
| Sazonalidade confunde o efeito do desconto | Uplift superestimado | Separar por tipo de promoção; grupo de controle na v2 |
| Updates/DLC coincidindo com a promoção | Efeito misturado | Coletar notícias e marcar esses eventos em `int_promotion_windows` |
| Cotas do Free Edition | Jobs falhando | Coleta leve, processamento incremental |
| Atraso no início da coleta | Perder o baseline da Autumn Sale | Fase 1 tem prioridade máxima |

## 15. Decisões em aberto

| Decisão | Quando decidir |
|---|---|
| Formato da análise final: notebook, dashboard ou ambos | Fase 3, com algumas semanas de dados |
| Rigor estatístico: antes/depois ou diferenças-em-diferenças | Fase 3 |
| Valor-base do multiplicador de reviews | Fase 3 |
| Lista curada de publishers AAA e critérios finais da coorte | Fase 1 |
| Foco da vitrine: Engenharia de Dados, Analytics Engineering ou Análise | Antes da Fase 4 |

## 16. Glossário

| Termo | Significado |
|---|---|
| **CCU** | *Concurrent users*: jogadores simultâneos |
| **Uplift** | Quanto uma métrica cresceu em relação ao baseline |
| **Baseline** | Comportamento normal do jogo, sem promoção |
| **Contrafactual** | O que teria acontecido sem a promoção |
| **Break-even** | Uplift mínimo para a receita empatar apesar do desconto |
| **Ressaca (pull-forward)** | Queda nas vendas após a promoção, por compras antecipadas |
| **SCD2** | *Slowly Changing Dimension* tipo 2: histórico versionado de um atributo |
| **Medallion** | Arquitetura em camadas bronze (cru) → silver (tratado) → gold (negócio) |
| **Método Boxleiter** | Estimativa de vendas a partir do número de reviews |
