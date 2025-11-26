# Execute Python Best Practices

**Last Updated**: 2025-11-26
**Purpose**: Patterns for using the `execute_python` MCP tool for server-side data processing

---

## Executive Summary

The `execute_python` tool allows AI agents to write Python code that processes SQL data server-side, returning only summarized results. This dramatically reduces context usage (99%+ reduction for large datasets).

**Key Findings**:
- ✅ SQL JOINs in query() work excellently
- ✅ pandas aggregations (groupby, rolling, pivot) work perfectly
- ✅ Multiple independent queries work fine
- ❌ Dynamic SQL with f-strings fails (use JOINs instead)
- ❌ Returning raw DataFrames wastes context

---

## Tool Selection Guide

| Scenario | Tool | Why |
|----------|------|-----|
| Simple lookup (<100 rows) | `read_data` | Direct, fast, no Python overhead |
| Single aggregation | `read_data` with SQL | SQL aggregates are efficient |
| Complex multi-step analysis | `execute_python` | pandas processing power |
| Large datasets (1000+ rows) | `execute_python` | 99% context reduction |
| Rolling calculations | `execute_python` | pandas rolling() functions |
| Multiple related queries | `execute_python` | Process all in one call |

---

## How It Works

### Agent Writes Full Python Code

The AI agent writes complete Python code - not just JSON output:

```python
# Agent writes this entire code block
df = query("SELECT Region, Revenue FROM Sales WHERE Year = 2024")

# Agent writes the processing logic
total = df['Revenue'].sum()
by_region = df.groupby('Region')['Revenue'].sum()

# Agent must assign to 'result' - only this gets returned
result = {
    "total_revenue": total,
    "by_region": by_region.to_dict()
}
```

### Execution Flow

```
┌─────────────────────────────────────────────────────────┐
│ 1. Agent writes full Python code                        │
│    (SQL queries + pandas processing + result)           │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ 2. Tool executes in sandboxed Python environment        │
│    - query() fetches SQL data → DataFrame               │
│    - pandas/numpy/sklearn process data                  │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ 3. Only 'result' variable returned to agent             │
│    (must be JSON-serializable)                          │
└─────────────────────────────────────────────────────────┘
```

---

## Critical Rules (Always Follow)

### 1. Use SQL JOINs for Related Data

**✅ CORRECT - Use SQL JOIN**:
```python
# Generic example
df = query("""
    SELECT o.order_id, o.amount, c.name, c.region
    FROM Orders o
    JOIN Customers c ON o.customer_id = c.id
    WHERE o.year = 2024
""")
result = df.groupby('region')['amount'].sum().to_dict()
```

```python
# Real-world example with your tables
df = query("""
    SELECT p.TICKER, p.TRADE_DATE, p.PX_LAST, s.L2 as Sector, s.L3 as SubSector
    FROM Market_Data p
    JOIN Sector_Map s ON p.TICKER = s.Ticker
    WHERE s.L2 = 'Brokerage'
    ORDER BY p.TICKER, p.TRADE_DATE
""")
```

**❌ WRONG - Dynamic SQL with f-strings fails**:
```python
# This FAILS because dry run returns empty DataFrame
tickers_df = query("SELECT Ticker FROM Sector_Map WHERE L2 = 'Banks'")
ticker_list = "','".join(tickers_df['Ticker'].tolist())  # Empty in dry run!
prices = query(f"SELECT * FROM Market_Data WHERE TICKER IN ('{ticker_list}')")
```

**Why**: The tool uses two-phase execution. In phase 1 (dry run), queries return empty DataFrames to collect SQL statements. F-strings built from empty results produce invalid SQL.

---

### 2. Always Assign to `result` Variable

```python
# ✅ CORRECT
df = query("SELECT * FROM Sales")
result = {"total": df['Revenue'].sum()}

# ❌ WRONG - Nothing returned
df = query("SELECT * FROM Sales")
total = df['Revenue'].sum()
# Forgot to assign result!
```

---

### 3. Result Must Be JSON-Serializable

