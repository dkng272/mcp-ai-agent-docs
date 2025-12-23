# Database Schema Documentation

**Last Updated**: 2025-12-23 (Added macro_research_embeddings collection and mongo_vector_search tool)
**AI Agent Guide**: This document provides table schemas and simple query examples. For complex query patterns, see **QUERY_BEST_PRACTICES.md**.

---

> **📘 For Complex Queries**: This document covers table schemas and simple examples. For advanced query patterns (multi-table JOINs, window functions, nested derived tables, complex aggregations), see **[QUERY_BEST_PRACTICES.md](./QUERY_BEST_PRACTICES.md)**.

---

## Table of Contents
1. [Overview](#overview)
2. [Azure SQL - dclab](#azure-sql---dclab)
3. [MongoDB - IRIS](#mongodb---iris)
4. [Common Query Patterns](#common-query-patterns)

---

## Overview

### Access Methods
- **Azure SQL**: MCP Server (`mcp__MicrosoftSQL__*` tools)
  - Database: `dclab`
  - Server: `sqls-dclab.database.windows.net`
  - Tables: 44 (financial data, sector operations, commodities, market intelligence, banking rates, bonds, analyst comments, portfolios)

- **MongoDB**: MCP Server (`mcp__claudetrade-mcp__mongo_*` tools)
  - Database: `IRIS`
  - Collections: 8 (analyst models, real estate projects, interbank updates, lending rates, MoC data, AI knowledge base, macro research articles, vector embeddings)

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

**Example**: `SELECT TICKER, DATE, VALUE, YoY FROM FA_Quarterly WHERE KEYCODE = 'NPATMI' AND YEAR = 2024`

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

**Example**: `SELECT TICKER, DATE, VALUE, RATING FROM Forecast WHERE KEYCODE = 'NPATMI' AND DATE = '2025'`

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

**Example**: `SELECT TICKER, KEYCODE, DATE, VALUE FROM Forecast_Consensus WHERE KEYCODE LIKE '%.NPATMI' AND DATE = '2025'`

---

#### `Forecast_history`
**Purpose**: Historical NPATMI forecasts by ticker with timestamp tracking

**Schema**:
```sql
Ticker        nvarchar   -- Stock ticker
Year          int        -- Forecast year
Quarter       int        -- Quarter (1-4 for quarterly, 5 for annual)
KeyCode       nvarchar   -- Metric identifier (currently NPATMI only)
Value         decimal    -- Forecast value
CreatedDate   datetime2  -- When forecast was created
```

**Key Notes**:
- `Quarter = 5` indicates **annual** forecast (not Q5)
- Currently contains only `NPATMI` forecasts
- Tracks forecast changes over time via `CreatedDate`

**Data Coverage**: 2,882 records

**Example**: `SELECT Ticker, Year, Quarter, Value, CreatedDate FROM Forecast_history WHERE Ticker = 'VNM' ORDER BY Year DESC, Quarter DESC`

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
VOLUME              float          -- Trading volume (matched)
VALUE               float          -- Trading value (matched)

-- Deal/Block Trade Data
TOTALDEALVOLUME     float          -- Block/deal trade volume
TOTALDEALVALUE      float          -- Block/deal trade value

-- Market Cap & Shares
MKT_CAP             float          -- Market capitalization in VND
SHARES              float          -- Shares outstanding

-- Foreign Trading - Matched Orders
FOREIGNBUYVALUEMATCHED    float    -- Foreign buy value (matched)
FOREIGNBUYVOLUMEMATCHED   float    -- Foreign buy volume (matched)
FOREIGNSELLVALUEMATCHED   float    -- Foreign sell value (matched)
FOREIGNSELLVOLUMEMATCHED  float    -- Foreign sell volume (matched)

-- Foreign Trading - Deal/Block Orders
FOREIGNBUYVALUEDEAL       float    -- Foreign buy value (deal/block)
FOREIGNBUYVOLUMEDEAL      float    -- Foreign buy volume (deal)
FOREIGNSELLVALUEDEAL      float    -- Foreign sell value (deal)
FOREIGNSELLVOLUMEDEAL     float    -- Foreign sell volume (deal)

-- Foreign Trading - Total (Matched + Deal)
FOREIGNBUYVALUETOTAL      float    -- Foreign buy value (total)
FOREIGNBUYVOLUMETOTAL     float    -- Foreign buy volume (total)
FOREIGNSELLVALUETOTAL     float    -- Foreign sell value (total)
FOREIGNSELLVOLUMETOTAL    float    -- Foreign sell volume (total)

-- Foreign Ownership Room
FOREIGNTOTALROOM          float    -- Total foreign ownership room
FOREIGNCURRENTROOM        float    -- Current available foreign room

-- Metadata
UPDATE_TIMESTAMP    datetime       -- Last update time
```

**Key Features**:
- ✅ All numeric fields are **float** (no TRY_CAST required)
- ✅ Clean column names (PE instead of [P/E], no special characters)
- ✅ Combines 5 data types: valuation, price/volume, market cap, foreign flows, ownership room
- ✅ Date type is `date` (clean, simple)
- ✅ **Stock-level foreign flow data** (previously only available at index level in MarketIndex)
- ⚠️ Do NOT use `MC($USm)` from `Sector_Map` - use `MKT_CAP` from this table

**Example**: `SELECT TICKER, TRADE_DATE, PE, PB, MKT_CAP, PX_LAST FROM Market_Data WHERE TICKER = 'HPG' ORDER BY TRADE_DATE DESC`

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

**Example**: `SELECT TRADINGDATE, INDEXVALUE, PERCENTINDEXCHANGE FROM MarketIndex WHERE COMGROUPCODE = 'VNINDEX' ORDER BY TRADINGDATE DESC`

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

**Example**: `SELECT TICKER, ROE, NPL, NIM, CASA FROM BankingMetrics WHERE YEARREPORT = 2024 AND LENGTHREPORT = 3 AND ACTUAL = 1`

---

#### `Banking_Comments` / `Brokerage_Comments`
**Purpose**: Analyst commentary on banking/brokerage results

**Common columns**: `TICKER`, `YEARREPORT`, `LENGTHREPORT`, `DATE`, commentary fields

---

#### `Bank_Deposit_Rates`
**Purpose**: Deposit interest rates by bank and tenure

**Schema**:
```sql
Bank              nvarchar   -- Bank name (e.g., "VCB", "Techcombank", "BIDV")
Date              datetime   -- Rate effective date
Type              nvarchar   -- Deposit type
Early_withdrawal  float      -- Early withdrawal rate
rate_1            float      -- 1-month rate
rate_3            float      -- 3-month rate
rate_6            float      -- 6-month rate
rate_9            float      -- 9-month rate
rate_12           float      -- 12-month rate
rate_13           float      -- 13+ month rate
```

**Banks Covered**: 46 banks including VCB, BIDV, Techcombank, MB, ACB, TPBank, VPBank, HDBank, Sacombank, foreign banks (HSBC, Standard Chartered, Shinhan, UOB)

**Data Coverage**: 14,499 records

**Example**: `SELECT Bank, rate_6, rate_12 FROM Bank_Deposit_Rates WHERE Date = (SELECT MAX(Date) FROM Bank_Deposit_Rates)`

---

#### `Bank_Writeoff`
**Purpose**: Bank loan write-off data

**Schema**:
```sql
TICKER      nvarchar   -- Bank ticker
DATE        nvarchar   -- Period (e.g., "2024Q3")
Writeoff    float      -- Write-off amount
```

**Data Coverage**: 621 records

**Example**: `SELECT TICKER, DATE, Writeoff FROM Bank_Writeoff WHERE DATE = '2024Q3' ORDER BY Writeoff DESC`

---

### Commodity Price Table

#### `Commodity`
**Purpose**: Centralized commodity price data for all sectors

**Schema**:
```sql
Ticker    varchar     -- Commodity identifier (NOT the same as stock tickers!)
Sector    varchar     -- Commodity sector/category
Date      date        -- Price date
Price     decimal     -- Commodity price
```

**IMPORTANT - Ticker Mapping**:
- The `Commodity` table uses `Ticker` column for commodity identifiers (NOT stock symbols!)
- Use `Ticker_Reference` table to map commodity names to ticker identifiers
- **Example**: Query `Ticker_Reference` to find "Iron Ore" → Returns Ticker "ORE62" in Sector "Metals"

**Available Sectors** (10 total):
- `Agricultural` - Grains, soybeans, corn, wheat, sugar
- `Chemicals` - Caustic soda, PVC, MEG
- `Energy` - Crude oil, coal, gas, jet fuel
- `Fertilizer` - Urea, DAP, NPK
- `Livestock` - Hog, cattle prices
- `Metals` - Aluminum, copper, zinc, alumina
- `Power` - Electricity prices
- `Shipping_Freight` - Container rates
- `Steel` - HRC, scrap, rebar
- `Textile` - Cotton, polyester

**Example**: `SELECT Date, Price, Ticker FROM Commodity WHERE Sector = 'Steel' ORDER BY Date DESC`

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

**Data Coverage**: 37K records, 25 tickers

**Example**: `SELECT DISTINCT Ticker, KeyCode FROM iris_manual_data WHERE Category IS NULL` (stock tickers)

---

### Sector Operational Data Tables

#### `Steel_data`
**Purpose**: Vietnam steel industry production, consumption, and inventory by company and product category

**Schema**:
```sql
Company Name    nvarchar   -- Steel company name (e.g., "Hoa Sen Group", "Hoa Phat Group")
Date            date       -- Observation date
Classification  nvarchar   -- Data type: Production, Consumption, Inventory
Channel         nvarchar   -- Sales channel: Domestic, Export (empty for Production/Inventory)
ProductCat      nvarchar   -- Product category
Value           decimal    -- Metric value
```

**Product Categories (ProductCat)**:
- `Galvanized` - Galvanized steel sheets
- `Rebar` - Construction steel bars
- `Pipe` - Steel pipes
- `Hot Roll Steel Coil` - HRC
- `Cool Roll Steel Coil` - CRC
- `Electrical Steel Coil` - Electrical steel

**Key Companies**: Hoa Sen Group (HSG), Hoa Phat Group (HPG), Nam Kim Steel (NKG), Ton Dong A (GDA), Formosa Ha Tinh, Pomina Steel

**Data Coverage**: 17,647 records, 2013-2025

**Example**: `SELECT [Company Name], Date, ProductCat, Classification, Value FROM Steel_data WHERE ProductCat = 'Galvanized' ORDER BY Date DESC`

---

#### `Aviation_Operations`
**Purpose**: Vietnam airline operational metrics (passengers, market share, load factor, on-time performance)

**Schema**:
```sql
Date            date       -- Observation date
Period_type     varchar    -- Period type (Monthly, etc.)
Airline         varchar    -- Airline code/name
Metric_type     varchar    -- Metric type
Traffic_type    varchar    -- Domestic, International, N/A
Metric_value    decimal    -- Metric value
Unit            varchar    -- Measurement unit
Created_date    datetime   -- Record creation date
```

**Airlines**:
- Listed tickers: HVN (Vietnam Airlines), VJC (Vietjet Air), ACV (Airports Corp)
- Other: Bamboo, Pacific, VASCO, Vietravel

**Metric Types**:
- `Passengers` - Passenger count (Domestic/International)
- `Market_share` - Market share percentage (Domestic/International)
- `Load_factor` - Aircraft load factor
- `Flight_count` - Number of flights
- `Aircraft_count` - Fleet size
- `OTP` - On-time performance

**Data Coverage**: 1,088 records, 2019-2025

**Example**: `SELECT Date, Airline, Metric_type, Traffic_type, Metric_value FROM Aviation_Operations WHERE Metric_type = 'Passengers' ORDER BY Date DESC`

---

#### `Aviation_Airfare`
**Purpose**: Airline ticket pricing data by route, flight time, and booking period

**Schema**:
```sql
Date            date       -- Flight date
Airline         varchar    -- Airline name (Vietnam Airlines, Vietjet, etc.)
Route           varchar    -- Route code (e.g., SGN-HAN, HAN-DAD)
Flight_time     varchar    -- Departure time (e.g., "21:00", "08:30")
Booking_period  varchar    -- Booking lead time (1Q = 1 quarter ahead, etc.)
Fare            decimal    -- Ticket price
Created_date    datetime   -- Record creation date
```

**Airlines**: Vietnam Airlines, Vietjet, Bamboo Airways

**Data Coverage**: 328,807 records

**Example**: `SELECT Airline, Route, AVG(Fare) as Avg_Fare FROM Aviation_Airfare GROUP BY Airline, Route ORDER BY Route`

---

#### `Aviation_Market`
**Purpose**: Vietnam aviation market metrics (fuel prices, passenger mix by country, throughput)

**Schema**:
```sql
Date            date       -- Observation date
Metric_type     varchar    -- Metric category
Metric_name     varchar    -- Specific metric
Metric_value    decimal    -- Value
Unit            varchar    -- Measurement unit
Created_date    datetime   -- Record creation date
```

**Metric Types**:
- `Fuel_price` - Singapore jet fuel prices
- `Passenger_mix` - International passenger breakdown by country (China, Korea, Japan, Taiwan, USA, Australia, Thailand, Singapore, India, Others)
- `Throughput` - Domestic and International passenger throughput

**Data Coverage**: 308 records

**Example**: `SELECT Date, Metric_type, Metric_name, Metric_value FROM Aviation_Market ORDER BY Date DESC`

---

#### `Aviation_Revenue`
**Purpose**: Airline revenue breakdown by type (HVN, VJC)

**Schema**:
```sql
Date            date       -- Observation date
Year            int        -- Calendar year
Quarter         int        -- Quarter (1-4)
Airline         varchar    -- Airline ticker (HVN, VJC)
Revenue_type    varchar    -- Revenue category
Revenue_amount  decimal    -- Revenue value
Currency        varchar    -- Currency code
Created_date    datetime   -- Record creation date
```

**Airlines & Revenue Types**:
- **HVN**: Pax, Cargo, Charter, Other
- **VJC**: Domestic pax, International pax, Ancilliary, Charter

**Data Coverage**: 80 records

**Example**: `SELECT Airline, Year, Quarter, Revenue_type, Revenue_amount FROM Aviation_Revenue ORDER BY Year DESC, Quarter DESC`

---

#### `Power_Company_Operations`
**Purpose**: Power company operational data (generation volume, revenue, average selling price) by plant

**Schema**:
```sql
Company    varchar    -- Stock ticker (PC1, REE, HDG, GEG, POW)
Plant      nvarchar   -- Power plant name
Date       date       -- Observation date
Metric     varchar    -- Metric type: volume, revenue, asp, contracted, mobilized
Value      decimal    -- Metric value
```

**Companies**: PC1, REE, HDG, GEG, POW

**Metrics**:
- `volume` - Generation volume (MWh)
- `revenue` - Revenue
- `asp` - Average selling price
- `contracted` - Contracted capacity (POW)
- `mobilized` - Mobilized capacity (POW)
- `npat` - Net profit (REE)

**Data Coverage**: 2,753 records, 2018-2025

**Example**: `SELECT Company, Plant, Date, Metric, Value FROM Power_Company_Operations WHERE Company = 'REE' ORDER BY Date DESC`

---

#### `Power_Metrics`
**Purpose**: Vietnam power market generation mix by source and economic indicators

**Schema**:
```sql
Category     varchar   -- Category: generation, market, economic
Entity       varchar   -- Source/metric identifier
Date         varchar   -- Period (e.g., "2024Q3")
Frequency    varchar   -- Data frequency
Metric       varchar   -- Sub-metric (for market category)
Value        decimal   -- Metric value
Unit         varchar   -- Measurement unit
```

**Categories & Entities**:
- **generation**: COAL, HYDRO, GAS, RENEWABLES, IMPORT, GSO_TOTAL, HND, QTP, PPC
- **market**: coal, hydro, gas, wind, solar_farm, rooftop_regular, rooftop_bidding, oil, import, other, load, price
- **economic**: gdp_growth, power_yoy, industry_yoy, ONI, thermal_alpha, hydro_alpha

**Data Coverage**: 10,240 records, 2011Q1-2025Q4

**Example**: `SELECT Date, Category, Entity, Value FROM Power_Metrics WHERE Category = 'generation' ORDER BY Date DESC`

---

#### `Power_Reservoir_Metrics`
**Purpose**: Hydropower reservoir water levels across Vietnam (38 reservoirs, 6 regions)

**Schema**:
```sql
Region      nvarchar   -- Geographic region (Vietnamese names)
Reservoir   nvarchar   -- Reservoir name
Datetime    datetime   -- Observation timestamp
Metric      varchar    -- Metric type
Value       decimal    -- Water level/volume
```

**Regions**: Tây Bắc Bộ, Đông Bắc Bộ, Bắc Trung Bộ, Nam Trung Bộ, Tây Nguyên, Đông Nam Bộ

**Major Reservoirs**: Hòa Bình, Sơn La, Lai Châu, Trị An, Ialy, Đồng Nai 3/4

**Data Coverage**: 60,619 records, 2020-2025 (sub-daily frequency)

**Example**: `SELECT Region, Reservoir, Datetime, Value FROM Power_Reservoir_Metrics ORDER BY Datetime DESC`

---

#### `Container_volume`
**Purpose**: Vietnam port container throughput by company and port

**Schema**:
```sql
Date              datetime2   -- Observation date
Region            nvarchar    -- Northern, Southern, Central
Company           nvarchar    -- Port operator ticker
Port              nvarchar    -- Port name
Total throughput  float       -- Container volume (TEUs)
```

**Regions & Key Ports**:
- **Northern**: Hai Phong ports - PHP (Đình Vũ, Tân Vũ), VSC (Green, VIP Green), SNP (Lạch Huyện), GMD (Nam Đình Vũ)
- **Southern**: Ho Chi Minh/Vung Tau - SNP (Cat Lai, TCIT), GMD (GEMALINK), SGP (Sai Gon)
- **Central**: CDN (Da Nang)

**Companies (Tickers)**: PHP, VSC, SNP, GMD, SGP, PDN, CDN, PAP, MVN

**Data Coverage**: 1,806 records, 2020-2025

**Example**: `SELECT Company, Port, Date, [Total throughput] FROM Container_volume ORDER BY Date DESC`

---

#### `BrokerageMetrics`
**Purpose**: Vietnamese securities brokerage firm financial metrics (similar structure to BankingMetrics)

**Schema**:
```sql
TICKER          nvarchar   -- Brokerage ticker (SSI, VCI, HCM, VND, etc.)
ORGANCODE       nvarchar   -- Organization code
YEARREPORT      bigint     -- Report year
LENGTHREPORT    bigint     -- Quarter (1-4)
ACTUAL          bit        -- 1 = Actual, 0 = Estimated
QUARTER_LABEL   nvarchar   -- Quarter label
STARTDATE       nvarchar   -- Period start
ENDDATE         nvarchar   -- Period end
KEYCODE         nvarchar   -- Metric identifier
KEYCODE_NAME    nvarchar   -- Metric name
VALUE           float      -- Metric value
_content_hash   nvarchar   -- Data integrity hash
```

**Key Tickers** (127 total): SSI, VCI, HCM, VND, MBS, FPTS, VIX, SHS, TVS, VCSC, BSI, CTS, AGR, FTS

**Key KEYCODEs** (201 total):

| Category | KEYCODEs |
|----------|----------|
| **Profitability** | `NPATMI`, `NPAT`, `PBT`, `EBIT`, `ROE`, `ROA` |
| **Revenue/Income** | `Net_Revenue`, `Net_Brokerage_Income`, `Net_Interest_Income`, `Net_Investment`, `Net_Fee_Income`, `Net_Margin_lending_Income`, `Net_Trading_Income`, `Net_IB_Income`, `Total_Operating_Income` |
| **Balance Sheet** | `TOTAL_Asset`, `TOTAL_Equity`, `Total_Liabilities`, `Cash_Equivalent`, `Total_Debt`, `ST_Debt`, `LT_Debt`, `Margin_Lending_book` |
| **Valuation** | `EPS`, `BVPS`, `P/E`, `P/B`, `P/S`, `MarketCap`, `Target_Price` |
| **Ratios** | `CIR`, `NIM`, `Debt_Equity`, `Asset_Equity`, `Investment_Yield`, `Dividend_Ratio`, `Interest_Coverage_Ratio` |
| **Margin Lending** | `Margin_Lending_book`, `MARGIN_LENDING_RATE`, `MARGIN_EQUITY_RATIO`, `MarginLending_Equity`, `MarginLending_TotalAsset` |
| **Trading Volume** | `Institution_shares_trading_value`, `Investor_shares_trading_value`, `Institution_bond_trading_value`, `Investor_bond_trading_value` |
| **Investment Portfolio** | `AFS_MV`, `FVTPL_MV`, `HTM_BV`, `bonds_cost`, `bonds_market_value`, `mtm_equities_cost`, `mtm_equities_market_value` |

**Data Coverage**: 495,281 records

**Example**: `SELECT TICKER, KEYCODE, VALUE FROM BrokerageMetrics WHERE KEYCODE = 'NPATMI' AND YEARREPORT = 2024 AND ACTUAL = 1`

---

#### `Brokerage_Propbook`
**Purpose**: Brokerage firm proprietary book holdings (what stocks brokerages hold in their trading books)

**Schema**:
```sql
Ticker      varchar    -- Brokerage ticker (SSI, VCI, HCM, VND, etc.)
Quarter     varchar    -- Period (e.g., "2024Q3")
Holdings    varchar    -- Stock ticker held in propbook
Keycode     varchar    -- Metric type
Value       float      -- Position value
```

**Brokerages Covered**: SSI, VCI, HCM, VND, SHS, VIX, CTS, FTS, BSI, VDS, DSE, OCBS, LBPS

**Holdings Types**: Individual stock tickers (FPT, MBB, VPB, etc.), PBT (proprietary bond trading), OTHERS (aggregated smaller positions)

**Data Coverage**: 322 records

**Example**: `SELECT Ticker, Quarter, Holdings, Value FROM Brokerage_Propbook ORDER BY Quarter DESC, Value DESC`

---

### Capital Markets Tables

#### `Bonds_Issuance`
**Purpose**: Corporate bond issuance data (issuer, rate, term, sector)

**Schema**:
```sql
id                          int        -- Record ID
issuer                      nvarchar   -- Issuer name (Vietnamese)
bond_code                   nvarchar   -- Bond identifier
industry_sector             nvarchar   -- Industry sector
issuance_value_billion_vnd  float      -- Issuance value in billion VND
issuance_method             nvarchar   -- Issuance method
interest_rate               nvarchar   -- Interest rate (fixed or floating)
issue_date                  date       -- Bond issue date
issue_date_str              nvarchar   -- Issue date string format
term_years                  int        -- Bond term in years
source_file                 nvarchar   -- Source document
report_week_start           date       -- Report period start
report_week_end             date       -- Report period end
report_month                nvarchar   -- Report month
report_year                 int        -- Report year
extracted_at                datetime   -- Data extraction timestamp
created_at                  datetime   -- Record creation timestamp
```

**Industry Sectors**: Banking, Real Estate, Securities, Manufacturing, Energy, etc.

**Data Coverage**: 847 records

**Example**: `SELECT industry_sector, issuer, issuance_value_billion_vnd, interest_rate, issue_date FROM Bonds_Issuance ORDER BY issue_date DESC`

---

### Macro Data Tables

#### `Vietnam_Credit_Deposit`
**Purpose**: Vietnam banking system aggregate credit, deposit, and M2 money supply data

**Schema**:
```sql
date                    datetime   -- Observation date
credit_vnd_bn           float      -- Total credit outstanding (billion VND)
credit_growth_ytd_pct   float      -- Credit growth YTD (%)
credit_growth_yoy_pct   float      -- Credit growth YoY (%)
deposit_vnd_bn          float      -- Total deposits (billion VND)
deposit_growth_ytd_pct  float      -- Deposit growth YTD (%)
deposit_growth_yoy_pct  float      -- Deposit growth YoY (%)
m2                      float      -- M2 money supply
m2_ytd_pct              float      -- M2 growth YTD (%)
m2_yoy_pct              float      -- M2 growth YoY (%)
pure_ldr                float      -- Pure Loan-to-Deposit Ratio
```

**Data Coverage**: 251 records (monthly data)

**Example**: `SELECT date, credit_vnd_bn, credit_growth_yoy_pct, deposit_vnd_bn, pure_ldr FROM Vietnam_Credit_Deposit ORDER BY date DESC`

#### `ceic_macro_data`
**Purpose**: CEIC economic indicator time series for Vietnam and global benchmarks (GDP, FX, interest rates, government fiscal data)

**Schema**:
```sql
id                 varchar    -- Unique series identifier (e.g., "260850901")
name               varchar    -- Descriptive series name (e.g., "Forecast: GDP: Current Prices: USD: Vietnam")
value              float      -- Numeric value
date               datetime   -- Observation date
unit               varchar    -- Measurement unit ("USD bn", "% pa", "VND bn")
last_update_time   varchar    -- When CEIC last updated this data point
```

**Data Coverage**: 545 unique series, 151K+ records (2004-2030, includes forecasts)

**Top Series by Data Points**:
- U.S. Dollar Index (id: 248650402) - 5,481 daily
- FX Spot Rate VND/USD (id: 44149401) - 5,309 daily
- VNIBOR rates (ids: 134040801, 134040901) - 5,206+ daily

**Common Name Patterns**:
- GDP: `name LIKE '%GDP%'`
- Interest rates: `name LIKE '%VNIBOR%'` or `name LIKE '%Interest%'`
- FX: `name LIKE '%FX%'` or `name LIKE '%Exchange Rate%'`
- Forecasts: `name LIKE 'Forecast:%'`

**Example**: `SELECT DISTINCT id, name, unit FROM ceic_macro_data WHERE name LIKE '%GDP%'`

---

### Portfolio Tables

#### `Model_Portfolio`
**Purpose**: Model portfolio holdings and weights (NAV-based allocation)

**Schema**:
```sql
Port          varchar    -- Portfolio name
Date          datetime2  -- Portfolio date
AssetId       nvarchar   -- Stock ticker
NAVWeight     float      -- Weight as percentage of NAV (0.01 = 1%)
UpdatedOn     datetime2  -- Last update timestamp
```

**Portfolios Available**:
- `BIG PORT INCEPTION` - Large-cap focused portfolio
- `SMALL PORT INCEPTION` - Small-cap focused portfolio

**Data Coverage**: 2,593 records

**Example**: `SELECT Port, AssetId, NAVWeight FROM Model_Portfolio WHERE Date = (SELECT MAX(Date) FROM Model_Portfolio) ORDER BY NAVWeight DESC`

---

### Analyst Commentary Tables

#### `IRIS_Company_Comments`
**Purpose**: Analyst commentary and news updates on companies (internal research notes)

**Schema**:
```sql
TICKER        nvarchar   -- Stock ticker
TITLE         nvarchar   -- Comment title/headline
DESCRIPTION   nvarchar   -- Full comment text
CATEGORY      nvarchar   -- Comment category
IMPACT        nvarchar   -- Expected impact assessment
CREATEDATE    datetime2  -- Creation timestamp
CREATEBY      nvarchar   -- Author
UPDATEDATE    datetime2  -- Last update timestamp
UPDATEBY      nvarchar   -- Updated by
ISDELETED     bit        -- Soft delete flag
```

**Categories**: Updates, Reports, Daily, Marketing, RUMORS

**Impact Values**: Positive, Negative, Neutral, In Focus

**Data Coverage**: 8,483 records

**Example**: `SELECT TICKER, TITLE, CATEGORY, IMPACT, CREATEDATE FROM IRIS_Company_Comments WHERE ISDELETED = 0 ORDER BY CREATEDATE DESC`

---

### Reference & Mapping Tables

#### `Ticker_Reference`
**Purpose**: Commodity lookup table - maps commodity names to sectors in the Commodity table

**⚠️ CRITICAL - Start Here for Commodity Price Queries**:
This is the **first table to query** when looking up commodity prices!

**Schema**:
```sql
Ticker       varchar     -- Commodity identifier (e.g., "ORE62", "AACNPC01 SMMC Index")
Name         varchar     -- Commodity name (e.g., "Ore 62", "Alumina")
Sector       varchar     -- Commodity sector (e.g., "Metals", "Steel", "Energy")
Data_Source  varchar     -- Source system
Active       bit         -- Active status
```

**Key Mapping Rules**:
1. **Sector → Commodity Sector Filter**: `Sector` column tells you which sector to filter in the `Commodity` table
   - Sector = "Metals" → Filter `Commodity WHERE Sector = 'Metals'`
   - Sector = "Steel" → Filter `Commodity WHERE Sector = 'Steel'`
   - Sector = "Energy" → Filter `Commodity WHERE Sector = 'Energy'`
2. **Name**: Human-readable commodity name (e.g., "Ore 62", "Alumina")
3. **Ticker → Price Query Key**: Use in WHERE clause of Commodity table

**Example Workflow**:
```sql
-- Step 1: Find which sector contains Iron Ore
SELECT Ticker, Name, Sector FROM Ticker_Reference
WHERE Name LIKE '%Iron%'
-- Returns: Ticker = 'ORE62', Name = 'Ore 62', Sector = 'Metals'

-- Step 2: Query the Commodity table with sector and ticker filters
SELECT Date, Price, Ticker FROM Commodity
WHERE Sector = 'Metals' AND Ticker = 'ORE62'
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
   - The `Commodity` table uses `Ticker` for **commodity identifiers**, NOT stock symbols
   - Always query `Ticker_Reference` first to find which sector contains a commodity:
     ```sql
     -- Get commodity details from Ticker_Reference and Commodity
     SELECT tr.Ticker, tr.Name, tr.Sector, c.Price, c.Date
     FROM Commodity c
     JOIN Ticker_Reference tr ON c.Ticker = tr.Ticker AND c.Sector = tr.Sector
     WHERE tr.Name LIKE '%Iron Ore%'
     ORDER BY c.Date DESC
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
| **"Check Iron Ore prices"** ⚠️ | **`Ticker_Reference` first!** → then `Commodity` | `Name`, `Sector`, `Ticker` |
| "VNINDEX performance" | `MarketIndex` | `COMGROUPCODE`, `INDEXVALUE`, `TRADINGDATE` |
| "Market breadth / Foreign flows (index)" | `MarketIndex` | `TOTALSTOCKUPPRICE`, `FOREIGNBUYVALUETOTAL` |
| "Foreign flows by stock" | `Market_Data` | `TICKER`, `FOREIGNBUYVALUETOTAL`, `FOREIGNSELLVALUETOTAL` |
| "Foreign ownership room" | `Market_Data` | `TICKER`, `FOREIGNTOTALROOM`, `FOREIGNCURRENTROOM` |
| "Bank ROE comparison" | `BankingMetrics` | `TICKER`, `ROE`, `DATE` |
| "Bank deposit rates" | `Bank_Deposit_Rates` | `Bank`, `rate_12`, `Date` |
| "Bank write-offs" | `Bank_Writeoff` | `TICKER`, `Writeoff`, `DATE` |
| "System credit/deposit growth" | `Vietnam_Credit_Deposit` | `credit_growth_yoy_pct`, `pure_ldr` |
| "Market cap for all stocks" | `Market_Data` | `TICKER`, `MKT_CAP`, `TRADE_DATE` |
| "P/E, P/B, valuation ratios" | `Market_Data` | `TICKER`, `PE`, `PB`, `PS`, `EV_EBITDA` |
| "Stock prices (OHLCV)" | `Market_Data` | `TICKER`, `PX_LAST`, `PX_HIGH`, `PX_LOW`, `VOLUME` |
| "What sector is HPG in?" | `Sector_Map` | `Ticker`, `Sector`, `L1`, `L2` |
| "Commodity price history" | `Commodity` | `Ticker`, `Sector`, `Date`, `Price` |
| "Which table has a commodity?" | `Ticker_Reference` | `Name`, `Sector` |
| "Analyst forecasts for VNM" | `Forecast` | `TICKER`, `KEYCODE`, `VALUE`, `FORECASTDATE` |
| "Target price for stock" | `Forecast` (KEYCODE LIKE '%.Target_Price') | `TICKER`, `VALUE`, `RATING` |
| "Consensus forecasts" | `Forecast_Consensus` | `TICKER`, `KEYCODE`, `VALUE` |
| "Steel production/consumption" | `Steel_data` | `Company Name`, `ProductCat`, `Classification`, `Value` |
| "Airline passengers/market share" | `Aviation_Operations` | `Airline`, `Metric_type`, `Traffic_type`, `Metric_value` |
| "Airline ticket prices" | `Aviation_Airfare` | `Airline`, `Route`, `Fare`, `Date` |
| "Airline revenue breakdown" | `Aviation_Revenue` | `Airline`, `Revenue_type`, `Revenue_amount` |
| "Passenger mix by country" | `Aviation_Market` | `Metric_type`, `Metric_name`, `Metric_value` |
| "Power company operations" | `Power_Company_Operations` | `Company`, `Plant`, `Metric`, `Value` |
| "Power generation mix" | `Power_Metrics` | `Category`, `Entity`, `Value` |
| "Reservoir water levels" | `Power_Reservoir_Metrics` | `Reservoir`, `Region`, `Value` |
| "Port container throughput" | `Container_volume` | `Company`, `Port`, `Total throughput` |
| "Brokerage firm metrics" | `BrokerageMetrics` | `TICKER`, `KEYCODE`, `VALUE` |
| "Brokerage proprietary holdings" | `Brokerage_Propbook` | `Ticker`, `Holdings`, `Value` |
| "Corporate bond issuance" | `Bonds_Issuance` | `issuer`, `issuance_value_billion_vnd`, `interest_rate` |
| "Analyst comments/news on stocks" | `IRIS_Company_Comments` | `TICKER`, `TITLE`, `IMPACT`, `CREATEDATE` |
| "NPATMI forecast history" | `Forecast_history` | `Ticker`, `Year`, `Quarter`, `Value`, `CreatedDate` |
| "Model portfolio holdings" | `Model_Portfolio` | `Port`, `AssetId`, `NAVWeight`, `Date` |

---

## Key Integration Points

| Data Need | Table |
|-----------|-------|
| Stock financials | `FA_Quarterly`, `FA_Annual` |
| Analyst forecasts | `Forecast`, `Forecast_Consensus`, `Forecast_history` |
| Target prices & ratings | `Forecast` (broker-specific KEYCODE) |
| Commodity prices | `Commodity` (filter by Sector) |
| Commodity price lookup | `Ticker_Reference` (commodities only!) |
| Stock sector mapping | `Sector_Map` |
| Valuation multiples (PE, PB, PS) | `Market_Data` |
| Market cap | `Market_Data` (MKT_CAP column) |
| Stock prices (OHLCV) | `Market_Data` |
| Foreign flows (stock-level) | `Market_Data` (FOREIGNBUY/SELL columns) |
| Foreign ownership room | `Market_Data` (FOREIGNTOTALROOM, FOREIGNCURRENTROOM) |
| Market indices (VNINDEX, VN30, etc.) | `MarketIndex` |
| Banking metrics | `BankingMetrics` |
| Bank deposit rates | `Bank_Deposit_Rates` |
| Bank write-offs | `Bank_Writeoff` |
| System credit/deposit | `Vietnam_Credit_Deposit` |
| Brokerage metrics | `BrokerageMetrics` |
| Brokerage prop book | `Brokerage_Propbook` |
| Corporate bond issuance | `Bonds_Issuance` |
| Analyst comments/news | `IRIS_Company_Comments` |
| Model portfolio weights | `Model_Portfolio` |
| Steel industry operations | `Steel_data` |
| Aviation operations | `Aviation_Operations` |
| Airline ticket prices | `Aviation_Airfare` |
| Airline revenue | `Aviation_Revenue` |
| Aviation market metrics | `Aviation_Market` |
| Power company operations | `Power_Company_Operations` |
| Power market/generation | `Power_Metrics` |
| Reservoir levels | `Power_Reservoir_Metrics` |
| Port container volume | `Container_volume` |

---

## MongoDB - IRIS

MongoDB database `IRIS` contains 6 collections for analyst models, real estate valuations, and market intelligence.

**MCP Tools**: Use `mcp__claudetrade-mcp__mongo_find` and `mcp__claudetrade-mcp__mongo_aggregate` for queries.

---

### `CompanyModels`
**Purpose**: Analyst financial models with income statement, balance sheet, segment data, and forecasts

**Document Count**: 3

**Schema**:
```javascript
{
  _id: ObjectId,
  ticker: string,              // Stock ticker (e.g., "MWG", "BCM")
  company_name: string,        // Full company name
  sector: string,              // Sector classification
  sub_sector: string,          // Sub-sector
  source_file: string,         // Excel source file name
  source_sheet: string,        // Source sheet name (e.g., "FOR_AI")
  model_version: string,       // Model quarter (e.g., "Q2 2025")
  analyst: string,             // Analyst name
  upload_date: Date,           // Upload timestamp
  created_at: Date,
  updated_at: Date,

  // Financial Data (nested by segment for retail models like MWG)
  data: {
    [SegmentName]: {           // e.g., "BachHoaXanh", "DienMayXanh"
      [MetricName]: {          // e.g., "Revenue", "Gross Profit", "EBITDA"
        unit: string,          // "VNDbn", "Count", "%"
        actual: {
          annual: { "2020": number, "2021": number, ... },
          quarterly: { "1Q20": number, "2Q20": number, ... }
        },
        forecast: {
          annual: { "2025": number, "2026": number, ... },
          quarterly: { "4Q25": number, "1Q26": number, ... }
        }
      }
    }
  },

  // Or traditional IS/BS format (BCM style)
  income_statement: {
    [year]: {
      revenue: number,
      gross_profit: number,
      ebitda: number,
      net_income: number,
      net_income_to_shareholders: number,
      // ... more line items
    }
  },
  balance_sheet: {
    [year]: {
      cash_and_equivalents: number,
      total_assets: number,
      total_equity: number,
      st_debt: number,
      lt_debt: number,
      // ... more line items
    }
  },

  // Valuation & Analysis
  valuation: {
    methodology: string,       // "SOTP", "DCF", etc.
    sotp: { components: {...}, total_nav_mil_vnd: number, nav_per_share_vnd: number }
  },
  coverage: {
    status: string,            // "Active", "Not Rated"
    recommendation: string,    // "BUY", "OUTPERFORM", etc.
    target_price: number,
    current_price: number,
    upside_percent: number
  },
  investment_thesis: {
    catalysts: [string],
    risks: [string],
    key_assumptions: [string]
  },
  industry_metrics: {...}      // Sector-specific metrics
}
```

**Example Queries**:
```javascript
// Get MWG model
mongo_find("CompanyModels", { ticker: "MWG" })

// Get all models with forecasts
mongo_find("CompanyModels", {}, { ticker: 1, model_version: 1, "coverage.target_price": 1 })
```

---

### `RealEstateProjects`
**Purpose**: Individual real estate project DCF valuations with NPV, IRR, and yearly cash flows

**Document Count**: 146

**Schema**:
```javascript
{
  _id: ObjectId,
  project_name: string,        // Project name (e.g., "Taseco Trung Van")
  ticker: string,              // Developer ticker (e.g., "TAL", "VHM", "NLG")
  location: string,            // Location (e.g., "Nam Tu Liem, Hanoi")
  project_type: string,        // "Apartment", "Township", "Mixed-use", "Industrial"
  status: string,              // "Planning", "Under Development", "Completed"

  // Ownership & Scale
  ownership_pct: number,       // Developer ownership (0-100)
  land_area_ha: number,        // Land area in hectares
  gfa_sqm: number,             // Gross Floor Area in sqm
  sellable_area_sqm: number,   // Net Sellable Area in sqm
  total_units: number,         // Number of units

  // Pricing & Costs (per sqm)
  asp_per_sqm: number,         // Average Selling Price per sqm
  land_cost_per_sqm: number,   // Land cost per sqm
  construction_cost_per_sqm: number,

  // Timeline
  launch_year: number,
  completion_year: number,

  // Financials (VND billion)
  total_revenue: number,
  total_cost: number,
  gross_margin_pct: number,

  // Valuation
  project_npv: number,         // Project NPV (VND billion)
  npv_to_company: number,      // NPV × ownership_pct
  irr: number,                 // Internal Rate of Return (%)

  // Yearly Cash Flows
  yearly_data: [
    {
      year: number,
      pct_sold: number,        // % of units sold this year
      units_sold: number,
      sales_amount: number,    // Sales value
      cash_in: number,         // Cash collections
      land_cost: number,
      construction_cost: number,
      sga_cost: number,
      tax: number,
      net_cash_flow: number,
      revenue: number,         // Revenue recognition
      cogs: number,
      gross_profit: number,
      net_profit: number,
      wacc: number,            // Discount rate (typically 10-12%)
      discount_factor: number,
      npv_contribution: number
    }
  ],

  model_version: string,
  last_updated: Date,
  updated_by: string
}
```

**Example Queries**:
```javascript
// Get all VHM projects sorted by NPV
mongo_find("RealEstateProjects", { ticker: "VHM" }, { project_name: 1, npv_to_company: 1, irr: 1 }, { npv_to_company: -1 })

// Get high-IRR projects (>30%)
mongo_find("RealEstateProjects", { irr: { $gt: 30 } }, { ticker: 1, project_name: 1, irr: 1, npv_to_company: 1 })

// Aggregate NAV by developer
mongo_aggregate("RealEstateProjects", [
  { $group: { _id: "$ticker", total_nav: { $sum: "$npv_to_company" }, project_count: { $sum: 1 } } },
  { $sort: { total_nav: -1 } }
])
```

---

### `InterbankUpdates`
**Purpose**: Daily interbank/FX market updates with exchange rates and open market operations

**Document Count**: 40

**Schema**:
```javascript
{
  _id: ObjectId,
  date: Date,                  // Update date
  created_at: Date,

  exchange_rate: {
    usd_vnd_interbank: number, // Interbank USD/VND rate
    usd_vnd_opening: number,   // Opening rate
    central_rate: number,      // SBV central rate
    ceiling_rate: number,      // Ceiling rate
    forecast_range: {
      low: number,
      high: number
    },
    trend: string,             // "increase", "decrease", "stable"
    commentary: string         // Market commentary
  },

  open_market: {
    omo: {                     // Open Market Operations
      injection: number,       // Daily OMO injection (billion VND)
      terms: [number],         // Available terms (7, 14, 28, 91 days)
      interest_rate: number,   // OMO interest rate (%)
      outstanding_balance: number,
      maturity_today: number,
      maturity_this_month: number
    },
    state_treasury: {
      outstanding_balance: number,
      maturity_today: number,
      maturity_this_month: number
    }
  }
}
```

**Example Queries**:
```javascript
// Get latest interbank update
mongo_find("InterbankUpdates", {}, {}, { date: -1 }, 1)

// Get exchange rate trends
mongo_find("InterbankUpdates", {}, { date: 1, "exchange_rate.usd_vnd_interbank": 1, "exchange_rate.trend": 1 }, { date: -1 }, 10)
```

---

### `LendingRates`
**Purpose**: Bank mortgage/lending rate offers for real estate projects

**Document Count**: 14

**Schema**:
```javascript
{
  _id: ObjectId,
  bank: string,                // Bank name (e.g., "MB Bank", "Vietcombank")
  ticker: string,              // Bank ticker (e.g., "MBB", "VCB")
  branch: string,              // Branch name (optional)
  language: string,            // "English", "Vietnamese"
  document_type: string,       // "Offer Letter - Project specific", etc.
  effective_date: Date,
  extracted_date: string,

  // Project-specific (optional)
  project: {
    name: string,
    location: string,
    full_name: string
  },

  // Loan Terms
  loan_terms: {
    ltv_ratio: number,         // Loan-to-Value ratio (0.75 = 75%)
    ltv_note: string,
    max_loan_term_months: number,
    max_loan_term_years: number,
    grace_period_months: number,
    repayment_method: string
  },

  // Interest Rates (%)
  interest_rates: {
    fixed_6_months: number,    // OR by product type:
    fixed_12_months: number,   // fixed_12_months: { land_purchase: 8.3, house_project: 7.8, ... }
    fixed_18_months: number,
    fixed_24_months: number,
    post_fixed_year_1: string, // e.g., "Reference Rate + 1.5%"
    post_fixed_remaining: string,
    reference_rate_adjustment: string
  },

  // OR detailed by product type (VCB style)
  fixed_rates: {
    "6_months": { land_purchase: number, house_project: number, consumer_car: number },
    "12_months": { ... },
    "18_months_first_6": { ... },
    "18_months_last_12": { ... }
  },

  prepayment_fees: {
    year_1_to_3: { other_sources: number, refinance_from_other_bank: number },
    // OR by package: "12_month_package": { year_1_2_3: 0.02, ... }
  },

  max_loan_terms: {
    house_land_with_certificate: string,
    house_project_no_certificate: string
  },

  disbursement_schedule: [
    { phase: number, description: string, customer_equity_pct: number, loan_disbursement_pct: number }
  ],

  customer_requirements: [string],
  customer_benefits: [string]
}
```

**Example Queries**:
```javascript
// Get all lending rates
mongo_find("LendingRates", {}, { bank: 1, "interest_rates.fixed_12_months": 1, effective_date: 1 })

// Get rates for a specific bank
mongo_find("LendingRates", { ticker: "VCB" })
```

---

### `MoCData`
**Purpose**: Ministry of Construction (MoC) quarterly real estate statistics

**Document Count**: 25

**Schema**:
```javascript
{
  _id: ObjectId,
  quarter: string,             // "3Q19", "4Q24"
  year: number,                // 2019, 2024
  quarter_num: number,         // 1, 2, 3, 4
  source: string,              // "Ministry of Construction (MoC)"

  data: {
    transactions: {
      apartments_landed_houses: number,  // Transaction count
      total: number
    },
    credit: {
      total: number            // Outstanding real estate credit (billion VND)
    }
  },

  created_at: Date,
  updated_at: Date
}
```

**Example Queries**:
```javascript
// Get all MoC data sorted by quarter
mongo_find("MoCData", {}, {}, { year: -1, quarter_num: -1 })

// Get 2024 transaction data
mongo_find("MoCData", { year: 2024 }, { quarter: 1, "data.transactions": 1, "data.credit": 1 })

// Aggregate annual transactions
mongo_aggregate("MoCData", [
  { $group: { _id: "$year", total_transactions: { $sum: "$data.transactions.total" } } },
  { $sort: { _id: -1 } }
])
```

---

### `AgentSkillSets`
**Purpose**: AI agent knowledge base with sector analysis frameworks and ticker-specific knowledge

**Document Count**: 15

**Schema**:
```javascript
{
  _id: ObjectId,
  skill_set_id: string,        // "sector:real_estate", "ticker:VHM"
  type: string,                // "sector" or "ticker"
  name: string,                // "Real Estate Sector Knowledge", "VHM Company Knowledge"

  // Sector-level fields
  sector_name: string,         // "real_estate"

  // Ticker-level fields
  ticker: string,              // "VHM"
  company_name: string,
  sector: string,
  sub_sector: string,
  business_overview: string,
  key_drivers: [string],
  metrics_to_watch: [string],

  // Knowledge Content
  key_metrics: [
    { metric: string, description: string, unit: string }
  ],
  valuation_methods: [
    { method: string, description: string, primary: boolean }
  ],
  data_units: {
    revenue: string,           // "VND billion"
    npv: string,
    percentages: string        // "Number format (75 means 75%)"
  },

  project_types: [string],     // ["Apartment", "Township", "Industrial"]
  project_statuses: [string],

  // Best Practices & Issues
  best_practices: [
    { practice: string, category: string }
  ],
  common_issues: [
    { issue: string, solution: string }
  ],

  // Analyst Notes & Knowledge Entries
  analyst_notes: [
    { note: string, context: string, added_at: Date, added_by: string }
  ],
  knowledge_entries: [
    { category: string, content: string, source: string, added_at: Date, added_by: string }
  ],

  // Ticker-specific
  historical_patterns: [
    { pattern: string, context: string, date: string }
  ],
  model_assumptions: [
    { assumption: string, context: string, date: string }
  ],
  excel_specific: {
    model_file: string,
    valuation_tab: string,
    layout_detection: string
  },

  // Metadata
  created_at: Date,
  last_updated: Date,
  created_by: string,
  version: number
}
```

**Example Queries**:
```javascript
// Get sector knowledge
mongo_find("AgentSkillSets", { type: "sector" }, { skill_set_id: 1, name: 1, key_metrics: 1 })

// Get VHM ticker knowledge
mongo_find("AgentSkillSets", { skill_set_id: "ticker:VHM" })

// Get all best practices
mongo_aggregate("AgentSkillSets", [
  { $unwind: "$best_practices" },
  { $project: { skill_set_id: 1, practice: "$best_practices.practice", category: "$best_practices.category" } }
])
```

---

### `macro_research_articles`
**Purpose**: Curated macro research articles from leading research firms for RAG-based retrieval

**Document Count**: 1,085

**Schema**:
```javascript
{
  _id: ObjectId,
  title: string,               // Article title
  source: string,              // Research source ("gavekal", "goldman_sachs")
  author: string,              // Author name
  date: Date,                  // Article publication date
  url: string,                 // Article URL
  content: string,             // Full article text
  summary: string,             // AI-generated summary with thesis and key points
  series_name: string,         // Series name (optional, e.g., "Weekly Fund Flows")
  created_at: Date             // Database insertion timestamp
}
```

**Sources**: Gavekal Research, Goldman Sachs (Weekly Fund Flows, economic reports)

**Use Case**: Full-text search and semantic retrieval for macro research and market analysis

**Example**: `mongo_find("macro_research_articles", { source: "gavekal" }, { title: 1, date: 1, summary: 1 }, { date: -1 }, 10)`

---

### `macro_research_embeddings`
**Purpose**: Vector embeddings for semantic search over macro research articles (RAG retrieval)

**Document Count**: ~1,900 chunks (articles split into searchable chunks)

**Schema**:
```javascript
{
  _id: ObjectId,
  article_id: ObjectId,           // Reference to parent article in macro_research_articles
  title: string,                  // Article title
  source: string,                 // "gavekal" or "goldman_sachs"
  author: string,                 // Author name
  date: Date,                     // Article publication date
  series_name: string,            // Series name (Goldman Sachs only, e.g., "Weekly Fund Flows")
  chunk_index: number,            // Chunk position within article (0-indexed)
  chunk_text: string,             // Text content of this chunk
  embedding: [number],            // 1536-dimensional vector (OpenAI text-embedding-3-small)
  created_at: Date                // Embedding creation timestamp
}
```

**Vector Search Tool**: Use `mongo_vector_search` instead of direct queries for semantic search.

**Example Vector Search**:
```python
# Semantic search for inflation-related articles
mongo_vector_search(query="inflation outlook", days=7)

# Filter by source with date range
mongo_vector_search(query="Fed rate cuts", source="gavekal", date_from="2025-01-01")

# Search across all sources
mongo_vector_search(query="China economy tariffs", source=["gavekal", "goldman_sachs"], limit=10)
```

---

## MongoDB Quick Reference

| Question | Collection/Tool | Query |
|----------|-----------------|-------|
| "Get analyst model for MWG" | `CompanyModels` | `{ ticker: "MWG" }` |
| "List all real estate projects by NAV" | `RealEstateProjects` | Sort by `npv_to_company` desc |
| "Get VHM projects with IRR > 30%" | `RealEstateProjects` | `{ ticker: "VHM", irr: { $gt: 30 } }` |
| "Latest interbank exchange rate" | `InterbankUpdates` | Sort by `date` desc, limit 1 |
| "Bank mortgage rates" | `LendingRates` | `{}` (all) or `{ ticker: "VCB" }` |
| "Real estate transaction volume" | `MoCData` | `{}` sorted by year/quarter |
| "AI sector knowledge for real estate" | `AgentSkillSets` | `{ skill_set_id: "sector:real_estate" }` |
| "Macro research articles on Vietnam" | `macro_research_articles` | `{ content: { $regex: "Vietnam", $options: "i" } }` |
| "Find articles about Fed policy" | `mongo_vector_search` | `query="Fed rate policy", days=30` |
| "Recent China economy research" | `mongo_vector_search` | `query="China economy", source="gavekal"` |

---

**For AI Agents**: This documentation provides table schemas and simple examples. For complex query patterns, see **QUERY_BEST_PRACTICES.md**.
