# IRIS Manual Data - Schema & Documentation

**Last Updated**: 2025-12-17
**Table**: `dbo.iris_manual_data`
**Purpose**: Manually collected high-frequency operational and market data

---

## Table Overview

**Dual-purpose table** containing:
1. **Commodity & Macro Data**: Prices and spreads (3 categories)
2. **Alternative Data**: Company-specific operational metrics (17 stock tickers)

**Data Coverage**:
- 37,314 records
- 25 unique tickers/series
- Date range: 2002-2025
- Frequency: Daily to Monthly

---

## Table Schema

| Column | Data Type | Description |
|--------|-----------|-------------|
| `Category` | varchar | Category grouping (NULL for stock tickers) |
| `Ticker` | varchar | Series identifier (stock ticker or commodity name) |
| `KeyCode` | varchar | Metric type (varies by category/ticker) |
| `Frequency` | varchar | Update frequency (Daily, Weekly, Monthly, Quarterly) |
| `MeasureUnit` | varchar | Measurement unit (kg, ton, %, USD, etc.) |
| `Date` | datetime | Observation date |
| `Value` | float | Numeric value |
| `UpdatedDate` | datetime | Last update timestamp |

**Primary Index**: `Ticker`, `KeyCode`, `Date`

**Key Filtering Pattern**:
```sql
-- Commodity/Macro data only
WHERE Category IS NOT NULL

-- Stock ticker alternative data only
WHERE Category IS NULL
```

---

## Part 1: Commodity & Macro Categories

**Filter**: `WHERE Category IS NOT NULL`

### 1. Hog (4 tickers, 9,406 records)
Vietnamese piglet prices by region and type:

| Ticker | Description | Unit |
|--------|-------------|------|
| Hog_corporate_20kg_North | 20kg piglet corporate price (North) | VND/kg |
| Hog_corporate_baby_pig_6_7kg_North | Baby pig 6-7kg corporate (North) | VND/con |
| Hog_corporate_baby_pig_6_7kg_South | Baby pig 6-7kg corporate (South) | VND/con |
| Hog_farmer_baby_pig_7_9kg_South | Baby pig 7-9kg farmer (South) | VND/con |

**KeyCodes**: Average, High, Low
**Frequency**: Daily
**Purpose**: Track piglet prices for agricultural/livestock sector

### 2. Fertilizer (2 tickers, 8,983 records)
Urea spread analysis:

| Ticker | Description |
|--------|-------------|
| Global_Urea_Spread | Global urea price spread |
| Urea_Simulated_Spread | Simulated spread model |

**KeyCode**: Price
**Frequency**: Daily
**Unit**: USD/ton
**Purpose**: Key input cost indicator for fertilizer sector (DCM, DPM)

### 3. Energy (1 ticker, 4,608 records)
| Ticker | Description |
|--------|-------------|
| JETFUEL | Jet fuel prices |

**KeyCode**: Price
**Frequency**: Daily
**Purpose**: Aviation cost indicator (ACV, VJC)

### 4. Ure India Tender (1 ticker, 19 records)
India urea tender tracking:

| KeyCode | Description |
|---------|-------------|
| Price | Tender price |
| Target_quantity | Target procurement quantity |
| Volume_secured | Volume awarded |

**Purpose**: Track India urea tenders affecting Vietnamese fertilizer exports

---

## Part 2: Stock Ticker Alternative Data

**Filter**: `WHERE Category IS NULL`

**17 stock tickers** with high-frequency operational metrics providing early signals before quarterly earnings.

### Technology (1 ticker)

#### FPT - FPT Corporation (11 KeyCodes)
FPT Software (Fsoft) monthly operational metrics:

