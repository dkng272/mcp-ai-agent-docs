# Database Schema Documentation

**Last Updated**: 2026-02-03
**Note**: Use MCP tools (`mssql_describe_table`, `mongo_describe_collection`) for full schemas on demand.

---

## Overview

### Azure SQL - dclab
- **Tools**: `mssql_read_data`, `mssql_describe_table`, `mssql_export_data`
- **Tables**: 43 (financials, sector operations, commodities, banking, bonds, portfolios)

### MongoDB - IRIS
- **Tools**: `mongo_find`, `mongo_aggregate`, `mongo_describe_collection`
- **Collections**: 23 (analyst models, real estate, interbank, macro research, ticker news, CBRE data, O&G projects, container data)

### MongoDB - CEODatabase
- **Tools**: Same as IRIS with `database: "CEODatabase"` parameter
- **Collections**: 4 (VEIL fund data, AUM snapshots)

### Supabase (PostgreSQL) - Broker Consensus
- **Tools**: `supabase_consensus_*` (4 tools)
- **Data**: Broker research reports, quarterly consensus, sentiment analysis

### Supabase (PostgreSQL) - Social Sentiment
- **Tools**: `f319_*` (5 tools), `zalo_*` (5 tools), `sentiment_dashboard_get`
- **Data**: F319 forum analysis, Zalo group signals, KOL tracking

### Utility Tools
- `find_relevant_tables` - Semantic search across all databases
- `execute_python` - Server-side Python with DB access
- `mongo_vector_search` - Hybrid search over macro research

---

## SQL Tables

### Financial Data

#### `FA_Quarterly` / `FA_Annual`
Financial statement metrics. Key: `TICKER`, `KEYCODE`, `DATE`, `VALUE`, `YoY`.

**KEYCODEs**: `Net_Revenue`, `EBIT`, `EBITDA`, `NPAT`, `NPATMI`, `EBIT_Margin`, `Total_Asset`, `Cash`, `Debt`, `Equity`, `Operating_CF`, `FCF`

```sql
SELECT TICKER, DATE, VALUE, YoY FROM FA_Quarterly WHERE KEYCODE = 'NPATMI' AND YEAR = 2024
```

#### `Forecast`
DC internal + broker consensus forecasts. Key: `TICKER`, `KEYCODE`, `DATE`, `VALUE`, `RATING`, `FORECASTDATE`.

**KEYCODE formats**:
- DC internal: `NPATMI`, `Target_Price` (no prefix)
- Broker: `HSC.NPATMI`, `MBS.Target_Price` (broker prefix)

**Brokers**: ACBS, BSC, BVS, FPTS, HSC, MBKE, MBS, PHS, SSI, VCBS, VCSC, VDSC, VND, VPBS

```sql
-- DC internal
SELECT * FROM Forecast WHERE KEYCODE = 'NPATMI' AND DATE = '2025'
-- Broker consensus
SELECT * FROM Forecast WHERE TICKER = 'HPG' AND KEYCODE LIKE '%.NPATMI'
```

#### `Forecast_history`
Historical NPATMI forecast tracking. Key: `Ticker`, `Year`, `Quarter`, `Value`, `CreatedDate`.

⚠️ `Quarter = 5` means **annual** forecast (not Q5).

#### `Market_Data`
Unified market data: valuation, OHLCV, market cap, foreign flows. Key: `TICKER`, `TRADE_DATE`.

**Valuation**: `PE`, `PB`, `PS`, `EV_EBITDA`
**Price**: `PX_OPEN`, `PX_HIGH`, `PX_LOW`, `PX_LAST`, `VOLUME`, `VALUE`
**Market Cap**: `MKT_CAP`, `SHARES`
**Foreign**: `FOREIGNBUYVALUETOTAL`, `FOREIGNSELLVALUETOTAL`, `FOREIGNTOTALROOM`, `FOREIGNCURRENTROOM`

⚠️ Use this table for market cap, NOT `Sector_Map.MC($USm)`.

