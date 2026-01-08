# Database Schema Documentation

**Last Updated**: 2026-01-08
**Note**: Use MCP tools (`describe_table`, `collection-schema`) for full schemas on demand.

---

## Overview

### Azure SQL - dclab
- **Tools**: `mcp__MicrosoftSQL__read_data`, `mcp__MicrosoftSQL__describe_table`
- **Tables**: 43 (financials, sector operations, commodities, banking, bonds, portfolios)

### MongoDB - IRIS
- **Tools**: `mcp__DC_MongoDB_MCP__find`, `mcp__DC_MongoDB_MCP__aggregate`, `mcp__DC_MongoDB_MCP__collection-schema`
- **Collections**: 12 (analyst models, real estate, interbank, macro research, CBRE data, O&G projects, container data)

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

---

### Other Tables

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

---

## Quick Reference

| Question | Table/Collection | Key Filter |
|----------|------------------|------------|
| NPATMI for MWG | `FA_Quarterly` | `KEYCODE = 'NPATMI'` |
| DC forecast | `Forecast` | `KEYCODE = 'NPATMI'` (no prefix) |
| Broker consensus | `Forecast` | `KEYCODE LIKE '%.NPATMI'` |
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
| Corporate bonds | `Bonds_Issuance` | `industry_sector`, `issue_date` |
| Analyst comments | `IRIS_Company_Comments` | `TICKER`, `ISDELETED = 0` |
| Sector mapping | `Sector_Map` | `Ticker`, `Sector` |
| RE project NPV | `RealEstateProjects` | `ticker`, `npv_to_company` |
| CBRE market data | `CBREMarketData` | `city`, `year`, `quarter` |
| O&G projects | `O&GProjectsData` | `metadata.data_type` |
| Commodity news | `commodity_news` | `commodity_group`, `direction` |
| Macro research | `macro_research_articles` | `source`, `date_iso` |
| Semantic search | `mongo_vector_search` | `query`, `source`, `days` |
