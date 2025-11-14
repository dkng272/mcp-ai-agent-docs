# IRIS Manual Data - Schema & Documentation

**Last Updated**: 2025-11-12
**Table**: `dbo.iris_manual_data`
**Purpose**: Manually collected high-frequency operational and market data

---

## Table Overview

**Dual-purpose table** containing:
1. **Commodity & Macro Data**: Prices, rates, trade statistics (13 categories)
2. **Alternative Data**: Company-specific operational metrics (46 stock tickers)

**Data Coverage**:
- 158,021 records
- 95 unique tickers/series
- Date range: 2000-2025
- Frequency: Daily to Monthly

---

## Table Schema

| Column | Data Type | Description |
|--------|-----------|-------------|
| `Category` | varchar | Category grouping (NULL for stock tickers) |
| `Ticker` | varchar | Series identifier (stock ticker or commodity name) |
| `KeyCode` | varchar | Metric type (varies by category/ticker) |
| `Frequency` | varchar | Update frequency (Daily, Weekly, Monthly) |
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

### 1. Hog (11 tickers, 36,804 records)
Vietnamese hog prices by region and type:
- **By region**: North, Middle, South
- **By seller type**: Corporate vs Farmer prices
- **By category**: Standard hog, 20kg piglet, baby pig (6-7kg)
- **Frequency**: Daily
- **Unit**: VND/kg or VND/con (per pig)

**Tickers**:
- Hog_corporate_South, Hog_corporate_North, Hog_corporate_Middle
- Hog_farmer_South, Hog_farmer_North, Hog_farmer_Middle
- Hog_corporate_20kg_South
- Hog_corporate_baby_pig_6_7kg_North

### 2. Chemical (3 tickers, 16,612 records)
Phosphorus prices from China:
- **Thermal_phospho_China**: Thermal process phosphorus
- **Yellow_phospho_China**: Yellow phosphorus
- **Frequency**: Daily
- **Purpose**: Input cost indicator for fertilizer producers

### 3. Interest Rate (3 tickers, 12,095 records)
Vietnam banking system rates:
- **Deposit_Rate**: 3M, 6M, 12M tenor deposit rates
- **ON_Interbank_Rate**: Overnight interbank rate (Price & Volume)
- **Frequency**: Monthly
- **Purpose**: Track Vietnam interest rate environment

### 4. Fertilizer (2 tickers, 8,931 records)
Urea spread analysis:
- **Global_Urea_Spread**: Global urea price spread
- **Urea_Simulated_Spread**: Simulated spread model
- **Frequency**: Daily
- **Purpose**: Key input cost for agricultural sector

### 5. BSR Crack (4 tickers, 6,917 records)
Refinery crack spreads for petroleum products:
- **Products**: Gasoline_92, Diesel, Fuel_oil, etc.
- **Frequency**: Daily
- **Purpose**: Margin indicator for oil refineries

### 6. Energy (1 ticker, 4,584 records)
- **JETFUEL**: Jet fuel prices
- **Frequency**: Daily
- **Purpose**: Aviation cost indicator

### 7. Vietnam Macro (3 tickers, 1,962 records)
Banking system-wide aggregates:

**Total_Credit** (226 records, 2005-2023):
- Total_Credit: Total outstanding credit in banking system
- Credit_Growth_YoY: Year-over-year growth rate
- Credit_Growth_YTD: Year-to-date growth rate

**Total_Deposit** (216 records, 2005-2022):
- Total_Deposit: Total deposits in banking system
- Deposit_Growth_YoY: Year-over-year growth rate
- Deposit_Growth_YTD: Year-to-date growth rate

**Total_M2** (216 records, 2005-2022):
- Total_M2: Money supply M2
- M2_Growth_YoY: Year-over-year growth rate
- M2_Growth_YTD: Year-to-date growth rate

**Frequency**: Monthly
**Purpose**: Track Vietnam banking system liquidity and credit expansion

### 8. Agriculture (6 tickers, 960 records)
Rice price benchmarks (2000-2022):

**Vietnam domestic** (3 tickers):
- Rice_AnGiang, Rice_BacLieu, Rice_DongThap
- Mekong Delta provinces (key rice producing regions)

**Vietnam export** (1 ticker):
- Rice_VN_Ex_FAO: FAO export price benchmark

**Thailand comparison** (2 tickers):
- Rice_Thailand_D: Domestic price
- Rice_Thailand_Ex: Export price

**Frequency**: Monthly
**Unit**: USD/ton
**Purpose**: Compare Vietnam vs Thailand rice competitiveness + regional price variations

