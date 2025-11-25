# MCP SQL Server Query Best Practices

**Last Updated**: 2025-11-25 (CTEs now supported, Azure CLI silent auth)
**Purpose**: Proven patterns for querying Azure SQL via Microsoft SQL MCP Server

---

> **📌 Note on Forecast Tables**: Examples below use `Forecast_Consensus` for broker forecasts (KEYCODE format: `BROKER.METRIC`) and `Forecast` for firm's internal forecasts (KEYCODE format: `METRIC` only). See DATABASE_SCHEMA_SQL_ONLY.md for complete table documentation.

---

## Executive Summary

Based on extensive stress testing with the Microsoft SQL MCP Server, this document provides concrete patterns for writing queries that work reliably with complex aggregations, joins, and calculations.

**Key Findings**:
- ✅ Window functions work excellently (use liberally)
- ✅ CTEs (WITH clause) now fully supported
- ✅ Derived tables (subqueries in FROM) work as alternative to CTEs
- ✅ 2-3 table JOINs work reliably
- ❌ Correlated subqueries in SELECT clause fail
- ❌ 4+ table JOINs become unreliable

---

## Critical Rules (Always Follow)

### 1. All Tables Now Float - No TRY_CAST Needed!

**✅ All numeric fields converted to float**:
- `FA_Quarterly`: VALUE, YoY → float
- `FA_Annual`: VALUE, YoY → float
- `BankingMetrics`: All 47 numeric columns → float
- `Market_Data`: All columns → float

```sql
-- Direct calculations everywhere
SELECT VALUE * 1.1, YoY * 100, ROE / CASA
FROM FA_Quarterly
WHERE KEYCODE = 'NPATMI'
```

### 2. ALWAYS Use NULLIF for Division

```sql
-- ❌ WRONG - Division by zero errors
(Value1 - Value2) / Value2 * 100

-- ✅ CORRECT - Safe division
(Value1 - Value2) / NULLIF(Value2, 0) * 100
```

### 3. CTEs (WITH Clause) Now Supported

**UPDATE (2025-11-25)**: CTEs are now fully supported after MCP server update.

```sql
-- ✅ NOW WORKS - CTEs enabled
WITH ForecastAvg AS (
    SELECT TICKER, AVG(VALUE) AS Avg_Forecast
    FROM Forecast
    GROUP BY TICKER
)
SELECT * FROM ForecastAvg

-- ✅ ALSO WORKS - Derived table alternative
SELECT *
FROM (
    SELECT TICKER, AVG(VALUE) AS Avg_Forecast
    FROM Forecast
    GROUP BY TICKER
) AS ForecastAvg
```

**When to use CTEs vs Derived Tables**:
- CTEs: Better readability for multi-step queries, self-referencing logic
- Derived tables: Simpler for single subquery scenarios

### 4. NO Correlated Subqueries in SELECT

```sql
-- ❌ FAILS - Correlated subquery execution error
SELECT
    f.TICKER,
    (SELECT AVG(VALUE) FROM FA_Quarterly
     WHERE TICKER = f.TICKER AND KEYCODE = 'NPATMI') AS Avg_Actual
FROM Forecast f

-- ✅ WORKS - Use JOIN instead
SELECT f.TICKER, AVG(fa.VALUE) AS Avg_Actual
FROM Forecast f
LEFT JOIN FA_Quarterly fa
    ON f.TICKER = fa.TICKER AND fa.KEYCODE = 'NPATMI'
GROUP BY f.TICKER
```

---

## Proven Query Patterns

### Pattern 1: Simple Aggregation with Group By

**Use Case**: Calculate averages, counts, stddev by ticker or category