```python
# ✅ CORRECT - dict, list, numbers, strings
result = {
    "total": 12345.67,
    "items": ["A", "B", "C"],
    "breakdown": {"North": 100, "South": 200}
}

# ❌ WRONG - DataFrame is not JSON-serializable
result = df  # Error!

# ✅ CORRECT - Convert DataFrame to dict
result = df.to_dict('records')  # List of dicts
result = df.to_dict()           # Dict of columns
```

---

### 4. Aggregate Before Returning

```python
# ✅ GOOD - Returns small summary (247 bytes)
df = query("SELECT * FROM Sales WHERE Year = 2024")  # 45,000 rows
result = {
    "total": df['Revenue'].sum(),
    "by_region": df.groupby('Region')['Revenue'].sum().to_dict(),
    "count": len(df)
}

# ❌ BAD - Returns all data (2MB+)
df = query("SELECT * FROM Sales WHERE Year = 2024")
result = df.to_dict('records')  # 45,000 records!
```

---

## Query Patterns

### Pattern 1: Single Table with Aggregations

**Use Case**: Calculate statistics from one table

```python
# Generic example
df = query("SELECT category, price, quantity FROM Products WHERE active = 1")
result = {
    "total_products": len(df),
    "avg_price": df['price'].mean(),
    "by_category": df.groupby('category')['price'].mean().to_dict()
}
```

```python
# Real-world example
df = query("""
    SELECT TICKER, VALUE, YoY
    FROM FA_Quarterly
    WHERE KEYCODE = 'NPATMI' AND YEAR = 2024
""")
result = {
    "companies": len(df),
    "avg_profit": df['VALUE'].mean(),
    "top_growth": df.nlargest(5, 'YoY')[['TICKER', 'YoY']].to_dict('records')
}
```

---

### Pattern 2: JOIN with Processing

**Use Case**: Combine tables and analyze

```python
# Generic example
df = query("""
    SELECT o.amount, o.date, c.region, c.segment
    FROM Orders o
    JOIN Customers c ON o.customer_id = c.id
    WHERE o.date >= '2024-01-01'
""")
result = {
    "by_region": df.groupby('region')['amount'].sum().to_dict(),
    "by_segment": df.groupby('segment')['amount'].mean().to_dict()
}
```

```python
# Real-world example - Sector analysis
df = query("""
    SELECT f.TICKER, f.VALUE as Forecast, fa.VALUE as Actual, s.Sector
    FROM Forecast f
    JOIN FA_Quarterly fa ON f.TICKER = fa.TICKER AND fa.KEYCODE = 'NPATMI' AND fa.YEAR = 2024
    JOIN Sector_Map s ON f.TICKER = s.Ticker
    WHERE f.KEYCODE = 'NPATMI' AND f.DATE = '2025'
""")

df['Growth'] = (df['Forecast'] - df['Actual']) / df['Actual'] * 100
result = {
    "by_sector": df.groupby('Sector')['Growth'].mean().to_dict(),
    "top_growth": df.nlargest(10, 'Growth')[['TICKER', 'Sector', 'Growth']].to_dict('records')
}
```

---

### Pattern 3: Rolling Calculations

**Use Case**: Time-series analysis with moving averages, drawdowns

```python
# Generic example
df = query("SELECT date, ticker, price FROM Prices WHERE date >= '2024-01-01' ORDER BY ticker, date")

results = []
for ticker in df['ticker'].unique():
    stock = df[df['ticker'] == ticker].copy()
    stock['MA20'] = stock['price'].rolling(20).mean()
    stock['High20'] = stock['price'].rolling(20).max()
    latest = stock.iloc[-1]
    results.append({
        'ticker': ticker,
        'price': latest['price'],
        'ma20': latest['MA20'],
        'from_high': (latest['price'] / latest['High20'] - 1) * 100
    })

result = {'stocks': results}
```