### 9. Textile (8 tickers, 959 records)
Vietnam textile & garment export values (2010-2023):

**By product**:
- VN_Textiles_garments_Export: Garments
- VN_Foot_wears_Export: Footwear
- VN_Yarn_Export: Yarn
- VN_Textile_leather_footwear_materials_Export: Raw materials

**By destination market**:
- VN_Textile_Export_US: Exports to United States
- VN_Textile_Export_EU: Exports to European Union
- VN_Textile_Export_Japan: Exports to Japan

**Total**:
- VN_Total_Textiles_Export: Aggregate exports

**Frequency**: Monthly
**Unit**: USD (export value)
**Purpose**: Track Vietnam's textile/garment export performance - key export industry

### 10. Yarn (1 ticker, 290 records)
Vietnam yarn imports (2011-2023):
- **VN_Yarn_Import**:
  - KeyCode: "Volume" (tons) and "Value" (USD)
- **Frequency**: Monthly
- **Purpose**: Track raw material imports for textile/garment manufacturing

### 11. Cotton (1 ticker, 290 records)
Vietnam cotton imports (2011-2023):
- **VN_Cotton_Import**:
  - KeyCode: "Volume" (tons) and "Value" (USD)
- **Frequency**: Monthly
- **Purpose**: Track raw material imports for textile/garment manufacturing

### 12. Ure future (5 tickers, 1,656 records)
- Urea futures prices
- Related to fertilizer market

### 13. Empty Category (2 tickers, 6,561 records)
- **Gold_SJC**: Gold prices in Vietnam
  - KeyCode: "Price_Bid" and "Price_Ask"

---

## Part 2: Stock Ticker Alternative Data

**Filter**: `WHERE Category IS NULL`

**46 stock tickers** with high-frequency operational metrics providing early signals before quarterly earnings.

### Retail & Consumer (2 tickers)

#### MWG - Mobile World (10 KeyCodes)
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
- Monthly_NPAT: Monthly net profit
- Revstore_BHX: Revenue per BHX store

**Frequency**: Monthly

#### PNJ - Phu Nhuan Jewelry (2 KeyCodes)
- Monthly_Sales: Monthly sales revenue
- Monthly_NPATMI: Monthly net profit

**Frequency**: Monthly

### Seafood Export (3 tickers)

Export-focused companies with detailed trade data by destination:

#### VHC - Vinh Hoan (26 KeyCodes) - Most detailed
#### ANV - An Viet (14 KeyCodes)
#### IDI - IDI Corp (13 KeyCodes)

**Metrics for each destination** (China, EU, US, Total):
- Export_{Destination}_Price: Price per ton
- Export_{Destination}_Volume: Export volume in tons
- Export_{Destination}_Value: Export value in USD
- Export_{Destination}_Price_Monthly / Volume_Monthly / Value_Monthly: Monthly aggregates

**Input costs**:
- Input_price: Raw material cost in VND
- Input_Price_USD: Raw material cost in USD

**Frequency**: Weekly (VHC), varies by company
**Purpose**: Early indicator of export performance and margins before quarterly earnings

### Aviation (1 ticker)

#### ACV - Airport Corporation of Vietnam (4 KeyCodes)
Passenger traffic:
- Domestic_passengers: Domestic passenger count
- International_passengers: International passenger count
- Total_passengers: Total passengers
- Air_cargo: Air cargo volume

**Frequency**: Monthly
**Purpose**: Leading indicator for tourism/aviation sector earnings

### Banking Sector (25+ tickers)

Major banks with operational metrics:
**VCB, TCB, MBB, BID, CTG, ACB, HDB, TPB, STB, VPB, VIB, LPB, EIB, OCB, MSB, SHB, SSB, ABB, BVB, NAB, KLB, PTB, DBC, NVB, VCS, SGB, BAB, VBB, VAB, PGB**

**Common KeyCodes**:
- **CAR**: Capital Adequacy Ratio
- **NIM_Quarterly**: Net Interest Margin (quarterly)
- **NIM_Yearly** or **NIM_ Yearly**: Net Interest Margin (annual)
- **STF_MLTL**: Staffing/headcount metrics

**VCB-specific**:
- 12M_Deposit: 12-month deposit rate
- 6M_Deposit: 6-month deposit rate

**Frequency**: Monthly/Quarterly
**Purpose**: Track key banking metrics between quarterly earnings reports

### Steel & Industrial (1 ticker)