**✅ WORKS RELIABLY**:
```sql
-- Aggregate forecasts by ticker (VALUE is float - no TRY_CAST needed)
SELECT
    f.TICKER,
    COUNT(DISTINCT f.KEYCODE) AS Broker_Count,
    AVG(f.VALUE) AS Avg_Forecast,
    STDEV(f.VALUE) AS Forecast_StdDev,
    MIN(f.VALUE) AS Min_Forecast,
    MAX(f.VALUE) AS Max_Forecast,
    (MAX(f.VALUE) - MIN(f.VALUE)) / NULLIF(AVG(f.VALUE), 0) * 100 AS Spread_Pct
FROM Forecast f
WHERE f.KEYCODE = 'NPATMI' AND f.DATE = '2025'
GROUP BY f.TICKER
HAVING COUNT(DISTINCT f.KEYCODE) >= 3
ORDER BY Avg_Forecast DESC
```

**Why It Works**:
- Single table with aggregations
- Direct calculation (Forecast.VALUE is float)
- Multiple aggregate functions
- HAVING clause for filtering aggregated results
- Safe division with NULLIF

---

### Pattern 2: Two-Table JOIN with Aggregations

**Use Case**: Compare forecasts to actual results

**✅ WORKS RELIABLY**:
```sql
-- Compare forecasts to actual results (both tables have float VALUES)
SELECT TOP 10
    f.TICKER,
    AVG(f.VALUE) AS Avg_Forecast_2025,
    AVG(fa.VALUE) AS Avg_Actual_2024,
    (AVG(f.VALUE) - AVG(fa.VALUE)) / NULLIF(AVG(fa.VALUE), 0) * 100 AS Growth_Pct
FROM Forecast f
LEFT JOIN FA_Quarterly fa
    ON f.TICKER = fa.TICKER
    AND fa.KEYCODE = 'NPATMI'
    AND fa.YEAR = 2024
WHERE f.KEYCODE = 'NPATMI' AND f.DATE = '2025'
GROUP BY f.TICKER
HAVING AVG(fa.VALUE) IS NOT NULL
ORDER BY Growth_Pct DESC
```

**Why It Works**:
- Clear JOIN conditions with multiple predicates
- Direct calculation (both VALUE fields are float)
- Aggregations work across joined tables
- HAVING filters out NULLs after aggregation

---

### Pattern 3: Three-Table JOIN (Maximum Recommended)

**Use Case**: Combine forecasts, actuals, and sector classifications

**✅ WORKS WITH CARE**:
```sql
SELECT TOP 15
    f.TICKER,
    s.Sector,
    s.L1,
    COUNT(DISTINCT f.KEYCODE) AS Broker_Count,
    AVG(f.VALUE) AS Avg_Forecast_2025,
    STDEV(f.VALUE) AS Forecast_Volatility
FROM Forecast f
LEFT JOIN Sector_Map s ON f.TICKER = s.Ticker
LEFT JOIN FA_Quarterly fa
    ON f.TICKER = fa.TICKER
    AND fa.KEYCODE = 'NPATMI'
    AND fa.YEAR = 2024
WHERE f.KEYCODE LIKE '%.NPATMI'
    AND f.DATE = '2025'
    AND s.Sector IS NOT NULL
GROUP BY f.TICKER, s.Sector, s.L1
HAVING COUNT(DISTINCT f.KEYCODE) >= 5
ORDER BY Avg_Forecast_2025 DESC
```

**Best Practices for 3-Table JOINs**:
- Use LEFT JOIN to preserve main table rows
- Add NULL checks in WHERE clause (`s.Sector IS NOT NULL`)
- Keep JOIN conditions simple and indexed (TICKER, DATE, KEYCODE)
- Limit aggregated columns to avoid complexity

**⚠️ WARNING**: Adding a 4th table (e.g., Valuation) often causes query execution failures. If you need data from 4+ tables, use derived tables.

---

### Pattern 4: Window Functions (Highly Recommended)

**Use Case**: Ranking, running averages, partitioned calculations