| KeyCode | Description | Frequency |
|---------|-------------|-----------|
| Fsoft_Monthly_Rev | Monthly revenue | Monthly |
| Fsoft_Monthly_PBT | Monthly profit before tax | Monthly |
| Fsoft_Monthly_PBT_YoY | PBT year-over-year growth | Monthly |
| Fsoft_Monthly_Signed_Rev | Monthly signed contract revenue | Monthly |
| Fsoft_Monthly_Signed_Rev_YoY | Signed revenue YoY growth | Monthly |
| Fsoft_YTD_Rev | Year-to-date revenue | Monthly |
| Fsoft_YTD_PBT | Year-to-date PBT | Monthly |
| Fsoft_YTD_PBT_YoY | YTD PBT year-over-year growth | Monthly |
| Fsoft_YTD_Signed_Rev | YTD signed contract revenue | Monthly |
| Fsoft_YTD_Signed_Rev_YoY | YTD signed revenue YoY growth | Monthly |

**Unit**: VND billion
**Purpose**: Track FPT's IT services growth momentum

### Retail & Consumer (3 tickers)

#### MWG - Mobile World (9 KeyCodes)
Store count by brand:
- Store_TGDD: Thế Giới Di Động (mobile phones)
- Store_DMX: Điện Máy Xanh (consumer electronics)
- Store_BHX: Bách Hóa Xanh (grocery stores)

Revenue by segment:
- ICT_Revenue: ICT segment revenue
- CE_Revenue: Consumer electronics revenue
- BHX_Revenue: Grocery store revenue
- ICTCE_Revenue: Combined ICT + CE revenue
- Monthly_Revenue: Total monthly revenue

Performance metrics:
- Revstore_BHX: Revenue per BHX store

**Frequency**: Monthly

#### PNJ - Phu Nhuan Jewelry (2 KeyCodes)
| KeyCode | Description |
|---------|-------------|
| Monthly_Sales | Monthly sales revenue |
| Monthly_NPATMI | Monthly net profit |

**Frequency**: Monthly

#### QNS - Quang Ngai Sugar (2 KeyCodes)
| KeyCode | Description |
|---------|-------------|
| Monthly_Sales | Monthly sales revenue |
| Monthly_PBT | Monthly profit before tax |

**Frequency**: Monthly

### Seafood Export (3 tickers)

Export-focused pangasius companies with detailed trade data by destination:

#### VHC - Vinh Hoan (13 KeyCodes)
#### ANV - An Viet (13 KeyCodes)
#### IDI - IDI Corp (12 KeyCodes)

**Metrics for each destination** (China, EU, US, Total):

| KeyCode Pattern | Description |
|-----------------|-------------|
| Export_{Dest}_Price | Price per ton (USD/ton) |
| Export_{Dest}_Volume | Export volume (tons) |
| Export_{Dest}_Value | Export value (USD) |

**Input costs** (VHC, ANV only):
- Input_Price_USD: Raw material cost in USD/kg

**Frequency**: Weekly
**Purpose**: Early indicator of export performance and margins before quarterly earnings

### Fertilizer (2 tickers)

#### DCM - Ca Mau Fertilizer (3 KeyCodes)
| KeyCode | Description | Frequency |
|---------|-------------|-----------|
| Sales_Ure_Domestic | Domestic urea sales volume (tons) | Monthly |
| Sales_NPK | NPK fertilizer sales volume (tons) | Monthly |
| Sales_NPK_Export | NPK export volume (tons) | Monthly |

#### DPM - Petrovietnam Fertilizer (3 KeyCodes)
| KeyCode | Description | Frequency |
|---------|-------------|-----------|
| Sales_Ure | Urea sales volume (tons) | Quarterly |
| Sales_NPK | NPK sales volume (tons) | Quarterly |
| Gas_cost | Gas input cost | Quarterly |

### Aviation (1 ticker)

#### ACV - Airport Corporation of Vietnam (4 KeyCodes)
| KeyCode | Description |
|---------|-------------|
| Domestic_passengers | Domestic passenger count |
| International_passengers | International passenger count |
| Total_passengers | Total passengers |
| Air_cargo | Air cargo volume |

**Frequency**: Monthly
**Purpose**: Leading indicator for tourism/aviation sector earnings

### Textile & Apparel (1 ticker)