#### HPG - Hoa Phat Group (10 KeyCodes)
Steel market prices:
- Spot_VN_HRC: Vietnam Hot Rolled Coil spot price
- Spot_VN_LS: Vietnam Long Steel spot price
- Spot_China_HRC: China HRC spot price
- Spot_China_LS: China LS spot price

15-day moving averages:
- VN_HRC_MA15, VN_LS_MA15
- China_HRC_MA15, China_LS_MA15

Input costs:
- EAF: Electric Arc Furnace cost
- EAF_MA15: EAF 15-day moving average

**Frequency**: Daily
**Purpose**: Track steel market dynamics and margin indicators for HPG

### Fertilizer & Chemicals (3 tickers)

#### DCM, DPM (1 KeyCode each)
- Ure_DCM: DCM urea price
- Ure_DPM: DPM urea price

**Frequency**: Daily
**Purpose**: Fertilizer market price tracking

#### BSR - Binh Son Refining (1 KeyCode)
- Refinery operations metrics

### Oil & Gas (2 tickers)

#### PVT - PetroVietnam Transportation (3 KeyCodes)
Transportation-related metrics

#### PVD - PetroVietnam Drilling (2 KeyCodes)
Drilling operations metrics

### Other Companies (8+ tickers)
TNG, CTR, QNS, VTP and others with 1-5 KeyCodes each for company-specific operational tracking.

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
WHERE Ticker = 'MWG'
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

**For complex query patterns and best practices, see**: [MCP_SQL_QUERY_BEST_PRACTICES.md](MCP_SQL_QUERY_BEST_PRACTICES.md)

---

## Use Cases

### 1. Alternative Data Trading
High-frequency operational metrics provide early signals before quarterly earnings:
- MWG store expansion tracking
- ACV passenger traffic trends
- VHC/ANV/IDI export volume changes
- Banking sector NIM/CAR movements

### 2. Commodity Price Analysis
Track commodity price trends across multiple categories:
- Hog prices (regional comparison)
- Rice prices (Vietnam vs Thailand)
- Fertilizer/chemical input costs
- Steel market dynamics

### 3. Banking System Monitoring
Macro-level banking indicators:
- Total credit/deposit growth (Vietnam Macro category)
- Interest rate environment (Interest Rate category)
- Individual bank metrics (Banking sector tickers)

### 4. Export Trade Analysis
Vietnam's key export industries:
- Textile/garment exports by product and destination
- Seafood exports by destination (VHC, ANV, IDI)
- Raw material imports (yarn, cotton)

### 5. Input Cost Tracking
Monitor raw material costs for margin analysis:
- Phosphorus (for fertilizer producers)
- Urea (for agricultural sector)
- EAF costs (for steel producers)
- Jet fuel (for aviation)

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
    AND QUARTER(f.DATE) = QUARTER(i.Date)
WHERE f.TICKER = 'MWG'
    AND f.KEYCODE = 'Net_Revenue'
ORDER BY f.DATE DESC
```

### Combine with Market Data
```sql
-- Example: Banking NIM trends vs stock price
SELECT
    i.Date,
    i.Value AS NIM,
    m.CLOSE_PRICE
FROM dbo.iris_manual_data i
LEFT JOIN dbo.Market_Data m
    ON i.Ticker = m.TICKER
    AND i.Date = m.TRADE_DATE
WHERE i.Ticker = 'VCB'
    AND i.KeyCode = 'NIM_Quarterly'
ORDER BY i.Date DESC
```

---

## Data Quality Notes

1. **Frequency Varies**: Check `Frequency` column - mix of Daily, Weekly, Monthly
2. **Measure Units**: Always check `MeasureUnit` for correct interpretation
3. **Date Ranges**: Not all tickers have continuous data - use MIN/MAX date queries
4. **Category NULL**: Intentional for stock tickers - use as filter criterion
5. **KeyCode Consistency**: KeyCode naming may vary slightly between tickers

---

## Related Documentation

- **[DATABASE_SCHEMA_SQL_ONLY.md](DATABASE_SCHEMA_SQL_ONLY.md)**: Complete database schema
- **[MCP_SQL_QUERY_BEST_PRACTICES.md](MCP_SQL_QUERY_BEST_PRACTICES.md)**: Query patterns and optimization
- **[VIETNAM_MACRO_DATA_SCHEMA_AND_BEST_PRACTICES.md](VIETNAM_MACRO_DATA_SCHEMA_AND_BEST_PRACTICES.md)**: CEIC macro data guide

---

**Last Verified**: 2025-11-12
**Data Coverage**: 2000-2025 (varies by ticker/category)
**Total Records**: 158,021