**✅ WORKS EXCELLENTLY**:
```sql
-- Rank forecasts within each ticker
SELECT TOP 10
    f.TICKER,
    f.KEYCODE,
    f.VALUE,
    ROW_NUMBER() OVER (PARTITION BY f.TICKER ORDER BY f.VALUE DESC) AS Forecast_Rank,
    AVG(f.VALUE) OVER (PARTITION BY f.TICKER) AS Avg_Forecast_By_Ticker,
    RANK() OVER (ORDER BY f.VALUE DESC) AS Overall_Rank
FROM Forecast f
WHERE f.KEYCODE LIKE '%.NPATMI' AND f.DATE = '2025'
ORDER BY f.TICKER, Forecast_Rank
```

**Advanced Window Function Example**:
```sql
-- Top 3 forecasts per sector using RANK
SELECT TOP 10
    TICKER, Sector, Avg_Forecast_2025, Broker_Count, Forecast_Rank
FROM (
    SELECT
        f.TICKER,
        s.Sector,
        AVG(f.VALUE) AS Avg_Forecast_2025,
        COUNT(DISTINCT f.KEYCODE) AS Broker_Count,
        RANK() OVER (PARTITION BY s.Sector ORDER BY AVG(f.VALUE) DESC) AS Forecast_Rank
    FROM Forecast f
    LEFT JOIN Sector_Map s ON f.TICKER = s.Ticker
    WHERE f.KEYCODE LIKE '%.NPATMI'
        AND f.DATE = '2025'
        AND s.Sector IS NOT NULL
    GROUP BY f.TICKER, s.Sector
) AS RankedForecasts
WHERE Forecast_Rank <= 3
ORDER BY Sector, Forecast_Rank
```

**Why Window Functions Excel**:
- No performance penalty vs GROUP BY
- Can rank, partition, and calculate running totals
- Work perfectly with derived tables
- Combine with aggregations seamlessly

---

### Pattern 5: CASE Expressions for Pivoting

**Use Case**: Create columns from row data (quarterly comparisons)

**✅ WORKS RELIABLY**:
```sql
SELECT TOP 10
    Combined.TICKER,
    Combined.Avg_Forecast_2025,
    Combined.Actual_Q3_2024,
    Combined.Actual_Q2_2024,
    Combined.Actual_Q1_2024,
    (Combined.Actual_Q3_2024 - Combined.Actual_Q2_2024) /
        NULLIF(Combined.Actual_Q2_2024, 0) * 100 AS QoQ_Growth_Q3,
    (Combined.Avg_Forecast_2025 - Combined.Actual_Q3_2024) /
        NULLIF(Combined.Actual_Q3_2024, 0) * 100 AS Forecast_vs_Q3_Growth
FROM (
    SELECT
        f.TICKER,
        AVG(f.VALUE) AS Avg_Forecast_2025,
        MAX(CASE WHEN fa.DATE = '2024Q3' THEN fa.VALUE END) AS Actual_Q3_2024,
        MAX(CASE WHEN fa.DATE = '2024Q2' THEN fa.VALUE END) AS Actual_Q2_2024,
        MAX(CASE WHEN fa.DATE = '2024Q1' THEN fa.VALUE END) AS Actual_Q1_2024
    FROM Forecast f
    LEFT JOIN FA_Quarterly fa
        ON f.TICKER = fa.TICKER
        AND fa.KEYCODE = 'NPATMI'
        AND fa.YEAR = 2024
    WHERE f.KEYCODE = 'NPATMI' AND f.DATE = '2025'
    GROUP BY f.TICKER
) AS Combined
WHERE Combined.Actual_Q3_2024 IS NOT NULL
ORDER BY Forecast_vs_Q3_Growth DESC
```

**Why This Pattern Works**:
- CASE expressions with MAX create pivot columns
- Direct use of VALUE (FA_Quarterly is float)
- Derived table (Combined) simplifies outer query
- Can calculate quarter-over-quarter growth cleanly
- Filters applied after pivoting

---

### Pattern 6: Sector/Category Aggregations

