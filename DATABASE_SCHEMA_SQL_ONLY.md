# Database Schema Documentation - SQL Server Only

**Last Updated**: 2025-11-12 (Added iris_manual_data)
**AI Agent Guide**: This document provides table schemas and simple query examples. For complex query patterns, see **MCP_SQL_QUERY_BEST_PRACTICES.md**.

---

> **📘 For Complex Queries**: This document covers table schemas and simple examples. For advanced query patterns (multi-table JOINs, window functions, nested derived tables, complex aggregations), see **[MCP_SQL_QUERY_BEST_PRACTICES.md](./MCP_SQL_QUERY_BEST_PRACTICES.md)**.

---

## Table of Contents
1. [Overview](#overview)
2. [Azure SQL - dclab](#azure-sql---dclab)
3. [Common Query Patterns](#common-query-patterns)

---

## Overview

### Access Methods
- **Azure SQL**: MCP Server
  - Database: `dclab`
  - Server: `sqls-dclab.database.windows.net`
  - Tables: 39 (financial data, commodities, market intelligence)

### Key Concepts
- **Tickers**: Vietnamese stock symbols (e.g., HPG, MWG, VNM)
- **Commodities**: Raw materials tracked for price movements (Iron Ore, HRC, Steel, etc.)
- **KEYCODE**: Metric identifiers in financial tables (NPATMI, Net_Revenue, EBIT, etc.)

### MCP Server Requirements

**⚠️ CRITICAL - Microsoft SQL MCP Server**:
- The MCP server uses the **official Microsoft SQL MCP** implementation
- All queries **MUST be valid JSON format** (not raw SQL strings)
- The server expects queries via the `mcp__MicrosoftSQL__read_data` tool with a `query` parameter

**JSON Query Format**:
- Queries are passed as JSON strings to the MCP tool
- Special characters must be properly escaped
- Use single quotes inside SQL strings, not double quotes

### ✅ All Numeric Fields Now Float!

**All tables converted** - No TRY_CAST needed:
- `FA_Quarterly`: VALUE, YoY → float
- `FA_Annual`: VALUE, YoY → float
- `BankingMetrics`: All 47 numeric columns → float
- `Market_Data`: All columns → float

**Direct calculations everywhere**:
```sql
-- Direct calculation on all tables
SELECT TICKER, VALUE * 1.1 AS Projected, YoY * 100 AS YoY_Pct
FROM FA_Quarterly
WHERE KEYCODE = 'Net_Revenue'

-- Always use NULLIF for division to avoid divide-by-zero
SELECT VALUE / NULLIF(Previous_Value, 0) AS Growth_Rate
FROM FA_Quarterly
```

---

## Azure SQL - dclab

### Financial Data Tables

#### `FA_Quarterly` & `FA_Annual`
**Purpose**: Financial statement metrics (quarterly and annual)

**FA_Quarterly Schema**:
```sql
KEYCODE      nvarchar   -- Metric identifier
TICKER       nvarchar   -- Stock symbol
DATE         nvarchar   -- Period (e.g., "2024Q3")
VALUE        float      -- Metric value
YEAR         int        -- Calendar year
YoY          float      -- Year-over-year change
```

**FA_Annual Schema**:
```sql
KEYCODE      nvarchar   -- Metric identifier
TICKER       nvarchar   -- Stock symbol
DATE         nvarchar   -- Period (e.g., "2024")
VALUE        float      -- Metric value
YEAR         bigint     -- Calendar year
YoY          float      -- Year-over-year change
```

**Key KEYCODES**:
- **Revenue**: `Net_Revenue`
- **Profitability**: `EBIT`, `EBITDA`, `NPAT`, `NPATMI`
- **Margins**: `EBIT_Margin`, `EBITDA_Margin`, `NPAT_Margin`, `NPATMI_Margin`
- **Balance Sheet**: `Total_Asset`, `Cash`, `Inventory`, `Debt`, `Equity`
- **Cash Flow**: `Operating_CF`, `Capex`, `FCF`

**Example Queries**:
```sql
-- FA_Quarterly - direct calculation
SELECT DATE, VALUE AS NPATMI, VALUE * 1.1 AS Projected, YoY * 100 AS YoY_Pct
FROM FA_Quarterly
WHERE TICKER = 'MWG' AND KEYCODE = 'NPATMI' AND DATE = '2024Q3'

-- FA_Annual - direct calculation
SELECT DATE, VALUE AS Revenue, YoY * 100 AS YoY_Growth_Pct
FROM FA_Annual
WHERE TICKER = 'HPG' AND KEYCODE = 'Net_Revenue'
ORDER BY YEAR DESC
```

---

#### `Forecast`
**Purpose**: Financial forecasts from the user's firm only (internal forecasts)

**⚠️ IMPORTANT**: This table contains ONLY forecasts made by the user's firm. Forecasts from other brokers/firms are stored in the `Forecast_Consensus` table.

**Schema**:
```sql
TICKER        nvarchar      -- Stock symbol
KEYCODE       nvarchar      -- Metric identifier (standard metrics only)
KEYCODENAME   nvarchar      -- Metric name
ORGANCODE     nvarchar      -- Organization code (user's firm)
DATE          nvarchar      -- Forecast period (e.g., "2025", "2026")
VALUE         float         -- Forecast value
RATING        nvarchar      -- Analyst rating (BUY, SELL, etc.)
FORECASTDATE  nvarchar      -- When the forecast was made
_content_hash nvarchar      -- Data integrity hash
```

**KEYCODE Format**:
- **Standard financial metrics only**: Same as FA_Quarterly/FA_Annual
- Examples: `NPATMI`, `Net_Revenue`, `EBIT`, `EBITDA`, `Total_Asset`, `ROE`, `ROA`
- NO broker-specific format (e.g., no `MBKE.NPATMI` - those are in Forecast_Consensus)

**RATING Values**:
- `BUY` - Strong buy recommendation
- `OUTPERFORM` - Expected to outperform market
- `MARKET-PERFORM` - Expected to match market
- `UNDER-PERFORM` - Expected to underperform market
- `SELL` - Sell recommendation
- `AVOID` - Avoid investment
- `WATCH` - Watch list

**Example Queries**:
```sql
-- Get firm's NPATMI forecasts for VNM
SELECT TICKER, DATE, VALUE, RATING, FORECASTDATE
FROM Forecast
WHERE TICKER = 'VNM' AND KEYCODE = 'NPATMI'
ORDER BY DATE

-- Get all firm's forecasts with BUY rating
SELECT TICKER, KEYCODE, DATE, VALUE, FORECASTDATE
FROM Forecast
WHERE RATING = 'BUY'
ORDER BY FORECASTDATE DESC
```

---

#### `Forecast_Consensus`
**Purpose**: Forecasts from other brokers/firms (market consensus)

**⚠️ IMPORTANT**: This table contains forecasts from OTHER brokers and firms (not the user's firm). The user's own forecasts are in the `Forecast` table.

**Schema**:
```sql
TICKER        varchar       -- Stock symbol
KEYCODE       varchar       -- Metric identifier (broker-specific format)
KEYCODENAME   varchar       -- Metric name
ORGANCODE     varchar       -- Broker/organization code
DATE          varchar       -- Forecast period (e.g., "2025", "2026")
VALUE         float         -- Forecast value
RATING        varchar       -- Analyst rating
FORECASTDATE  varchar       -- When forecast was made
```

**KEYCODE Format**:
- **Broker-specific forecasts**: `{BrokerCode}.{Metric}`
- Examples: `MBKE.NPATMI`, `HSC.Net_Revenue`, `SSI.EBIT`, `VCBS.ROE`
- Common broker codes: ACBS, BSC, BVS, BVSC, FPTS, HSC, MBKE, MBS, PHS, SSI, VCBS, VCSC, VDSC, VND, VPBS

**Use Cases**:
- Compare market consensus to firm's internal forecasts
- Analyze broker coverage and sentiment
- Track forecast changes over time from specific brokers

**Example Queries**:
```sql
-- Get all broker NPATMI forecasts for VNM
SELECT TICKER, KEYCODE, DATE, VALUE, FORECASTDATE
FROM Forecast_Consensus
WHERE TICKER = 'VNM' AND KEYCODE LIKE '%.NPATMI'
ORDER BY FORECASTDATE DESC

-- Get HSC's latest forecasts
SELECT TICKER, KEYCODE, DATE, VALUE, RATING
FROM Forecast_Consensus
WHERE KEYCODE LIKE 'HSC.%'
  AND FORECASTDATE = (SELECT MAX(FORECASTDATE) FROM Forecast_Consensus WHERE KEYCODE LIKE 'HSC.%')
ORDER BY TICKER
```

---

#### `Market_Data`
**Purpose**: Centralized market data combining valuation multiples, OHLCV price data, and market capitalization

**⚠️ IMPORTANT**: This table **replaces** the old `Valuation` and `MarketCap` tables. All data types are **float** - no TRY_CAST needed!

**Schema**:
```sql
-- Identification
TICKER              varchar        -- Stock symbol
TRADE_DATE          date           -- Trading date

-- Valuation Multiples (all float - no TRY_CAST needed!)
PE                  float          -- Price-to-Earnings
PB                  float          -- Price-to-Book
PS                  float          -- Price-to-Sales
EV_EBITDA           float          -- Enterprise Value / EBITDA

-- OHLCV Price Data (all float)
PX_OPEN             float          -- Opening price
PX_HIGH             float          -- Highest price
PX_LOW              float          -- Lowest price
PX_LAST             float          -- Closing price
VOLUME              float          -- Trading volume
VALUE               float          -- Trading value

-- Market Cap (float - no TRY_CAST needed!)
MKT_CAP             float          -- Market capitalization in VND

-- Metadata
UPDATE_TIMESTAMP    datetime       -- Last update time
```

**Key Features**:
- ✅ All numeric fields are **float** (no TRY_CAST required)
- ✅ Clean column names (PE instead of [P/E], no special characters)
- ✅ Combines 3 data types: valuation, price/volume, market cap
- ✅ Date type is `date` (clean, simple)
- ⚠️ Do NOT use `MC($USm)` from `Sector_Map` - use `MKT_CAP` from this table

**Example Queries**:
```sql
-- Get latest valuation multiples for HPG
SELECT TICKER, TRADE_DATE, PE, PB, PS, EV_EBITDA, MKT_CAP, PX_LAST
FROM Market_Data
WHERE TICKER = 'HPG'
ORDER BY TRADE_DATE DESC
OFFSET 0 ROWS FETCH NEXT 1 ROWS ONLY

-- Compare PE ratios for steel sector
SELECT m.TICKER, s.Sector, m.PE, m.PB, m.MKT_CAP
FROM Market_Data m
JOIN Sector_Map s ON m.TICKER = s.Ticker
WHERE s.Sector = 'Steel'
  AND m.TRADE_DATE = (SELECT MAX(TRADE_DATE) FROM Market_Data)
ORDER BY m.PE ASC
```

---

#### `MarketIndex`
**Purpose**: Vietnamese stock market indices with daily OHLCV data and market breadth

**Schema**:
```sql
-- Index Identification
COMGROUPCODE              nvarchar(50)   -- Index code (e.g., VNINDEX, VN30, VN100)
TRADINGDATE               datetime2      -- Trading date

-- Price Data (OHLC)
INDEXVALUE                float          -- Current index value
OPENINDEX                 float          -- Opening index
CLOSEINDEX                float          -- Closing index
HIGHESTINDEX              float          -- Highest index
LOWESTINDEX               float          -- Lowest index
INDEXCHANGE               float          -- Point change
PERCENTINDEXCHANGE        float          -- Percentage change
REFERENCEINDEX            float          -- Reference/base index

-- Volume & Value
TOTALMATCHVOLUME          float          -- Total matched volume
TOTALMATCHVALUE           float          -- Total matched value
TOTALDEALVOLUME           float          -- Total deal volume
TOTALDEALVALUE            float          -- Total deal value
TOTALVOLUME               float          -- Combined total volume
TOTALVALUE                float          -- Combined total value

-- Market Breadth
TOTALSTOCKUPPRICE         bigint         -- Number of stocks up
TOTALSTOCKDOWNPRICE       bigint         -- Number of stocks down
TOTALSTOCKNOCHANGEPRICE   bigint         -- Number of stocks unchanged
TOTALUPVOLUME             float          -- Volume from advancing stocks
TOTALDOWNVOLUME           float          -- Volume from declining stocks
TOTALNOCHANGEVOLUME       float          -- Volume from unchanged stocks

-- Foreign Trading
FOREIGNBUYVALUEMATCHED    float          -- Foreign buy value (matched)
FOREIGNBUYVOLUMEMATCHED   float          -- Foreign buy volume (matched)
FOREIGNSELLVALUEMATCHED   float          -- Foreign sell value (matched)
FOREIGNSELLVOLUMEMATCHED  float          -- Foreign sell volume (matched)
FOREIGNBUYVALUETOTAL      float          -- Foreign buy value (total)
FOREIGNBUYVOLUMETOTAL     float          -- Foreign buy volume (total)
FOREIGNSELLVALUETOTAL     float          -- Foreign sell value (total)
FOREIGNSELLVOLUMETOTAL    float          -- Foreign sell volume (total)
```

**Available Indices** (37 total):
- **Main Indices**: VNINDEX, VN30, VN50GROWTH, VN100, VNALL, VNDIAMOND, VNMID, VNSML
- **Sector Indices**: VNCOND (Consumer Discretionary), VNCONS (Consumer Staples), VNENE (Energy), VNFIN (Financials), VNHEAL (Healthcare), VNIND (Industrials), VNIT (IT), VNMAT (Materials), VNREAL (Real Estate), VNUTI (Utilities)
- **Thematic**: VNFINLEAD (Financial Leaders), VNFINSELECT, FLEADTRI, VNDIVIDEND
- **Total Return Indices** (TRI): VN30TRI, VN100TRI, VNALLTRI, VNDIVIDENDTRI, VNDTRI, VNFSTRI, VNMIDCAPTRI, VNMITECHTRI, VNSMALLTRI
- **Other**: VNX50, VNXALL, VNSI, VNMITECH

**Example Queries**:
```sql
-- Get latest VNINDEX data
SELECT TRADINGDATE, INDEXVALUE, PERCENTINDEXCHANGE,
       TOTALSTOCKUPPRICE, TOTALSTOCKDOWNPRICE, TOTALVOLUME
FROM MarketIndex
WHERE COMGROUPCODE = 'VNINDEX'
ORDER BY TRADINGDATE DESC
OFFSET 0 ROWS FETCH NEXT 1 ROWS ONLY

-- Compare VN30 vs VNINDEX (last 30 days)
SELECT TRADINGDATE, COMGROUPCODE, INDEXVALUE, PERCENTINDEXCHANGE
FROM MarketIndex
WHERE COMGROUPCODE IN ('VNINDEX', 'VN30')
  AND TRADINGDATE >= DATEADD(day, -30, GETDATE())
ORDER BY TRADINGDATE DESC, COMGROUPCODE
```

---

### Banking Sector Tables

#### `BankingMetrics`
**Purpose**: Comprehensive banking financial metrics (quarterly)

**Schema**:
```sql
-- Identification
TICKER          nvarchar
YEARREPORT      int          -- Report year
LENGTHREPORT    int          -- Quarter (1-4)
ACTUAL          bit          -- Actual vs estimated
DATE            datetime2    -- Report date
BANK_TYPE       nvarchar
PERIOD_TYPE     nvarchar

-- All Numeric Metrics (float - 47 columns)
ROE             float        -- Return on Equity
ROA             float        -- Return on Assets
NPL             float        -- Non-Performing Loan ratio
NIM             float        -- Net Interest Margin
CIR             float        -- Cost-to-Income Ratio
LDR             float        -- Loan-to-Deposit Ratio
CASA            float        -- Current/Savings Account Ratio
NPATMI          float        -- Net Profit After Tax
TOI             float        -- Total Operating Income
PPOP            float        -- Pre-Provision Operating Profit
PBT             float        -- Profit Before Tax
Loan            float        -- Total loans
Deposit         float        -- Total deposits
Total Assets    float        -- Total assets
Total Equity    float        -- Shareholders' equity
Net Interest Income float
OPEX            float        -- Operating Expenses
Provision expense float
Fees Income     float
Customer loans  float
Deposit balance float
NPL Coverage ratio float
Leverage Multiple float
Loan yield      float
Deposit yield   float
-- ... ~22 more float fields
```

**⚠️ IMPORTANT - Quarter Format**:
- **YEARREPORT**: Calendar year (e.g., 2024, 2025)
- **LENGTHREPORT**: Quarter number (1 = Q1, 2 = Q2, 3 = Q3, 4 = Q4)
- **ACTUAL**: 1 = Actual data, 0 = Estimated/Forecast
- **DATE**: When the data was reported/updated

**Example Queries**:
```sql
-- All metrics - direct calculation
SELECT TICKER, YEARREPORT, LENGTHREPORT, ROE, NPL, NIM, CASA,
       OPEX, [Net Interest Income] AS NII, [Provision expense]
FROM BankingMetrics
WHERE TICKER = 'VCB' AND YEARREPORT = 2025 AND LENGTHREPORT = 3 AND ACTUAL = 1

-- Calculate quality score
SELECT TICKER, (ROE * 0.4) + ((100 - NPL) * 0.3) + (CASA * 0.3) AS Quality_Score
FROM BankingMetrics
WHERE ACTUAL = 1 AND YEARREPORT = 2024 AND LENGTHREPORT = 3
ORDER BY Quality_Score DESC
```

---

#### `Banking_Comments` / `Brokerage_Comments`
**Purpose**: Analyst commentary on banking/brokerage results

**Common columns**: `TICKER`, `YEARREPORT`, `LENGTHREPORT`, `DATE`, commentary fields

---

### Commodity Price Tables

All commodity tables follow similar structure:

#### `Steel`, `Metals`, `Energy`, `Chemicals`, `Fertilizer`, etc.
**Schema**:
```sql
Ticker    varchar     -- Commodity identifier (NOT the same as stock tickers!)
Date      date        -- Price date
Price     decimal     -- Commodity price
```

**IMPORTANT - Ticker Mapping**:
- Commodity tables use `Ticker` column for commodity identifiers (NOT stock symbols!)
- Use `Ticker_Reference` table to map commodity names to ticker identifiers
- **Example**: Query `Ticker_Reference` to find "Iron Ore" → Returns Ticker "ORE62" in Sector "Metals"

**Example Query**:
```sql
-- Get HRC steel prices for last 30 days
SELECT Date, Price FROM Steel
WHERE Ticker = 'HRC' AND Date >= DATEADD(day, -30, GETDATE())
ORDER BY Date DESC
```

**Available Commodity Tables**:
- `Steel` - HRC, scrap, rebar
- `Metals` - Aluminum, copper, zinc
- `Energy` - Crude oil, coal, gas
- `Chemicals` - Caustic soda, PVC
- `Fertilizer` - Urea, DAP, NPK
- `Agricultural` - Grains, livestock
- `Fishery` - Aquaculture products
- `Textile` - Cotton, polyester
- `Shipping_Freight` - Container rates
- `Aviation_*` - Airfare, operations, revenue
- `Container_volume` - Shipping volumes
- `Livestock` - Hog, cattle prices

#### `iris_manual_data`
**Purpose**: Manually collected high-frequency operational and market data

**Dual-purpose table** containing:
1. **Commodity & Macro Data**: 13 categories including hog prices, chemicals, fertilizer, rice, textile exports, Vietnam banking aggregates
2. **Alternative Data**: 46 stock tickers with operational metrics (store counts, export volumes, passenger traffic, banking ratios)

**Schema**:
```sql
Category      varchar   -- Category grouping (NULL for stock tickers)
Ticker        varchar   -- Series identifier
KeyCode       varchar   -- Metric type
Date          datetime  -- Observation date
Value         float     -- Numeric value
Frequency     varchar   -- Daily/Weekly/Monthly
MeasureUnit   varchar   -- Measurement unit
UpdatedDate   datetime  -- Last update timestamp
```

**Key Filtering**:
```sql
-- Commodity/Macro data only
WHERE Category IS NOT NULL

-- Stock ticker alternative data only
WHERE Category IS NULL
```

**Data Coverage**: 158K records, 95 tickers, 2000-2025

**📖 For detailed documentation**, see **[IRIS_MANUAL_DATA_GUIDE.md](./IRIS_MANUAL_DATA_GUIDE.md)**

---

### Reference & Mapping Tables

#### `Ticker_Reference`
**Purpose**: Commodity lookup table - maps commodity names to SQL price tables

**⚠️ CRITICAL - Start Here for Commodity Price Queries**:
This is the **first table to query** when looking up commodity prices!

**Schema**:
```sql
Ticker       varchar     -- Commodity identifier (e.g., "ORE62", "AACNPC01 SMMC Index")
Name         varchar     -- Commodity name (e.g., "Ore 62", "Alumina")
Sector       varchar     -- SQL table name (e.g., "Metals" → dbo.Metals table)
Data_Source  varchar     -- Source system
Active       bit         -- Active status
```

**Key Mapping Rules**:
1. **Sector → SQL Table Name**: `Sector` column tells you which price table to query
   - Sector = "Metals" → Query `dbo.Metals`
   - Sector = "Steel" → Query `dbo.Steel`
   - Sector = "Energy" → Query `dbo.Energy`
2. **Name**: Human-readable commodity name (e.g., "Ore 62", "Alumina")
3. **Ticker → Price Query Key**: Use in WHERE clause of commodity price tables

**Example Workflow**:
```sql
-- Step 1: Find which table contains Iron Ore
SELECT Ticker, Name, Sector FROM Ticker_Reference
WHERE Name LIKE '%Iron%'
-- Returns: Ticker = 'ORE62', Name = 'Ore 62', Sector = 'Metals'

-- Step 2: Query the commodity price table (dbo.Metals in this case)
SELECT Date, Price FROM Metals
WHERE Ticker = 'ORE62'
ORDER BY Date DESC
```

---

#### `Sector_Map`
**Purpose**: Advanced sector/index mapping with market cap classifications

**Schema**:
```sql
ReportFormat         nvarchar   -- Report format type
OrganCode            nvarchar   -- Organization code
Ticker               nvarchar   -- Stock symbol
ExportClassification nvarchar   -- Export classification
Sector               nvarchar   -- Sector
L1, L2, L3           nvarchar   -- Multi-level sector hierarchy
Top80                nvarchar   -- Top 80 stocks by market cap
McapClassification   nvarchar   -- Large/Mid/Small cap
VNI                  nvarchar   -- VN-Index constituent
MC($USm)             nvarchar   -- Market cap in USD millions (⚠️ For classification only - use Market_Data table for actual lookups!)
IndexWeight          nvarchar   -- Index weight
Status               nvarchar   -- Active status
```

**Note**: The `MC($USm)` field is used for classification purposes (Large/Mid/Small cap categories). For actual market cap queries, always use the `Market_Data` table (MKT_CAP column).

**Example Query**:
```sql
-- Get all large-cap stocks in VN-Index
SELECT Ticker, Sector, [MC($USm)]
FROM Sector_Map
WHERE VNI = 'Y' AND McapClassification = 'Large-cap'
```

---

## Data Quality Notes

### SQL Data Types
- ✅ **All numeric fields are float**: FA_Quarterly, FA_Annual, BankingMetrics, Market_Data
- Dates: mix of `datetime2`, `date`, and `nvarchar` formats

### Common Gotchas

1. **Direct calculations everywhere**:
   ```sql
   -- All tables - direct use
   SELECT VALUE * 1.1, YoY * 100, ROE / NPL
   FROM FA_Quarterly
   ```

2. **Multiple Date Formats**:
   - `datetime2` format in MarketIndex, BankingMetrics
   - `date` format in Market_Data, commodity price tables
   - `nvarchar` format in FA_Quarterly/FA_Annual (quarter strings like "2024Q3")

3. **Ticker vs TICKER**: Column names may vary case (case-insensitive in SQL)

4. **YEARREPORT/LENGTHREPORT Format** ⚠️:
   - Used in banking/brokerage tables for quarterly data
   - **LENGTHREPORT**: 1 = Q1, 2 = Q2, 3 = Q3, 4 = Q4
   - **ACTUAL = 1**: Filter for actual data (vs estimates)
   - **Getting latest quarter**: Use `MAX(DATE)` with `ACTUAL = 1`
   ```sql
   -- Always filter by ACTUAL = 1 for real data
   SELECT * FROM BankingMetrics
   WHERE TICKER = 'VCB'
     AND YEARREPORT = 2025
     AND LENGTHREPORT = 3
     AND ACTUAL = 1
   ```

5. **Commodity Ticker Mapping** ⚠️ **CRITICAL**:
   - Commodity tables (Steel, Metals, etc.) use `Ticker` for **commodity identifiers**, NOT stock symbols
   - Always query `Ticker_Reference` first to find which table contains a commodity:
     ```sql
     -- Get commodity details from Ticker_Reference
     SELECT tr.Ticker, tr.Name, tr.Sector, s.Price, s.Date
     FROM Steel s
     JOIN Ticker_Reference tr ON s.Ticker = tr.Ticker
     WHERE tr.Name LIKE '%Iron Ore%'
     ```

6. **KEYCODE Variations**:
   - `NPATMI` vs `NPAT` (different metrics)
   - `Net_Revenue` vs `Revenue` (check exact spelling)

7. **Market Cap Lookups** ⚠️:
   - Always use `Market_Data` table (MKT_CAP column) for market cap queries
   - Do NOT use `Sector_Map.MC($USm)` - that field is for classification only
   - `Market_Data` has daily time-series data with MKT_CAP in VND (float - no TRY_CAST needed!)
   - `Sector_Map.MC($USm)` is a static reference in USD millions (nvarchar)

---

## Quick Reference: "Where Do I Find...?"

| Question | Table | Key Fields |
|----------|-------|------------|
| "NPATMI for ticker MWG" | `FA_Quarterly` | `TICKER`, `KEYCODE`, `VALUE` |
| **"Check Iron Ore prices"** ⚠️ | **`Ticker_Reference` first!** → then `Metals` | `Name`, `Sector`, `Ticker` |
| "VNINDEX performance" | `MarketIndex` | `COMGROUPCODE`, `INDEXVALUE`, `TRADINGDATE` |
| "Market breadth / Foreign flows" | `MarketIndex` | `TOTALSTOCKUPPRICE`, `FOREIGNBUYVALUETOTAL` |
| "Bank ROE comparison" | `BankingMetrics` | `TICKER`, `ROE`, `DATE` |
| "Market cap for all stocks" | `Market_Data` | `TICKER`, `MKT_CAP`, `TRADE_DATE` |
| "P/E, P/B, valuation ratios" | `Market_Data` | `TICKER`, `PE`, `PB`, `PS`, `EV_EBITDA` |
| "Stock prices (OHLCV)" | `Market_Data` | `TICKER`, `PX_LAST`, `PX_HIGH`, `PX_LOW`, `VOLUME` |
| "What sector is HPG in?" | `Sector_Map` | `Ticker`, `Sector`, `L1`, `L2` |
| "Commodity price history" | Commodity tables (`Steel`, `Metals`, etc.) | `Ticker`, `Date`, `Price` |
| "Which table has a commodity?" | `Ticker_Reference` | `Name`, `Sector` |
| "Analyst forecasts for VNM" | `Forecast` | `TICKER`, `KEYCODE`, `VALUE`, `FORECASTDATE` |
| "Target price for stock" | `Forecast` (KEYCODE LIKE '%.Target_Price') | `TICKER`, `VALUE`, `RATING` |
| "Consensus forecasts" | `Forecast_Consensus` | `TICKER`, `KEYCODE`, `VALUE` |

---

## Key Integration Points

| Data Need | Table |
|-----------|-------|
| Stock financials | `FA_Quarterly`, `FA_Annual` |
| Analyst forecasts | `Forecast`, `Forecast_Consensus` |
| Target prices & ratings | `Forecast` (broker-specific KEYCODE) |
| Commodity prices | `Steel`, `Metals`, `Energy`, etc. |
| Commodity price lookup | `Ticker_Reference` (commodities only!) |
| Stock sector mapping | `Sector_Map` |
| Valuation multiples (PE, PB, PS) | `Market_Data` |
| Market cap | `Market_Data` (MKT_CAP column) |
| Stock prices (OHLCV) | `Market_Data` |
| Market indices (VNINDEX, VN30, etc.) | `MarketIndex` |
| Banking metrics | `BankingMetrics` |

---

**For AI Agents**: This documentation provides table schemas and simple examples. For complex query patterns (multi-table JOINs, window functions, aggregations, rankings), see **MCP_SQL_QUERY_BEST_PRACTICES.md**.