#### TNG - TNG Investment (5 KeyCodes)
| KeyCode | Description |
|---------|-------------|
| Monthly_Sales | Monthly sales revenue |
| Monthly_GrossProfit | Monthly gross profit |
| Monthly_GPM | Gross profit margin |
| Monthly_NPATMI | Monthly net profit |
| Monthly_NPM | Net profit margin |

**Frequency**: Monthly
**Purpose**: Track textile/garment manufacturer performance

### Oil & Gas (2 tickers)

#### PVT - PetroVietnam Transportation (2 KeyCodes)
| KeyCode | Description | Unit |
|---------|-------------|------|
| Crude_TC | Crude tanker time charter rate | $/day |
| ProductMR_TC | Product tanker (MR) time charter rate | $/day |

**Frequency**: Weekly
**Purpose**: Track tanker charter rates affecting PVT revenue

#### PVD - PetroVietnam Drilling (2 KeyCodes)
| KeyCode | Description |
|---------|-------------|
| SEAJU_Rate | Southeast Asia jack-up rig day rate |
| SEAJU_Ulti | Southeast Asia jack-up utilization rate |

**Frequency**: Monthly
**Purpose**: Track rig utilization and rates for drilling sector

### Building Materials (2 tickers)

#### PTB - Phu Tai (2 KeyCodes)
| KeyCode | Description |
|---------|-------------|
| Wood_Export_Value | Wood product export value |
| Quartz_Export_Value | Quartz/stone export value |

**Frequency**: Monthly

#### VCS - Vicostone (1 KeyCode)
| KeyCode | Description |
|---------|-------------|
| Quartz_Export_Value | Quartz/stone export value |

**Frequency**: Monthly

### Telecom Infrastructure (1 ticker)

#### CTR - Viettel Construction (1 KeyCode)
| KeyCode | Description | Unit |
|---------|-------------|------|
| Tower_count | Total telecom tower count | Tower |

**Frequency**: Monthly
**Purpose**: Track tower rollout for telecom infrastructure

### Logistics (1 ticker)

#### VTP - Viettel Post (2 KeyCodes)
| KeyCode | Description |
|---------|-------------|
| Service_Rev | Service revenue (quarterly) |
| Service_Rev_TTM | Service revenue trailing 12 months |

**Frequency**: Quarterly

---

## Data Access Patterns

### Basic Filtering

```sql
-- Get all commodity/macro categories
SELECT DISTINCT Category
FROM dbo.iris_manual_data
WHERE Category IS NOT NULL
ORDER BY Category

-- Get all stock tickers with alternative data
SELECT DISTINCT Ticker
FROM dbo.iris_manual_data
WHERE Category IS NULL
ORDER BY Ticker

-- Get KeyCodes for a specific ticker
SELECT DISTINCT KeyCode
FROM dbo.iris_manual_data
WHERE Ticker = 'FPT'
ORDER BY KeyCode
```

### Category-Level Analysis

```sql
-- Summary by category
SELECT Category,
       COUNT(DISTINCT Ticker) AS Ticker_Count,
       COUNT(*) AS Total_Records,
       MIN(Date) AS First_Date,
       MAX(Date) AS Last_Date
FROM dbo.iris_manual_data
WHERE Category IS NOT NULL
GROUP BY Category
ORDER BY Total_Records DESC
```

### Stock Ticker Analysis

```sql
-- Get ticker summary with KeyCode count
SELECT Ticker,
       COUNT(DISTINCT KeyCode) AS KeyCode_Count,
       COUNT(*) AS Total_Records,
       MIN(Date) AS First_Date,
       MAX(Date) AS Last_Date
FROM dbo.iris_manual_data
WHERE Category IS NULL
GROUP BY Ticker
ORDER BY KeyCode_Count DESC
```

### Example: FPT Software Monthly Metrics

```sql
SELECT Date, KeyCode, Value
FROM dbo.iris_manual_data
WHERE Ticker = 'FPT'
  AND KeyCode LIKE 'Fsoft_Monthly%'
  AND Date >= '2024-01-01'
ORDER BY Date DESC, KeyCode
```