**Use Case**: Aggregate metrics by sector or category

**✅ WORKS WELL**:
```sql
SELECT
    s.Sector,
    COUNT(DISTINCT f.TICKER) AS Company_Count,
    AVG(f.VALUE) AS Avg_Sector_Forecast,
    SUM(f.VALUE) AS Total_Sector_Forecast,
    MIN(f.VALUE) AS Min_Forecast,
    MAX(f.VALUE) AS Max_Forecast,
    STDEV(f.VALUE) AS Sector_Volatility
FROM Forecast f
LEFT JOIN Sector_Map s ON f.TICKER = s.Ticker
WHERE f.KEYCODE LIKE '%.NPATMI'
    AND f.DATE = '2025'
    AND s.Sector IS NOT NULL
GROUP BY s.Sector
HAVING COUNT(DISTINCT f.TICKER) >= 5
ORDER BY Total_Sector_Forecast DESC
```

**Best Practices**:
- Group by category dimension (Sector, L1, Region)
- Use HAVING to filter categories with minimum counts
- Multiple aggregate functions work fine
- DISTINCT counts prevent double-counting joined records

---

## Performance Optimization Tips

### 1. Filter Early in WHERE Clause

```sql
-- ✅ GOOD - Filter before aggregation
SELECT TICKER, AVG(VALUE) AS Avg_Forecast
FROM Forecast
WHERE KEYCODE LIKE '%.NPATMI'
    AND DATE = '2025'
    AND VALUE > 0
GROUP BY TICKER

-- ❌ SLOWER - Filter after aggregation
SELECT TICKER, Avg_Forecast
FROM (
    SELECT TICKER, AVG(VALUE) AS Avg_Forecast
    FROM Forecast
    GROUP BY TICKER
) AS F
WHERE Avg_Forecast > 0
```

### 2. Use Indexed Columns in JOINs

**Indexed Columns (Fast)**:
- `TICKER` - Primary key in most tables
- `DATE`, `TRADE_DATE` - Time-series queries
- `KEYCODE` - Metric lookups
- `YEAR` - Year filtering

**Example**:
```sql
-- ✅ FAST - Uses indexed columns
LEFT JOIN FA_Quarterly fa
    ON f.TICKER = fa.TICKER
    AND fa.KEYCODE = 'NPATMI'
    AND fa.YEAR = 2024

-- ❌ SLOW - Non-indexed column
LEFT JOIN FA_Quarterly fa
    ON f.TICKER = fa.TICKER
    AND fa.YoY > 0.1
```

### 3. Limit Result Sets with TOP

```sql
-- ✅ GOOD - Limit results
SELECT TOP 100 TICKER, VALUE
FROM Forecast
ORDER BY VALUE DESC

-- ❌ AVOID - Fetching all rows
SELECT TICKER, VALUE
FROM Forecast
ORDER BY VALUE DESC
```

### 4. Use COUNT(DISTINCT) Wisely

```sql
-- ✅ GOOD - Necessary for broker count
COUNT(DISTINCT f.KEYCODE) AS Broker_Count

-- ⚠️ CAREFUL - Multiple COUNT DISTINCT can be slow
COUNT(DISTINCT f.KEYCODE) AS Broker_Count,
COUNT(DISTINCT f.ORGANCODE) AS Org_Count,
COUNT(DISTINCT fa.DATE) AS Period_Count
-- Use only when needed
```

---

## Common Failure Patterns (Avoid These)

### ❌ Pattern 1: Correlated Subqueries in SELECT

```sql
-- FAILS: Query execution error
SELECT
    f.TICKER,
    (SELECT AVG(TRY_CAST(VALUE AS FLOAT))
     FROM FA_Quarterly
     WHERE TICKER = f.TICKER AND KEYCODE = 'NPATMI') AS Avg_Actual
FROM Forecast f
```

**Error**: "Database query execution failed"