#### `MarketIndex`
Index data (VNINDEX, VN30, etc). Key: `COMGROUPCODE`, `TRADINGDATE`, `INDEXVALUE`, `PERCENTINDEXCHANGE`.

**Indices**: VNINDEX, VN30, VN100, VNALL, VNFIN, VNREAL, VNIND, VNMAT + sector indices

---

### Banking

#### `BankingMetrics`
Bank financials (quarterly). Key: `TICKER`, `YEARREPORT`, `LENGTHREPORT`, `ACTUAL`.

**Metrics**: `ROE`, `ROA`, `NPL`, `NIM`, `CIR`, `LDR`, `CASA`, `NPATMI`, `TOI`, `Loan`, `Deposit`

⚠️ **LENGTHREPORT**: 1=Q1, 2=Q2, 3=Q3, 4=Q4. Always filter `ACTUAL = 1`.

```sql
SELECT TICKER, ROE, NPL, NIM FROM BankingMetrics WHERE YEARREPORT = 2024 AND LENGTHREPORT = 3 AND ACTUAL = 1
```

#### `Bank_Deposit_Rates`
Deposit rates by bank/tenure. Key: `Bank`, `Date`, `rate_1` to `rate_13`.

#### `Bank_Writeoff`
Loan write-offs. Key: `TICKER`, `DATE`, `Writeoff`.

#### `Vietnam_Credit_Deposit`
System-wide credit/deposit/M2. Key: `date`, `credit_vnd_bn`, `credit_growth_yoy_pct`, `deposit_vnd_bn`, `pure_ldr`.

---

### Commodities

#### `Commodity`
Commodity prices. Key: `Ticker`, `Sector`, `Date`, `Price`.

⚠️ **CRITICAL**: `Ticker` here is commodity ID, NOT stock symbol. Query `Ticker_Reference` first!

**Sectors**: Agricultural, Chemicals, Energy, Fertilizer, Livestock, Metals, Power, Shipping_Freight, Steel, Textile

#### `Ticker_Reference`
Commodity name → ticker mapping. Key: `Ticker`, `Name`, `Sector`.

```sql
-- Step 1: Find commodity ticker
SELECT Ticker, Sector FROM Ticker_Reference WHERE Name LIKE '%Iron%'
-- Step 2: Get prices
SELECT Date, Price FROM Commodity WHERE Sector = 'Metals' AND Ticker = 'ORE62'
```

#### `iris_manual_data`
High-frequency operational data. Key: `Category`, `Ticker`, `KeyCode`, `Date`, `Value`.

⚠️ `Category IS NOT NULL` = commodity/macro data. `Category IS NULL` = stock alternative data.

---

### Sector Operations

#### `Steel_data`
Steel production/consumption/inventory. Key: `[Company Name]`, `Date`, `Classification`, `Channel`, `ProductCat`, `Value`.

**Companies**: Hoa Sen (HSG), Hoa Phat (HPG), Nam Kim (NKG), Ton Dong A (GDA)
**ProductCat**: Galvanized, Rebar, Pipe, Hot Roll Steel Coil, Cool Roll Steel Coil

#### `Aviation_Operations`
Airline metrics. Key: `Date`, `Airline`, `Metric_type`, `Traffic_type`, `Metric_value`.

**Airlines**: HVN, VJC, ACV, Bamboo, Pacific, VASCO
**Metrics**: Passengers, Market_share, Load_factor, Flight_count, OTP

#### `Aviation_Airfare` / `Aviation_Revenue` / `Aviation_Market`
Ticket prices, revenue breakdown, and market metrics (fuel, passenger mix).

#### `Power_Company_Operations`
Power generation by plant. Key: `Company`, `Plant`, `Date`, `Metric`, `Value`.

**Companies**: PC1, REE, HDG, GEG, POW
**Metrics**: volume, revenue, asp, contracted, mobilized

#### `Power_Metrics`
Power generation mix. Key: `Category`, `Entity`, `Date`, `Value`.