### Example: Seafood Export Comparison

```sql
SELECT Ticker, Date,
       MAX(CASE WHEN KeyCode = 'Export_Total_Volume' THEN Value END) AS Volume,
       MAX(CASE WHEN KeyCode = 'Export_Total_Price' THEN Value END) AS Price,
       MAX(CASE WHEN KeyCode = 'Export_Total_Value' THEN Value END) AS Value
FROM dbo.iris_manual_data
WHERE Ticker IN ('VHC', 'ANV', 'IDI')
  AND KeyCode LIKE 'Export_Total%'
GROUP BY Ticker, Date
ORDER BY Date DESC, Ticker
```

---

## Use Cases

### 1. Alternative Data Trading
High-frequency operational metrics provide early signals before quarterly earnings:
- FPT Software signed contract growth
- MWG store expansion and revenue tracking
- ACV passenger traffic trends
- VHC/ANV/IDI export volume changes
- DCM/DPM fertilizer sales volumes

### 2. Commodity Price Analysis
Track commodity price trends:
- Urea spreads (for DCM, DPM margin analysis)
- Jet fuel prices (for aviation cost analysis)
- Hog/piglet prices (for livestock sector)

### 3. Export Performance Monitoring
Vietnam's key export industries:
- Seafood exports by destination (VHC, ANV, IDI)
- Textile performance (TNG)
- Building materials exports (PTB, VCS)

### 4. Oil & Gas Sector Indicators
- Tanker charter rates (PVT)
- Rig utilization rates (PVD)

---

## Integration with Other Tables

### Combine with Financial Statements
```sql
-- Example: MWG store growth vs revenue growth
SELECT
    f.DATE,
    f.VALUE AS Quarterly_Revenue,
    i.Value AS Store_Count
FROM dbo.FA_Quarterly f
LEFT JOIN dbo.iris_manual_data i
    ON f.TICKER = i.Ticker
    AND i.KeyCode = 'Store_TGDD'
    AND YEAR(f.DATE) = YEAR(i.Date)
    AND MONTH(f.DATE) = MONTH(i.Date)
WHERE f.TICKER = 'MWG'
    AND f.KEYCODE = 'Net_Revenue'
ORDER BY f.DATE DESC
```

### Combine with Market Data
```sql
-- Example: ACV passenger trends vs stock price
SELECT
    i.Date,
    i.Value AS Total_Passengers,
    m.CLOSE_PRICE
FROM dbo.iris_manual_data i
LEFT JOIN dbo.Market_Data m
    ON i.Ticker = m.TICKER
    AND CAST(i.Date AS DATE) = CAST(m.TRADE_DATE AS DATE)
WHERE i.Ticker = 'ACV'
    AND i.KeyCode = 'Total_passengers'
ORDER BY i.Date DESC
```

---

## Data Quality Notes

1. **Frequency Varies**: Check `Frequency` column - mix of Daily, Weekly, Monthly, Quarterly
2. **Measure Units**: Always check `MeasureUnit` for correct interpretation
3. **Date Ranges**: Not all tickers have continuous data - use MIN/MAX date queries
4. **Category NULL**: Intentional for stock tickers - use as filter criterion
5. **KeyCode Consistency**: KeyCode naming is consistent within each ticker

---

## Related Documentation

- **[DATABASE_SCHEMA_SQL_ONLY.md](DATABASE_SCHEMA_SQL_ONLY.md)**: Complete database schema
- **[MCP_SQL_QUERY_BEST_PRACTICES.md](MCP_SQL_QUERY_BEST_PRACTICES.md)**: Query patterns and optimization
- **[VIETNAM_MACRO_DATA_SCHEMA_AND_BEST_PRACTICES.md](VIETNAM_MACRO_DATA_SCHEMA_AND_BEST_PRACTICES.md)**: CEIC macro data guide

---

**Last Verified**: 2025-12-17
**Data Coverage**: 2002-2025 (varies by ticker/category)
**Total Records**: 37,314
