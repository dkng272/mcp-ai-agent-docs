# Query Best Practices

**Last Updated**: 2025-12-17
**Purpose**: Proven patterns for querying via MCP tools (SQL and Python)

---

## Tool Selection Guide

| Scenario | Tool | Why |
|----------|------|-----|
| Simple lookup (<100 rows) | `mssql_read_data` | Direct, fast |
| Single SQL aggregation | `mssql_read_data` | SQL aggregates are efficient |
| Complex multi-step analysis | `execute_python` | pandas processing power |
| Large datasets (1000+ rows) | `execute_python` | 99% context reduction |
| Rolling/window calculations | `execute_python` | pandas rolling() functions |
| Multiple related analyses | `execute_python` | Process all in one call |
| Don't know which table | `find_relevant_tables` | Semantic search for tables |
| Forum sentiment/thesis | `f319_*` tools | Pre-analyzed F319 data |
| Zalo signals/alerts | `zalo_*` tools | Real-time group signals |
| All-source sentiment | `sentiment_dashboard_get` | Aggregates all sources |

---

# Part 1: SQL Query Patterns

## Key Findings
- ✅ Window functions work excellently (use liberally)
- ✅ CTEs (WITH clause) fully supported
- ✅ Derived tables (subqueries in FROM) work as alternative
- ✅ 2-3 table JOINs work reliably
- ❌ Correlated subqueries in SELECT clause fail
- ❌ 4+ table JOINs become unreliable

---

## Critical SQL Rules

### 1. All Tables Are Float - No TRY_CAST Needed

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

### 3. CTEs (WITH Clause) Supported

```sql
-- ✅ WORKS
WITH ForecastAvg AS (
    SELECT TICKER, AVG(VALUE) AS Avg_Forecast
    FROM Forecast
    GROUP BY TICKER
)
SELECT * FROM ForecastAvg
```

### 4. NO Correlated Subqueries in SELECT

```sql
-- ❌ FAILS
SELECT f.TICKER,
    (SELECT AVG(VALUE) FROM FA_Quarterly WHERE TICKER = f.TICKER) AS Avg
FROM Forecast f

-- ✅ WORKS - Use JOIN instead
SELECT f.TICKER, AVG(fa.VALUE) AS Avg
FROM Forecast f
LEFT JOIN FA_Quarterly fa ON f.TICKER = fa.TICKER
GROUP BY f.TICKER
```

---

## SQL Query Patterns

### Pattern 1: Simple Aggregation

```sql
SELECT
    f.TICKER,
    COUNT(DISTINCT f.KEYCODE) AS Broker_Count,
    AVG(f.VALUE) AS Avg_Forecast,
    (MAX(f.VALUE) - MIN(f.VALUE)) / NULLIF(AVG(f.VALUE), 0) * 100 AS Spread_Pct
FROM Forecast f
WHERE f.KEYCODE = 'NPATMI' AND f.DATE = '2025'
GROUP BY f.TICKER
HAVING COUNT(DISTINCT f.KEYCODE) >= 3
ORDER BY Avg_Forecast DESC
```

### Pattern 2: Two-Table JOIN

```sql
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
ORDER BY Growth_Pct DESC
```

### Pattern 3: Window Functions

```sql
SELECT TOP 10
    f.TICKER,
    f.VALUE,
    ROW_NUMBER() OVER (PARTITION BY f.TICKER ORDER BY f.VALUE DESC) AS Rank,
    AVG(f.VALUE) OVER (PARTITION BY f.TICKER) AS Avg_By_Ticker
FROM Forecast f
WHERE f.KEYCODE LIKE '%.NPATMI'
ORDER BY f.TICKER, Rank
```

### Pattern 4: CASE for Pivoting

```sql
SELECT
    f.TICKER,
    MAX(CASE WHEN fa.DATE = '2024Q3' THEN fa.VALUE END) AS Q3_2024,
    MAX(CASE WHEN fa.DATE = '2024Q2' THEN fa.VALUE END) AS Q2_2024
FROM Forecast f
LEFT JOIN FA_Quarterly fa ON f.TICKER = fa.TICKER AND fa.KEYCODE = 'NPATMI'
WHERE f.KEYCODE = 'NPATMI'
GROUP BY f.TICKER
```