**Categories**: generation (COAL, HYDRO, GAS, RENEWABLES), market, economic

#### `Power_Reservoir_Metrics`
Hydropower reservoir levels. Key: `Region`, `Reservoir`, `Datetime`, `Value`.

#### `Container_volume`
Port throughput. Key: `Date`, `Region`, `Company`, `Port`, `[Total throughput]`.

**Companies**: PHP, VSC, SNP, GMD, SGP, CDN

---

### Brokerage

#### `BrokerageMetrics`
Brokerage financials. Key: `TICKER`, `YEARREPORT`, `LENGTHREPORT`, `ACTUAL`, `KEYCODE`, `VALUE`.

**Tickers**: SSI, VCI, HCM, VND, MBS, FPTS, VIX, SHS
**KEYCODEs**: `NPATMI`, `Net_Revenue`, `ROE`, `Margin_Lending_book`, `Net_Brokerage_Income`

#### `Brokerage_Propbook`
Proprietary holdings. Key: `Ticker`, `Quarter`, `Holdings`, `Value`.

#### `Brokerage_Market_Share`
Quarterly broker market share rankings. Key: `TICKER`, `YEAR`, `QUARTER`, `MARKET_SHARE_PCT`, `RANK`.

```sql
SELECT * FROM Brokerage_Market_Share WHERE YEAR = 2025 AND QUARTER = 4 ORDER BY RANK
```

#### `Brokerage_Comments`
AI-generated quarterly commentary for brokerage stocks. Key: `TICKER`, `QUARTER`, `COMMENTARY`, `GENERATED_AT`.

Detailed markdown analysis covering TOI breakdown, margin lending, investment/trading, IB, and cost control.

#### `Broker_Calls`
Broker investment theses with growth outlook and catalysts. Key: `Period`, `Broker`, `Ticker`, `Growth`, `Catalyst`.

```sql
SELECT * FROM Broker_Calls WHERE Ticker = 'VHM' ORDER BY Period DESC
```

---

### Other Tables

#### `PNJ_Gold_Prices`
Daily PNJ gold buy/sell prices (VND per tael). Key: `Date`, `Type`, `Buy`, `Sell`.

```sql
SELECT * FROM PNJ_Gold_Prices ORDER BY Date DESC
```

#### `Sector_Map`
Stock sector mapping. Key: `Ticker`, `Sector`, `L1`, `L2`, `L3`, `McapClassification`, `VNI`.

#### `Bonds_Issuance`
Corporate bonds. Key: `issuer`, `bond_code`, `industry_sector`, `issuance_value_billion_vnd`, `interest_rate`, `issue_date`.

#### `IRIS_Company_Comments`
Analyst commentary. Key: `TICKER`, `TITLE`, `DESCRIPTION`, `CATEGORY`, `IMPACT`, `CREATEDATE`.

#### `Model_Portfolio`
Portfolio holdings. Key: `Port`, `Date`, `AssetId`, `NAVWeight`.

#### `ceic_macro_data`
CEIC economic indicators. Key: `id`, `name`, `value`, `date`, `unit`.

---

## MongoDB Collections

### `CompanyModels`
Analyst financial models with forecasts. Query by `ticker`.

```javascript
mongo_find("CompanyModels", { ticker: "MWG" })
```

### `RealEstateProjects`
Project DCF valuations. Key fields: `ticker`, `project_name`, `npv_to_company`, `irr`, `yearly_data`.

```javascript
mongo_find("RealEstateProjects", { ticker: "VHM" }, {}, { npv_to_company: -1 })
```

### `InterbankUpdates`
Daily FX/interbank updates. Key: `date`, `exchange_rate`, `open_market`.

### `LendingRates`
Bank mortgage rates. Key: `bank`, `ticker`, `interest_rates`, `loan_terms`.

### `MoCData`
MoC quarterly real estate stats. Key: `quarter`, `year`, `data.transactions`, `data.credit`.