```python
# Real-world example - Brokerage drawdown from 50D high
df = query("""
    SELECT p.TICKER, p.TRADE_DATE, p.PX_LAST, s.L3 as Tier
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
    results.append({
        'ticker': ticker,
        'tier': latest['Tier'],
        'price': latest['PX_LAST'],
        'high_50d': latest['High_50D'],
        'drawdown_pct': round(drawdown, 2)
    })

result = {'drawdowns': sorted(results, key=lambda x: x['drawdown_pct'])}
```

---

### Pattern 4: Pivot Tables

**Use Case**: Reshape data for comparison

```python
# Generic example
df = query("SELECT quarter, region, revenue FROM Sales WHERE year = 2024")
pivot = df.pivot_table(values='revenue', index='region', columns='quarter', aggfunc='sum')
result = pivot.to_dict()
```

```python
# Real-world example - Quarterly profit comparison
df = query("""
    SELECT TICKER, DATE, VALUE
    FROM FA_Quarterly
    WHERE KEYCODE = 'NPATMI' AND YEAR = 2024
""")

pivot = df.pivot_table(values='VALUE', index='TICKER', columns='DATE', aggfunc='first')
pivot['QoQ_Growth'] = (pivot['2024Q3'] - pivot['2024Q2']) / pivot['2024Q2'] * 100
result = {
    'quarterly_data': pivot.reset_index().to_dict('records'),
    'avg_qoq_growth': pivot['QoQ_Growth'].mean()
}
```

---

## Error Handling

### KeyError - Column Not Found

The tool provides `available_columns` when KeyError occurs:

```json
{
  "error": "PYTHON_ERROR",
  "message": "KeyError: 'TICKER'",
  "available_columns": {"df": ["Ticker", "L2", "L3"]},
  "hint": "Column not found - check column name casing"
}
```

**Common cause**: SQL Server returns exact column casing from schema. Use `df.columns.tolist()` to check.

**Fix**:
```python
# Check the actual column names
print(df.columns.tolist())  # Shows in stdout

# Use correct casing
tickers = df['Ticker'].tolist()  # Not 'TICKER'
```

---

### SQL Error - Invalid Table/Column

```json
{
  "error": "SQL_EXECUTION_FAILED",
  "message": "Invalid object name 'NonExistent_Table'",
  "hint": "Table may not exist or check schema prefix (e.g., dbo.TableName)"
}
```

---

### Empty Results Warning

```json
{
  "success": true,
  "result": {...},
  "warning": "Query 1 returned 0 rows - check filter conditions"
}
```

**Common causes**:
- Wrong date format (use `'2024-01-01'` not `'01/01/2024'`)
- Case-sensitive string comparison
- Filter too restrictive

---

## Available Packages

| Package | Import As | Common Uses |
|---------|-----------|-------------|
| pandas | `pd` | DataFrames, groupby, merge, pivot_table |
| numpy | `np` | Numerical operations, statistics |
| scipy | `scipy`, `stats` | Statistical tests, distributions |
| scikit-learn | `sklearn` | Preprocessing, clustering |

### sklearn Submodules

```python
sklearn['preprocessing']   # StandardScaler, MinMaxScaler
sklearn['cluster']         # KMeans, DBSCAN
sklearn['decomposition']   # PCA
sklearn['metrics']         # silhouette_score
```

---

## Visualization

For charts and visualizations:
1. Use execute_python to aggregate/summarize large datasets
2. Return the small summary to Claude
3. Create visualizations locally with the aggregated data

This keeps execute_python focused on server-side data processing.

---

## Summary Checklist

Before submitting execute_python code:

- [ ] Using SQL JOINs for related tables (NOT f-strings)
- [ ] Assigned final output to `result` variable
- [ ] Result is JSON-serializable (dict, list, numbers, strings)
- [ ] Aggregated data before returning (not raw DataFrames)
- [ ] Used correct column casing (check with `df.columns.tolist()`)
- [ ] Division uses proper null handling (`/ df['col'].replace(0, np.nan)`)

**If query fails**:
1. Check `available_columns` in error response
2. Verify column name casing
3. Check if DataFrame is empty (wrong filters)
4. Ensure SQL JOINs are correct

---

**Last Tested**: 2025-11-26
**Success Rate**: 95%+ when following JOIN patterns
