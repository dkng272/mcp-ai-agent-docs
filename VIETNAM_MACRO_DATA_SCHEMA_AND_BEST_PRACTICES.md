# Vietnam Macro Data - Database Schema & Query Best Practices

**Last Updated**: 2025-11-12
**MCP Server**: VietnamMacroData (Read-Only, Table-Filtered)
**Purpose**: Schema documentation and proven query patterns for Vietnam macroeconomic data analysis

---

## Table of Contents
1. [Database Overview](#database-overview)
2. [Table Schema](#table-schema)
3. [Query Best Practices](#query-best-practices)
4. [Proven Query Patterns](#proven-query-patterns)
5. [Common Use Cases](#common-use-cases)

---

## Database Overview

### Available Table
- `dbo.ceic_macro_data` - CEIC economic indicators (545 series, 151K+ records)

### Key Statistics
- **CEIC Data**: 545 unique economic series from 2004 to 2030 (includes forecasts)
- **Update Frequency**: Daily to Monthly depending on series
- **Geographic Coverage**: Primarily Vietnam macro indicators with some global benchmarks

### Data Source
- **CEIC**: Official economic statistics (GDP, FX, interest rates, government fiscal data, inflation, trade)

---

## Table Schema

### dbo.ceic_macro_data

**Purpose**: CEIC economic indicator time series data for Vietnam and global benchmarks

**Key Columns**:

| Column | Data Type | Description | Example |
|--------|-----------|-------------|---------|
| `id` | varchar | Unique series identifier | "260850901" |
| `name` | varchar | Descriptive name of the economic series | "Forecast: GDP: Current Prices: USD: Vietnam" |
| `value` | float | Numeric value of the indicator | 666.786 |
| `date` | datetime | Observation date | 2030-12-01 |
| `unit` | varchar | Measurement unit | "USD bn", "% pa", "VND bn" |
| `last_update_time` | varchar | When CEIC last updated this data point | "2025-10-17" |

**Warehouse Columns** (metadata, generally not needed for analysis):
- `W_INSERT_DT` - Data warehouse insert timestamp
- `W_UPDATE_DT` - Data warehouse update timestamp
- `W_DELETE_FLG` - Delete flag (always 'N' for active records)
- `W_INTEGRATION_ID` - Composite key for data integration
- `W_DATASOURCE_NAME` - Always "CEIC"
- `W_BATCH_ID` - ETL batch identifier
- `W_FILENAME` - Source JSON filename

**Primary Index**: `id`, `date`

**Top Series by Data Coverage**:
1. U.S. Dollar Index (id: 248650402) - 5,481 daily observations
2. FX Spot Rate VND/USD (id: 44149401) - 5,309 daily observations
3. VNIBOR rates (ids: 134040801, 134040901, 134041001) - 5,206+ observations each

**Common Series Names**:
- GDP indicators: `name LIKE '%GDP%'`
- Interest rates: `name LIKE '%Interest%'` or `name LIKE '%VNIBOR%'`
- Foreign exchange: `name LIKE '%FX%'` or `name LIKE '%Exchange Rate%'`
- Government fiscal: `name LIKE '%Government%'`
- Forecasts: `name LIKE 'Forecast:%'`

**Query Example**:
```sql
-- Get all GDP-related series
SELECT DISTINCT id, name, unit
FROM dbo.ceic_macro_data
WHERE name LIKE '%GDP%'
ORDER BY name
```

---

## Data Filtering Guidelines

### 1. Time Period Filtering

**Best Practices for Date Filters**:
```sql
-- ✅ Recent data (last 2 years)
WHERE Date >= '2023-01-01'

-- ✅ Specific year
WHERE YEAR(Date) = 2025

-- ✅ Year-over-year comparison
WHERE Date >= DATEADD(year, -2, GETDATE())

-- ✅ Monthly/Quarterly aggregation
WHERE Date >= '2020-01-01'
GROUP BY YEAR(Date), MONTH(Date)
```

### 2. Series Identification

Use `id` for specific series, `name` for pattern matching:
```sql
-- Specific GDP forecast series
WHERE id = '260850901'

-- Multiple specific series
WHERE id IN ('260850901', '248650402', '44149401')

-- Pattern matching by name
WHERE name LIKE '%GDP%' AND name LIKE '%Forecast%'

-- Interest rate series
WHERE name LIKE '%Interest%' OR name LIKE '%VNIBOR%'

-- FX-related series
WHERE name LIKE '%FX%' OR name LIKE '%Exchange Rate%'
```

---

## Query Best Practices

### Critical Rules (Always Follow)

#### 1. All Values Are Float - No Casting Needed
The `value` column is float type. Direct arithmetic operations work without conversion.

```sql
-- ✅ CORRECT - Direct calculations
SELECT value * 1.1, value / 1000, value + 100
FROM dbo.ceic_macro_data

-- Window functions work directly
SELECT value, LAG(value) OVER (PARTITION BY id ORDER BY date)
FROM dbo.ceic_macro_data
```

#### 2. ALWAYS Use NULLIF for Division
Prevent division by zero errors in growth calculations.

```sql
-- ❌ WRONG - Division by zero risk
(Current_Value - Previous_Value) / Previous_Value * 100

-- ✅ CORRECT - Safe division
(Current_Value - Previous_Value) / NULLIF(Previous_Value, 0) * 100
```

#### 3. Use Window Functions for Time-Series Analysis
Window functions work excellently for moving averages, lag calculations, and rankings.

```sql
-- ✅ Excellent patterns
LAG(value, 1) OVER (PARTITION BY id ORDER BY date)
AVG(value) OVER (PARTITION BY id ORDER BY date ROWS BETWEEN 11 PRECEDING AND CURRENT ROW)
ROW_NUMBER() OVER (PARTITION BY id ORDER BY value DESC)
```

#### 4. Indexing Best Practices
- ✅ Single table queries work best (this MCP has only one table)
- ✅ Filter on indexed columns: `id`, `date`
- ✅ Use `name` patterns for series discovery
- ⚠️ Avoid filtering on warehouse metadata columns (`W_*` fields)

---

## Proven Query Patterns

### Pattern 1: Simple Time-Series Query

**Use Case**: Get latest values for specific economic indicators

```sql
-- Latest GDP-related indicators
SELECT TOP 10
    id,
    name,
    date,
    value,
    unit
FROM dbo.ceic_macro_data
WHERE name LIKE '%GDP%'
ORDER BY date DESC
```

**Why It Works**:
- Single table, simple filter
- Indexed columns (id, date)
- TOP N limits result set
- Pattern matching on name for series discovery

---

### Pattern 2: Time Period Aggregation

**Use Case**: Aggregate metrics by time period

```sql
-- Annual average for interest rate series
SELECT
    id,
    name,
    YEAR(date) AS Year,
    AVG(value) AS Avg_Rate,
    MIN(value) AS Min_Rate,
    MAX(value) AS Max_Rate,
    COUNT(*) AS Data_Points
FROM dbo.ceic_macro_data
WHERE name LIKE '%VNIBOR%'
    AND date >= '2020-01-01'
GROUP BY id, name, YEAR(date)
ORDER BY Year DESC, name
```

**Best Practices**:
- Filter before aggregation (WHERE clause)
- Use multiple aggregates (AVG, MIN, MAX, COUNT)
- GROUP BY all non-aggregated columns
- Use time functions (YEAR, MONTH, QUARTER)

---

### Pattern 3: Year-over-Year Growth with Derived Tables

**Use Case**: Calculate YoY growth rates

```sql
-- YoY growth in GDP by comparing annual values
SELECT TOP 10
    Current.id,
    Current.name,
    Current.Year AS Current_Year,
    Current.Avg_Value AS Current_Value,
    Previous.Avg_Value AS Previous_Value,
    (Current.Avg_Value - Previous.Avg_Value) / NULLIF(Previous.Avg_Value, 0) * 100 AS YoY_Growth_Pct
FROM (
    SELECT id, name, YEAR(date) AS Year, AVG(value) AS Avg_Value
    FROM dbo.ceic_macro_data
    WHERE name LIKE '%GDP%' AND YEAR(date) = 2024
    GROUP BY id, name, YEAR(date)
) AS Current
LEFT JOIN (
    SELECT id, YEAR(date) AS Year, AVG(value) AS Avg_Value
    FROM dbo.ceic_macro_data
    WHERE name LIKE '%GDP%' AND YEAR(date) = 2023
    GROUP BY id, YEAR(date)
) AS Previous ON Current.id = Previous.id
WHERE Previous.Avg_Value IS NOT NULL
ORDER BY YoY_Growth_Pct DESC
```

**Why This Pattern Works**:
- Derived tables isolate year-specific aggregations
- LEFT JOIN preserves all current-year series
- NULLIF prevents division by zero
- Clear, readable structure
- Filter NULL to show only comparable series

---

### Pattern 4: Window Functions for Time-Series Analysis

**Use Case**: Calculate period-over-period changes with LAG

```sql
-- Period-over-period changes in interest rates
SELECT TOP 20
    id,
    name,
    date,
    value AS Current_Value,
    LAG(value, 1) OVER (PARTITION BY id ORDER BY date) AS Previous_Value,
    (value - LAG(value, 1) OVER (PARTITION BY id ORDER BY date))
        / NULLIF(LAG(value, 1) OVER (PARTITION BY id ORDER BY date), 0) * 100 AS Change_Pct
FROM dbo.ceic_macro_data
WHERE name LIKE '%VNIBOR%'
    AND date >= '2024-01-01'
ORDER BY date DESC, name
```

**Window Function Benefits**:
- No need for self-joins
- Efficient for sequential calculations
- Works perfectly with PARTITION BY id
- Can use LAG, LEAD, AVG OVER, SUM OVER

---

### Pattern 5: Moving Average with Window Functions

**Use Case**: Calculate 12-month moving average

```sql
-- 12-month moving average for GDP forecast
SELECT TOP 10
    name,
    date,
    value AS GDP_Value,
    AVG(value) OVER (
        PARTITION BY id
        ORDER BY date
        ROWS BETWEEN 11 PRECEDING AND CURRENT ROW
    ) AS MA_12Month
FROM dbo.ceic_macro_data
WHERE id = '260850901'  -- GDP forecast series
    AND date >= '2020-01-01'
ORDER BY date DESC
```

**Why This Excels**:
- Single pass through data
- ROWS BETWEEN provides flexible window
- PARTITION BY handles multiple series
- More efficient than self-joins

---

### Pattern 6: CASE Expression for Pivoting

**Use Case**: Create columns for different years/periods

```sql
-- Compare GDP values across multiple years
SELECT TOP 10
    id,
    name,
    unit,
    MAX(CASE WHEN YEAR(date) = 2025 THEN value END) AS Value_2025,
    MAX(CASE WHEN YEAR(date) = 2024 THEN value END) AS Value_2024,
    MAX(CASE WHEN YEAR(date) = 2023 THEN value END) AS Value_2023,
    (MAX(CASE WHEN YEAR(date) = 2025 THEN value END) -
     MAX(CASE WHEN YEAR(date) = 2024 THEN value END))
        / NULLIF(MAX(CASE WHEN YEAR(date) = 2024 THEN value END), 0) * 100 AS YoY_Change_Pct
FROM dbo.ceic_macro_data
WHERE name LIKE '%GDP%'
GROUP BY id, name, unit
ORDER BY name
```

**Pattern Benefits**:
- Clean year-over-year comparison
- Single query instead of multiple
- Easy to add calculated columns
- Readable output format

---

### Pattern 7: Time Period Aggregations

**Use Case**: Aggregate by year/month/quarter

```sql
-- Monthly GDP trend analysis
SELECT
    YEAR(date) AS Year,
    MONTH(date) AS Month,
    AVG(value) AS Avg_GDP,
    MIN(value) AS Min_GDP,
    MAX(value) AS Max_GDP
FROM dbo.ceic_macro_data
WHERE id = '260850901'
    AND date >= '2020-01-01'
GROUP BY YEAR(date), MONTH(date)
ORDER BY Year DESC, Month DESC
```

**Temporal Aggregation Tips**:
- Use YEAR(), MONTH(), QUARTER() functions
- GROUP BY all date components used in SELECT
- Add HAVING for filtered aggregates
- Order by time dimensions

---

## Common Use Cases

### Use Case 1: Track GDP Growth Over Time

**Goal**: Analyze GDP forecast trends with moving averages

```sql
-- GDP forecast with YoY growth and 12-month MA
SELECT TOP 20
    date,
    value AS GDP_USD_Billion,
    unit,
    LAG(value, 1) OVER (ORDER BY date) AS Previous_Year_GDP,
    (value - LAG(value, 1) OVER (ORDER BY date))
        / NULLIF(LAG(value, 1) OVER (ORDER BY date), 0) * 100 AS YoY_Growth_Pct,
    AVG(value) OVER (ORDER BY date ROWS BETWEEN 11 PRECEDING AND CURRENT ROW) AS MA_12Month
FROM dbo.ceic_macro_data
WHERE id = '260850901'
    AND date >= '2015-01-01'
ORDER BY date DESC
```

---

### Use Case 2: Interest Rate Trend Analysis

**Goal**: Track VNIBOR interest rate trends over time

```sql
-- VNIBOR trends with moving averages
SELECT TOP 20
    id,
    name,
    date,
    value AS Rate,
    AVG(value) OVER (
        PARTITION BY id
        ORDER BY date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS MA_30Day,
    (value - LAG(value, 1) OVER (PARTITION BY id ORDER BY date))
        / NULLIF(LAG(value, 1) OVER (PARTITION BY id ORDER BY date), 0) * 100 AS Change_Pct
FROM dbo.ceic_macro_data
WHERE name LIKE '%VNIBOR%'
    AND date >= '2024-01-01'
ORDER BY date DESC, name
```

---

### Use Case 3: FX Rate Analysis

**Goal**: Track Vietnam Dong exchange rate trends

```sql
-- VND/USD exchange rate analysis
SELECT
    id,
    name,
    date,
    value AS FX_Rate,
    unit,
    AVG(value) OVER (
        PARTITION BY id
        ORDER BY date
        ROWS BETWEEN 29 PRECEDING AND CURRENT ROW
    ) AS MA_30Day,
    LAG(value, 365) OVER (PARTITION BY id ORDER BY date) AS Rate_1Year_Ago,
    (value - LAG(value, 365) OVER (PARTITION BY id ORDER BY date))
        / NULLIF(LAG(value, 365) OVER (PARTITION BY id ORDER BY date), 0) * 100 AS YoY_Change_Pct
FROM dbo.ceic_macro_data
WHERE name LIKE '%FX%' AND name LIKE '%USD%'
    AND date >= '2020-01-01'
ORDER BY date DESC
```

---

### Use Case 4: Government Fiscal Data

**Goal**: Analyze government fiscal indicators

```sql
-- Government fiscal trends
SELECT
    id,
    name,
    YEAR(date) AS Year,
    AVG(value) AS Avg_Value,
    MIN(value) AS Min_Value,
    MAX(value) AS Max_Value,
    unit
FROM dbo.ceic_macro_data
WHERE name LIKE '%Government%'
    AND date >= '2020-01-01'
GROUP BY id, name, YEAR(date), unit
ORDER BY name, Year DESC
```

---

## Performance Optimization Tips

### 1. Filter Early and Often
```sql
-- ✅ GOOD - Filter in WHERE clause
WHERE id = '260850901' AND date >= '2024-01-01'

-- ❌ SLOW - Filter after aggregation
HAVING MAX(date) >= '2024-01-01'
```

### 2. Use Indexed Columns for Filtering
**Indexed Columns (Fast)**:
- `id` - Series identifier (primary key)
- `date` - Observation date (time-series index)

**Non-Indexed Columns (Slower)**:
- `name`, `value`, `unit` - Use these after filtering by id/date
- `W_*` warehouse columns - Avoid filtering on these

### 3. Limit Result Sets with TOP
```sql
-- Always use TOP N for exploratory queries
SELECT TOP 100 id, name, date, value
FROM dbo.ceic_macro_data
```

### 4. Avoid SELECT *
```sql
-- ✅ GOOD - Select only needed columns
SELECT id, name, date, value, unit

-- ❌ SLOW - Fetches unnecessary warehouse columns
SELECT * FROM dbo.ceic_macro_data
```

---

## Common Failure Patterns (Avoid These)

### ❌ Pattern 1: Division Without NULLIF
```sql
-- WRONG - Risk of division by zero
(value - previous_value) / previous_value * 100

-- CORRECT - Safe division
(value - previous_value) / NULLIF(previous_value, 0) * 100
```

### ❌ Pattern 2: Ambiguous Date Ranges
```sql
-- WRONG - Unclear date filter
WHERE date > '2024'

-- CORRECT - Explicit date format
WHERE date >= '2024-01-01'
```

### ❌ Pattern 3: No Index Usage
```sql
-- SLOW - Non-indexed column filter only
WHERE name LIKE '%GDP%'

-- FAST - Indexed filter + pattern matching
WHERE id IN ('260850901', '468013507')
-- or: WHERE date >= '2024-01-01' AND name LIKE '%GDP%'
```

### ❌ Pattern 4: Fetching Unnecessary Columns
```sql
-- SLOW - Includes all warehouse columns
SELECT * FROM dbo.ceic_macro_data WHERE id = '260850901'

-- FAST - Only needed columns
SELECT id, name, date, value, unit
FROM dbo.ceic_macro_data WHERE id = '260850901'
```

---

## Summary Checklist

Before running a macro data query, verify:

- [ ] All division uses NULLIF(..., 0)
- [ ] Date filters use explicit format (YYYY-MM-DD)
- [ ] Filtering on indexed columns (id, date) when possible
- [ ] Using TOP N to limit exploratory queries
- [ ] Window functions for time-series calculations (LAG, AVG OVER, etc.)
- [ ] Derived tables instead of CTEs (WITH clause not supported)
- [ ] GROUP BY includes all non-aggregated SELECT columns
- [ ] Not fetching unnecessary warehouse columns (W_*)
- [ ] Series identification via id or name pattern matching

**If query fails**:
1. Verify NULLIF in all division operations
2. Check date format and range validity
3. Ensure GROUP BY includes all non-aggregated columns
4. Reduce result set with TOP N
5. Simplify complex window functions if needed

---

**Data Coverage**:
- CEIC: 2004-2030 (includes forecasts)
- 545 unique economic series
- 151K+ data points

**Last Tested**: 2025-11-12
**Test Queries**: 12+ patterns validated
**Success Rate**: 100% for properly formatted queries