### `AgentSkillSets`
AI knowledge base. Query by `skill_set_id` (e.g., "sector:real_estate", "ticker:VHM").

### `macro_research_articles`
Macro research articles. Key: `title`, `source`, `author`, `date_iso`, `summary`, `content`.

**Sources**: gavekal, goldman_sachs

```javascript
mongo_find("macro_research_articles", { source: "gavekal" }, {}, { date_iso: -1 }, 5)
```

### `macro_research_embeddings`
Vector embeddings for semantic search. Use `mongo_vector_search` tool.

```python
mongo_vector_search(query="Fed rate policy", source="gavekal", days=7)
```

### `CBREMarketData`
CBRE quarterly real estate data (HCMC, Hanoi). Key: `city`, `period`, `year`, `quarter`.

**Segments**: `condo_price`, `landed_price`, `office`, `retail`, `industrial`, `serviced_apartment`

```javascript
mongo_find("CBREMarketData", { city: "Ho Chi Minh City" }, {}, { year: -1, quarter: -1 }, 1)
```

### `O&GProjectsData`
Vietnam O&G/LNG projects. Key: `data.Project`, `data.Investors`, `data.Capex (USDbn)`, `metadata.data_type`.

```javascript
mongo_find("O&GProjectsData", { "metadata.data_type": "O&G_LNG_Projects" })
```

### `commodity_news`
AI commodity news summaries. Key: `commodity_group`, `summary`, `direction`, `timeline`.

```javascript
mongo_find("commodity_news", { direction: "bullish" }, {}, { search_date: -1 })
```

### `VPAContainerData`
VPA monthly container throughput by region/company/port. Key: `period`, `year`, `month`, `north`, `central`, `south`, `national_total`.

```javascript
mongo_find("VPAContainerData", {}, {}, { year: -1, month: -1 }, 1)
```

### `TickerNews`
Raw Vietnamese stock news articles. Key: `ticker`, `title`, `content`, `published_date`, `source`.

### `ProcessedTickerNews`
AI-processed WhatsApp news with ticker extractions. Key: `message_id`, `source_chat`, `timestamp`, `extractions[]`.

**Fields**: `extractions[].ticker`, `extractions[].sentiment`, `extractions[].news_type`, `extractions[].summary`

```javascript
mongo_find("ProcessedTickerNews", { "extractions.ticker": "DGW" }, {}, { timestamp: -1 }, 5)
```

### `supplement_data`
Company operational KPIs from Excel models. Key: `ticker`, `data_type`, `metrics`.

### `WhatsappCrawler`
Crawled WhatsApp group messages. Key: `group_name`, `message`, `timestamp`.

---

## MongoDB - CEODatabase

### `veil_monthly_snapshots`
Monthly fund NAV snapshots. Key: `fund_code`, `date`, `nav`, `aum`.

```javascript
mongo_find("veil_monthly_snapshots", { fund_code: "VEIL" }, {}, { date: -1 }, 12, "CEODatabase")
```

### `veil_shareholders`
Fund shareholder registry. Key: `fund_code`, `shareholder_name`, `shares`, `percentage`.

### `aum_snapshots`
AUM tracking over time. Key: `date`, `total_aum`, `fund_breakdown`.

### `veil_summary_stats`
Fund summary statistics. Key: `fund_code`, `ytd_return`, `total_return`, `sharpe_ratio`.

---

## Supabase - Broker Consensus

Access via dedicated tools (not direct SQL). Data stored in Supabase PostgreSQL.

### Tools

#### `supabase_consensus_analyze`
Get quarterly consensus + recent broker comparisons for a ticker.
```
ticker: "MWG", quarter: "Q4 2024", comparisons: 4
```

#### `supabase_consensus_new_reports`
Scan recent broker reports (last N days) for sentiment shifts.
```
days: 2
```

#### `supabase_consensus_comparison_get`
Full details of a specific broker comparison report.
```
ticker: "MWG", broker: "HSC", report_date: "2025-01-15"
```