---

## SQL Failure Patterns (Avoid)

### ❌ Too Many JOINs (4+)
Limit to 3 tables. Use derived tables to pre-aggregate.

### ❌ Grouping by Market_Data Columns
Market_Data has daily rows. Use `MAX()` instead of GROUP BY:
```sql
-- ✅ CORRECT
SELECT fq.TICKER, MAX(m.PE) AS PE
FROM FA_Quarterly fq
LEFT JOIN Market_Data m ON fq.TICKER = m.TICKER
GROUP BY fq.TICKER
```

---

## Query Complexity Guidelines

| Complexity | Works? | Examples |
|------------|--------|----------|
| Low | ✅ Always | Single table, 1-2 aggregates, window functions |
| Medium | ✅ Reliable | 2-table JOIN, CTEs, CASE pivoting |
| High | ⚠️ Care | 3-table JOINs, multiple window functions |
| Very High | ❌ Often fails | 4+ JOINs, correlated subqueries |

---

# Part 2: Python Executor Patterns

## Key Findings
- ✅ SQL JOINs in query() work excellently
- ✅ pandas aggregations (groupby, rolling, pivot) work perfectly
- ✅ Mixed SQL + MongoDB in same call works
- ✅ Large datasets (160k+ rows) handled efficiently
- ✅ `save_csv()` and `save_figure()` work reliably
- ❌ Dynamic SQL with f-strings fails (use JOINs instead)
- ❌ Returning raw DataFrames wastes context
- ⚠️ **CRITICAL BUG**: DataFrame column/row access breaks subsequent queries (see below)

---

## ⚠️ CRITICAL BUG: Query Execution Order

**Problem**: Accessing DataFrame columns or rows BREAKS subsequent `sql_query()` calls. The query silently returns empty results with no error.

```python
# ❌ BROKEN - second query silently fails
df1 = sql_query("SELECT Ticker FROM Sector_Map WHERE L2 = 'Banking'")
ticker_list = df1['Ticker'].tolist()  # <-- Accessing column breaks subsequent queries
df2 = sql_query("SELECT * FROM Market_Data")  # <-- Returns empty! No error raised

# These operations ALL break subsequent queries:
# - df['column']           (column access)
# - df['column'].tolist()  (to list)
# - df['column'].values    (to numpy)
# - df['column'].unique()  (unique values)
# - df.iloc[0]             (row access)

# BUT these are SAFE:
# - len(df)                (row count)
# - df.columns             (column names)
```

### Workaround 1: ALL Queries First, THEN Manipulate (Recommended)

```python
# ✅ WORKS - Execute ALL queries upfront before any DataFrame manipulation
df_sectors = sql_query("SELECT Ticker FROM Sector_Map WHERE L2 = 'Banking'")
df_prices = sql_query("SELECT TICKER, PX_LAST FROM Market_Data WHERE TRADE_DATE = '2025-01-13'")
df_banking = sql_query("SELECT TICKER, ROE FROM BankingMetrics WHERE YEARREPORT=2024")

# NOW safe to manipulate DataFrames
ticker_list = df_sectors['Ticker'].tolist()
filtered = df_prices[df_prices['TICKER'].isin(ticker_list)]
merged = filtered.merge(df_banking, on='TICKER')
result = {"data": merged.to_dict('records')}
```

### Workaround 2: Use SQL JOINs (BEST - Avoids Problem Entirely)

```python
# ✅ BEST - Single query with JOINs, no dynamic SQL needed
df = sql_query("""
    SELECT s.Ticker, m.PX_LAST, b.ROE
    FROM Sector_Map s
    LEFT JOIN Market_Data m ON s.Ticker = m.TICKER AND m.TRADE_DATE = '2025-01-13'
    LEFT JOIN BankingMetrics b ON s.Ticker = b.TICKER AND b.YEARREPORT = 2024
    WHERE s.L2 = 'Banking'
""")
# Manipulate freely after single query completes
result = df.nlargest(5, 'ROE').to_dict('records')
```