**Solution**: Use LEFT JOIN with GROUP BY (see Pattern 2 above)

---

### ❌ Pattern 2: Too Many JOINs (4+)

```sql
-- OFTEN FAILS: Too complex
SELECT f.TICKER, ...
FROM Forecast f
LEFT JOIN FA_Quarterly fa ON ...
LEFT JOIN Valuation v ON ...
LEFT JOIN MarketCap m ON ...
LEFT JOIN BankingMetrics b ON ...
```

**Error**: "Query execution failed" or timeout

**Solution**:
- Limit to 3 tables maximum
- Use derived tables to pre-aggregate before joining
- Split into multiple queries if needed

---

### ❌ Pattern 3: Grouping by Market_Data Columns

```sql
-- FAILS: Market_Data has multiple dates per ticker (daily data)
SELECT fq.TICKER, AVG(fq.VALUE), m.PE, m.PB
FROM FA_Quarterly fq
LEFT JOIN Market_Data m
    ON fq.TICKER = m.TICKER
    AND m.TRADE_DATE >= '2025-11-01'
WHERE fq.KEYCODE = 'NPATMI'
GROUP BY fq.TICKER, m.PE, m.PB  -- Creates separate groups for each date!
```

**Error**: "Database query execution failed"

**Why**: Market_Data has multiple rows per ticker (daily time series). Grouping by PE/PB/MKT_CAP creates a separate group for each date's values, exploding the result set.

**Solution 1 - Use MAX()** (Recommended):
```sql
SELECT fq.TICKER, AVG(fq.VALUE) AS Avg_NPATMI,
       MAX(m.PE) AS PE, MAX(m.PB) AS PB
FROM FA_Quarterly fq
LEFT JOIN Market_Data m ON fq.TICKER = m.TICKER AND m.TRADE_DATE >= '2025-11-01'
WHERE fq.KEYCODE = 'NPATMI'
GROUP BY fq.TICKER
HAVING MAX(m.PE) > 0
```

**Solution 2 - Derived Table**:
```sql
LEFT JOIN (
    SELECT TICKER, PE, PB, MKT_CAP
    FROM Market_Data
    WHERE TRADE_DATE = (SELECT MAX(TRADE_DATE) FROM Market_Data)
) m ON fq.TICKER = m.TICKER
GROUP BY fq.TICKER, m.PE, m.PB
```

---


## Query Complexity Guidelines

### ✅ Low Complexity (Always Works)
- Single table with GROUP BY
- 1-2 aggregate functions
- Simple WHERE filters
- Window functions on single table

### ✅ Medium Complexity (Works Reliably)
- 2-table JOIN with aggregations
- Window functions with PARTITION BY
- CASE expressions for pivoting
- CTEs (WITH clause) - now supported
- Derived tables (subqueries in FROM)
- 3-5 aggregate functions

### ⚠️ High Complexity (Use with Care)
- 3-table JOINs
- Multiple window functions
- Nested derived tables
- Complex CASE logic
- 6+ aggregate functions

### ❌ Very High Complexity (Often Fails)
- 4+ table JOINs
- Correlated subqueries in SELECT
- Recursive queries
- PIVOT/UNPIVOT operations

---

## Real-World Examples

### Example 1: Forecast vs Actual Analysis

**Goal**: Compare forecasts to actual results with growth metrics

```sql
SELECT TOP 10
    f.TICKER,
    AVG(f.VALUE) AS Avg_Forecast_2025,
    AVG(fa.VALUE) AS Avg_Actual_2024,
    STDEV(f.VALUE) AS Forecast_Spread,
    (AVG(f.VALUE) - AVG(fa.VALUE)) / NULLIF(AVG(fa.VALUE), 0) * 100 AS Expected_Growth_Pct
FROM Forecast f
LEFT JOIN FA_Quarterly fa
    ON f.TICKER = fa.TICKER
    AND fa.KEYCODE = 'NPATMI'
    AND fa.YEAR = 2024
WHERE f.KEYCODE = 'NPATMI' AND f.DATE = '2025'
GROUP BY f.TICKER
HAVING AVG(fa.VALUE) IS NOT NULL
ORDER BY Expected_Growth_Pct DESC
```