#### `supabase_consensus_pdf_get`
Get/download broker research PDF reports.
```
ticker: "MWG", broker: "HSC", download_path: "~/Downloads/report.pdf"
```

---

## Supabase - F319 Forum Data

AI-analyzed Vietnamese stock forum data from f319.com. Access via dedicated tools.

### Tools

#### `f319_discussion_points_search`
Search 87K+ AI-extracted discussion points with sentiment analysis.
- Filter: `ticker`, `sentiment` (bullish/bearish/neutral), `signal_score` (1-5), `informational_value` (High/Medium/Low)
```
ticker: "VNM", sentiment: "bullish", signal_score: 1, limit: 20
```

#### `f319_kol_posts_search`
Search 4.8K+ analyzed posts from Key Opinion Leaders.
- Filter: `kol_username`, `ticker`, `sentiment`, `signal_score`, `days`
```
kol_username: "livermore888", ticker: "FPT", days: 7
```

#### `f319_stock_thesis_get`
Synthesized investment thesis (bull/bear cases) for 509 stocks.
```
ticker: "VNM"
```

#### `f319_thread_search`
Search 49K+ forum threads with ticker mappings.
- Filter: `ticker`, `keyword`, `active_only`, `min_posts`, `sort_by` (recent/popular/posts)
```
ticker: "VNM", sort_by: "popular", limit: 20
```

#### `f319_kol_list`
List 21 KOL profiles with quality/accuracy scores.
- Filter: `sector`, `ticker`, `min_quality_score`, `verified_only`
```
sector: "Banking", min_quality_score: 70, sort_by: "accuracy"
```

---

## Supabase - Zalo Investment Groups

Real-time signals from 24 Vietnamese investment Zalo groups. Access via dedicated tools.

### Tools

#### `zalo_daily_recommendations_get`
4,100+ daily stock picks with BUY/SELL/HOLD signals.
- Filter: `ticker`, `analysis_date`, `recommendation_type`, `min_confidence`, `risk_level`
```
recommendation_type: "BUY", min_confidence: 0.7, sort_by: "confidence"
```

#### `zalo_realtime_alerts_search`
14,500+ real-time alerts (breakouts, volume spikes, sentiment shifts).
- Filter: `ticker`, `alert_type`, `severity`, `action_recommended`, `hours`
- Alert types: PRICE_BREAKOUT, VOLUME_SPIKE, SENTIMENT_SHIFT, TECHNICAL_SIGNAL, NEWS_IMPACT, ADMIN_SIGNAL
```
alert_type: "PRICE_BREAKOUT", severity: "HIGH", hours: 24
```

#### `zalo_market_sentiment_current`
Live sentiment snapshots from 12 groups.
- Filter: `group_name`, `sentiment` (BULLISH/BEARISH/NEUTRAL/MIXED), `admin_involved_only`
```
sentiment: "BULLISH", min_significance: 0.7
```

#### `zalo_ticker_heatmap_get`
Aggregated recommendations by ticker with BUY/SELL ratios.
- Filter: `ticker`, `analysis_date`, `net_sentiment`, `sort_by` (mentions/sentiment/buy_recs)
```
net_sentiment: "positive", sort_by: "mentions"
```

#### `zalo_market_shifts_search`
Detected sentiment changes and reversals.
- Filter: `ticker`, `significance_level`, `sentiment_after`, `hours`
```
significance_level: "HIGH", sentiment_after: "BULLISH", hours: 24
```

---

## Sentiment Dashboard

#### `sentiment_dashboard_get`
Comprehensive sentiment aggregation from all sources (Broker Consensus + Zalo + F319).
```
ticker: "VNM", timeframe_hours: 24, include_charts: true
```

Returns: Broker consensus, Zalo sentiment breakdown, F319 discussion analysis, aggregate score.

---

## Quick Reference