---

## Critical Python Rules

### 1. NO Import Statements

All packages pre-imported: `pd`, `np`, `scipy`, `stats`, `sklearn`, `plt`, `matplotlib`, `sql_query()`

**Export functions (for heavy data + complex charts):**
- `save_csv(df, filename, downloadable=True)` → download URL (30 min expiry)
- `save_figure(fig, filename)` → download URL for charts

Query + chart + export in one call — much faster than computer tool workflows.

```python
# ✅ CORRECT
df = query("SELECT * FROM Sales")
result = df.groupby('region')['amount'].sum().to_dict()

# ❌ WRONG
import pandas as pd  # Don't do this
```

### 2. Always Assign to 'result'

```python
# ✅ CORRECT
df = query("SELECT * FROM Sales")
result = {"total": df['Revenue'].sum()}

# ❌ WRONG - Nothing returned
summary = df['Revenue'].sum()  # Forgot 'result'
```

### 3. Use SQL JOINs, NOT f-strings

```python
# ✅ CORRECT - SQL JOIN
df = query("""
    SELECT p.TICKER, p.PX_LAST, s.Sector
    FROM Market_Data p
    JOIN Sector_Map s ON p.TICKER = s.Ticker
    WHERE s.L2 = 'Brokerage'
""")

# ❌ WRONG - Dynamic SQL fails
tickers = query("SELECT Ticker FROM Sector_Map WHERE L2 = 'Banks'")
prices = query(f"SELECT * FROM Market_Data WHERE TICKER IN ('{tickers}')")
```

### 4. Result Must Be JSON-Serializable

```python
# ✅ CORRECT
result = df.to_dict('records')  # List of dicts

# ❌ WRONG
result = df  # DataFrame not serializable
```

### 5. Aggregate Before Returning

```python
# ✅ GOOD - Small summary
result = {
    "total": df['Revenue'].sum(),
    "by_region": df.groupby('Region')['Revenue'].sum().to_dict()
}

# ❌ BAD - Returns all data
result = df.to_dict('records')  # 45,000 records!
```

---

## Python Patterns

### Pattern 1: JOIN with Aggregation

```python
df = query("""
    SELECT f.TICKER, f.VALUE as Forecast, fa.VALUE as Actual, s.Sector
    FROM Forecast f
    JOIN FA_Quarterly fa ON f.TICKER = fa.TICKER AND fa.KEYCODE = 'NPATMI'
    JOIN Sector_Map s ON f.TICKER = s.Ticker
    WHERE f.KEYCODE = 'NPATMI' AND f.DATE = '2025'
""")

df['Growth'] = (df['Forecast'] - df['Actual']) / df['Actual'] * 100
result = {
    "by_sector": df.groupby('Sector')['Growth'].mean().to_dict(),
    "top_growth": df.nlargest(10, 'Growth')[['TICKER', 'Growth']].to_dict('records')
}
```

### Pattern 2: Rolling Calculations

```python
df = query("""
    SELECT p.TICKER, p.TRADE_DATE, p.PX_LAST
    FROM Market_Data p
    JOIN Sector_Map s ON p.TICKER = s.Ticker
    WHERE s.L2 = 'Brokerage'
    ORDER BY p.TICKER, p.TRADE_DATE
""")

results = []
for ticker in df['TICKER'].unique():
    stock = df[df['TICKER'] == ticker].copy()
    stock['High_50D'] = stock['PX_LAST'].rolling(50).max()
    latest = stock.iloc[-1]
    drawdown = (latest['PX_LAST'] / latest['High_50D'] - 1) * 100
    results.append({'ticker': ticker, 'drawdown_pct': round(drawdown, 2)})

result = {'drawdowns': sorted(results, key=lambda x: x['drawdown_pct'])}
```

### Pattern 3: Pivot Tables