---

### Example 2: Sector Forecast Distribution

**Goal**: Analyze forecast spread and coverage by sector

```sql
SELECT
    s.Sector,
    COUNT(DISTINCT f.TICKER) AS Company_Count,
    AVG(f.VALUE) AS Avg_Sector_Forecast,
    MIN(f.VALUE) AS Min_Forecast,
    MAX(f.VALUE) AS Max_Forecast,
    STDEV(f.VALUE) AS Sector_Volatility,
    (MAX(f.VALUE) - MIN(f.VALUE)) / NULLIF(AVG(f.VALUE), 0) * 100 AS Range_Pct
FROM Forecast f
LEFT JOIN Sector_Map s ON f.TICKER = s.Ticker
WHERE f.KEYCODE LIKE '%.NPATMI'
    AND f.DATE = '2025'
    AND s.Sector IS NOT NULL
GROUP BY s.Sector
HAVING COUNT(DISTINCT f.TICKER) >= 5
ORDER BY Avg_Sector_Forecast DESC
```

---

### Example 3: Quarterly Momentum with Forecasts

**Goal**: Compare Q-o-Q growth trends to forward forecasts

```sql
SELECT TOP 10
    Combined.TICKER,
    Combined.Avg_Forecast_2025,
    Combined.Q3_2024,
    Combined.Q2_2024,
    (Combined.Q3_2024 - Combined.Q2_2024) / NULLIF(Combined.Q2_2024, 0) * 100 AS QoQ_Growth,
    (Combined.Avg_Forecast_2025 - Combined.Q3_2024) / NULLIF(Combined.Q3_2024, 0) * 100 AS Forecast_Growth
FROM (
    SELECT
        f.TICKER,
        AVG(f.VALUE) AS Avg_Forecast_2025,
        MAX(CASE WHEN fa.DATE = '2024Q3' THEN fa.VALUE END) AS Q3_2024,
        MAX(CASE WHEN fa.DATE = '2024Q2' THEN fa.VALUE END) AS Q2_2024
    FROM Forecast f
    LEFT JOIN FA_Quarterly fa
        ON f.TICKER = fa.TICKER
        AND fa.KEYCODE = 'NPATMI'
        AND fa.YEAR = 2024
    WHERE f.KEYCODE = 'NPATMI' AND f.DATE = '2025'
    GROUP BY f.TICKER
) AS Combined
WHERE Combined.Q3_2024 IS NOT NULL AND Combined.Q2_2024 IS NOT NULL
ORDER BY Forecast_Growth DESC
```

---

## Summary Checklist

Before running a complex query, verify:

- [ ] Query starts with SELECT or WITH (CTEs now supported)
- [ ] All division uses NULLIF(..., 0)
- [ ] Using 3 or fewer table JOINs
- [ ] No correlated subqueries in SELECT clause
- [ ] Filtering on indexed columns (TICKER, DATE, KEYCODE, YEAR)
- [ ] Using TOP N to limit results
- [ ] GROUP BY includes all non-aggregated SELECT columns
- [ ] HAVING clause for filtering aggregated results (not WHERE)
- [ ] Not grouping by Market_Data columns (use MAX() instead)

**If query fails**:
1. Remove tables from JOINs one by one
2. Simplify aggregations
3. Check for correlated subqueries
4. Verify NULLIF in all division operations
5. Check if grouping by Market_Data columns (use derived table or MAX())

---

**Last Tested**: 2025-11-25
**Test Dataset**: Forecast, FA_Quarterly, Sector_Map, Market_Data tables
**Success Rate**: 90% for 2-3 table queries, 98% for single/two-table queries (CTEs now supported)