| Question | Tool/Table | Key Filter |
|----------|------------|------------|
| NPATMI for MWG | `FA_Quarterly` | `KEYCODE = 'NPATMI'` |
| DC forecast | `Forecast` | `KEYCODE = 'NPATMI'` (no prefix) |
| Broker consensus (SQL) | `Forecast` | `KEYCODE LIKE '%.NPATMI'` |
| Stock price/volume | `Market_Data` | `TICKER`, `TRADE_DATE` |
| Market cap | `Market_Data` | `MKT_CAP` column |
| P/E, P/B valuation | `Market_Data` | `PE`, `PB`, `PS`, `EV_EBITDA` |
| Foreign flows (stock) | `Market_Data` | `FOREIGNBUYVALUETOTAL` |
| Foreign flows (index) | `MarketIndex` | `FOREIGNBUYVALUETOTAL` |
| VNINDEX performance | `MarketIndex` | `COMGROUPCODE = 'VNINDEX'` |
| Bank ROE/NPL | `BankingMetrics` | `ACTUAL = 1` |
| Bank deposit rates | `Bank_Deposit_Rates` | `Bank`, `Date` |
| System credit growth | `Vietnam_Credit_Deposit` | `credit_growth_yoy_pct` |
| Commodity prices | `Ticker_Reference` → `Commodity` | Query reference first! |
| Steel production | `Steel_data` | `Classification`, `ProductCat` |
| Airline passengers | `Aviation_Operations` | `Metric_type = 'Passengers'` |
| Power generation | `Power_Metrics` | `Category = 'generation'` |
| Port throughput | `Container_volume` or `VPAContainerData` | `Company`, `Port` |
| Brokerage metrics | `BrokerageMetrics` | `KEYCODE`, `ACTUAL = 1` |
| Broker market share | `Brokerage_Market_Share` | `YEAR`, `QUARTER`, `RANK` |
| Broker investment thesis | `Broker_Calls` | `Ticker`, `Period` |
| Brokerage AI commentary | `Brokerage_Comments` | `TICKER`, `QUARTER` |
| Gold prices (PNJ) | `PNJ_Gold_Prices` | `Date`, `Type` |
| Corporate bonds | `Bonds_Issuance` | `industry_sector`, `issue_date` |
| Analyst comments | `IRIS_Company_Comments` | `TICKER`, `ISDELETED = 0` |
| Sector mapping | `Sector_Map` | `Ticker`, `Sector` |
| RE project NPV | `RealEstateProjects` | `ticker`, `npv_to_company` |
| CBRE market data | `CBREMarketData` | `city`, `year`, `quarter` |
| O&G projects | `O&GProjectsData` | `metadata.data_type` |
| Commodity news | `commodity_news` | `commodity_group`, `direction` |
| Processed ticker news | `ProcessedTickerNews` | `extractions.ticker`, `timestamp` |
| Macro research | `macro_research_articles` | `source`, `date_iso` |
| Semantic macro search | `mongo_vector_search` | `query`, `source`, `days` |
| **Broker consensus analysis** | `supabase_consensus_analyze` | `ticker`, `quarter` |
| **New broker reports** | `supabase_consensus_new_reports` | `days` |
| **Download broker PDF** | `supabase_consensus_pdf_get` | `ticker`, `broker` |
| **Forum sentiment** | `f319_discussion_points_search` | `ticker`, `sentiment` |
| **KOL opinions** | `f319_kol_posts_search` | `kol_username`, `ticker` |
| **Stock thesis** | `f319_stock_thesis_get` | `ticker` |
| **Zalo buy/sell signals** | `zalo_daily_recommendations_get` | `ticker`, `recommendation_type` |
| **Real-time alerts** | `zalo_realtime_alerts_search` | `alert_type`, `severity` |
| **Market sentiment** | `zalo_market_sentiment_current` | `sentiment` |
| **All-source sentiment** | `sentiment_dashboard_get` | `ticker`, `timeframe_hours` |
| **Find tables** | `find_relevant_tables` | `question`, `database` |
| **Run Python** | `execute_python` | `code` |
| **VEIL fund data** | `veil_monthly_snapshots` | `database: CEODatabase` |