```python
df = query("""
    SELECT TICKER, DATE, VALUE
    FROM FA_Quarterly
    WHERE KEYCODE = 'NPATMI' AND YEAR = 2024
""")

pivot = df.pivot_table(values='VALUE', index='TICKER', columns='DATE', aggfunc='first')
pivot['QoQ'] = (pivot['2024Q3'] - pivot['2024Q2']) / pivot['2024Q2'] * 100
result = {'data': pivot.reset_index().to_dict('records')}
```

---

## Units Reference (CRITICAL)

Different tables use different units - **always convert before calculations!**

| Source | Field | Unit | Example |
|--------|-------|------|---------|
| `Market_Data` | MKT_CAP | **Millions VND** | 513,637,487 = 513.6T |
| `Market_Data` | PX_LAST | VND/share | 61,026 |
| `FA_Quarterly` | VALUE | VND | 8.57T (quarterly) |
| `BankingMetrics` | NPATMI, Total Assets | VND | 8.57T |
| `BankingMetrics` | ROE, NIM, NPL, CIR | Decimal | 0.177 = 17.7% |
| `Forecast` | NPATMI | VND (**annual**) | 35.6T |
| `Forecast` | Target_Price | VND/share | 69,300 |
| `RealEstateProjects` (MongoDB) | npv_to_company | **Billions VND** | 33,284 = 33.3T |

### Common Unit Conversion Errors

```python
# ❌ WRONG: P/E with mismatched units
pe = mkt_cap / npatmi  # MKT_CAP in millions, NPATMI in VND!

# ✅ CORRECT: Convert MKT_CAP to VND
pe = (mkt_cap * 1e6) / npatmi

# ❌ WRONG: NAV premium with different units
nav_premium = npv_to_company / mkt_cap  # Billions vs Millions!

# ✅ CORRECT: Both to same unit (VND)
nav_premium = (npv_to_company * 1e9) / (mkt_cap * 1e6)
# Or simplified: npv_to_company * 1000 / mkt_cap

# ❌ WRONG: Comparing annual forecast to quarterly actual
growth = (forecast_npatmi - quarterly_npatmi) / quarterly_npatmi

# ✅ CORRECT: Annualize or use consistent periods
annual_actual = quarterly_npatmi * 4  # rough annualization
growth = (forecast_npatmi - annual_actual) / annual_actual
```

### Unit Conversion Cheat Sheet

```python
# To convert everything to VND:
mkt_cap_vnd = mkt_cap * 1e6           # Market_Data.MKT_CAP
npv_vnd = npv_to_company * 1e9        # RealEstateProjects
fa_value_vnd = value                   # FA_Quarterly (already VND)
banking_vnd = npatmi                   # BankingMetrics (already VND)

# To convert ratios to percentages:
roe_pct = roe * 100                    # 0.177 → 17.7%
nim_pct = nim * 100                    # 0.028 → 2.8%
```

---

## Error Handling

### KeyError - Column Not Found
Check `available_columns` in error response. SQL Server preserves column casing.

### Empty Results
Check date format (`'2024-01-01'` not `'01/01/2024'`) and filter conditions.

---

## Summary Checklist

**SQL Queries:**
- [ ] All division uses NULLIF(..., 0)
- [ ] 3 or fewer table JOINs
- [ ] No correlated subqueries in SELECT
- [ ] Using TOP N to limit results
- [ ] Not grouping by Market_Data columns

**Python Executor:**
- [ ] NO import statements
- [ ] Using SQL JOINs (not f-strings)
- [ ] **ALL queries executed BEFORE any DataFrame manipulation** (critical!)
- [ ] Assigned to `result` variable
- [ ] Result is JSON-serializable
- [ ] Aggregated data (not raw DataFrames)
- [ ] Complex charts: use `save_figure(fig, filename)` for download URL
- [ ] Large CSV exports: use `save_csv(df, filename, downloadable=True)`

---

**Last Tested**: 2026-02-03
**Success Rate**: 90%+ for SQL, 95%+ for Python when following patterns
**Known Bug**: DataFrame column/row access breaks subsequent sql_query() calls - use workarounds above
