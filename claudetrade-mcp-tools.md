# ClaudeTrade MCP Server - Tools Documentation

Remote MCP server at `https://claudetrade.live/mcp-sse/sse` (HTTP transport)

## Overview

The ClaudeTrade MCP provides access to:
- **SQL Server** (Azure): Vietnamese stock market data, financials, sector data
- **MongoDB**: Company models, real estate projects, agent configurations
- **PostgreSQL/TimescaleDB**: Real-time CVD (Cumulative Volume Delta) data
- **Supabase**: Broker consensus and research reports

---

## 0. Table Discovery

### `find_relevant_tables`
Semantic search to find relevant database tables for your question.

```python
question: "Which table has banking sector loan data?"
database: "all"  # or "AzureSQL", "MongoDB", "PostgreSQL"
domain: "Banking"  # optional filter
top_k: 5  # number of results
```

**Use this FIRST** when you don't know which table to query. Returns:
- Table name and database
- Description and key columns
- Example queries and gotchas
- Similarity score

---

## 1. SQL Server Tools

### `mssql_list_table`
List all tables in the database.

```python
# No parameters required
```

### `mssql_describe_table`
Get schema (columns and types) for a table.

```python
tableName: "Market_Data"
```

### `mssql_read_data`
Execute SELECT query and return results.

```python
query: "SELECT TOP 10 * FROM Market_Data WHERE TICKER = 'VNM'"
```

### `mssql_export_data`
Export query results directly to CSV file on server.

```python
query: "SELECT * FROM Market_Data WHERE TRADE_DATE >= '2025-01-01'"
outputPath: "/root/exports/market_data.csv"
includeHeaders: true
```

### Available SQL Tables

| Table | Description |
|-------|-------------|
| `Market_Data` | Daily OHLCV price data for all tickers |
| `MarketIndex` | Index data (VNINDEX, VN30, etc.) |
| `FA_Quarterly` | Quarterly financial statements |
| `FA_Annual` | Annual financial statements |
| `Sector_Map` | Ticker to sector mapping (L1, L2, L3 tiers) |
| `Ticker_Reference` | Ticker metadata and company info |
| `Forecast` | Analyst earnings forecasts |
| `Forecast_Consensus` | Consensus estimates |
| `BankingMetrics` | Bank-specific KPIs |
| `BrokerageMetrics` | Brokerage company KPIs |
| `Power_Company_Operations` | Power sector operational data |
| `Steel_data` | Steel industry data |
| `Commodity` | Commodity prices |
| `Vietnam_Credit_Deposit` | Banking system credit/deposit data |
| `ceic_macro_data` | Macroeconomic indicators |

---

## 2. MongoDB Tools

### `mongo_list_collections`
List all collections in the database.

```python
database: "IRIS"  # optional, defaults to IRIS
list_databases: false
```

### `mongo_describe_collection`
Analyze collection schema by sampling documents.

```python
collection: "RealEstateProjects"
sample_size: 10
```

### `mongo_find`
Query documents from a collection.

```python
collection: "RealEstateProjects"
filter: {"ticker": "VHM"}
projection: {"project_name": 1, "irr": 1}
sort: {"irr": -1}
limit: 100
```

### `mongo_aggregate`
Run aggregation pipeline.

```python
collection: "RealEstateProjects"
pipeline: [
    {"$match": {"ticker": "DXG"}},
    {"$group": {"_id": "$status", "total_npv": {"$sum": "$npv_to_company"}}}
]
```

### `mongo_insert`
Insert documents into a collection.

```python
collection: "CompanyModels"
document: {"ticker": "ABC", "year": 2025, "data": {...}}
# or
documents: [{...}, {...}]  # for multiple
```

### `mongo_update`
Update documents in a collection.

```python
collection: "CompanyModels"
filter: {"ticker": "ABC"}
update: {"$set": {"status": "active"}}
upsert: false
operation: "updateOne"  # or "updateMany", "replaceOne"
```

### `mongo_delete`
Delete documents from a collection.

```python
collection: "CompanyModels"
filter: {"status": "inactive"}
delete_many: false
```

### `mongo_vector_search`
Semantic search over macro research articles using vector embeddings.

```python
query: "inflation outlook"           # Natural language search (required)
limit: 5                             # Max results (default: 5, max: 20)
source: "gavekal"                    # Filter: "gavekal", "goldman_sachs", or array of both
region: "asia"                       # Filter: "us", "asia", "europe", "global", or array
country: "china"                     # Filter: "us", "china", "japan", "india", "uk", "vietnam", or array
days: 7                              # Get articles from last N days
date_from: "2025-01-01"              # Start date (YYYY-MM-DD)
date_to: "2025-12-31"                # End date (YYYY-MM-DD)
deduplicate: true                    # Return unique articles only (default: true)
min_score: 0.5                       # Minimum similarity score 0-1 (default: 0.5)
```

**Sources:**
- Gavekal Research (~1100 chunks) - Independent macro research
- Goldman Sachs (~800 chunks) - Investment bank research (has series_name field)

**Regions/Countries:**
- Regions: `us`, `asia`, `europe`, `global`
- Countries: `us`, `china`, `japan`, `india`, `uk`, `vietnam`

**Examples:**
```python
# Recent updates on inflation
{"query": "inflation outlook", "days": 7}

# Asia-focused monetary policy
{"query": "monetary policy", "region": "asia"}

# China property market
{"query": "property market", "country": "china"}

# US + Europe interest rates
{"query": "interest rates", "region": ["us", "europe"]}

# Last month's Fed coverage from Gavekal
{"query": "Fed rate cuts", "days": 30, "source": "gavekal"}

# Date range search
{"query": "China economy", "date_from": "2025-10-01", "date_to": "2025-12-31"}
```

**Response:**
```json
{
  "success": true,
  "query": "inflation outlook",
  "filters": { "source": "all", "region": "all", "country": "all", "date_from": "2025-12-16", "date_to": "2025-12-23", "days": 7 },
  "deduplicated": true,
  "limit": 5,
  "min_score": 0.5,
  "result_count": 5,
  "results": [
    {
      "title": "Vietnam's Inflation Pressures",
      "source": "gavekal",
      "series_name": null,
      "date_iso": "2025-12-20",
      "region": "asia",
      "country": "vietnam",
      "content": "Article chunk text...",
      "summary": "Brief summary...",
      "chunk_index": 0,
      "total_chunks": 3,
      "score": 0.847
    }
  ]
}
```

| Response Field | Description |
|----------------|-------------|
| `score` | Similarity to query (0-1, higher = more relevant) |
| `date_iso` | Normalized date (YYYY-MM-DD) for sorting |
| `region` | Geographic region (us, asia, europe, global) |
| `country` | Specific country (us, china, japan, india, uk, vietnam) |
| `series_name` | Goldman Sachs report series (e.g., "USA", "China", "Global Markets Daily") |
| `chunk_index` / `total_chunks` | Which part of the article (long articles are split into chunks) |
| `deduplicated` | If true, each result is a unique article (best-scoring chunk shown) |

**When to use `mongo_vector_search` vs `mongo_find`:**

| Query Type | Tool | Example |
|------------|------|---------|
| **Topic/theme search** | `mongo_vector_search` | "Find articles about Fed rate policy" |
| **Region/country focus** | `mongo_vector_search` | "Asia monetary policy articles" |
| **Recent by source** | `mongo_find` | "Get 5 latest Goldman Sachs reports" |
| **Filter by date only** | `mongo_find` | "Gavekal articles from last week" |

```python
# Simple queries - use mongo_find on macro_research_articles
# Get 5 most recent Goldman Sachs reports
collection: "macro_research_articles"
filter: {"source": "goldman_sachs"}
projection: {"title": 1, "date_iso": 1, "summary": 1}
sort: {"date_iso": -1}
limit: 5

# Get articles from specific date range
filter: {"source": "gavekal", "date_iso": {"$gte": "2025-12-01"}}
```

### Available MongoDB Collections

| Collection | Description |
|------------|-------------|
| `CompanyModels` | Analyst financial models (FOR_AI sheets) |
| `RealEstateProjects` | Real estate project valuations (NPV, IRR) |
| `AgentSkillSets` | AI agent skill configurations |
| `InterbankUpdates` | Interbank rate updates |
| `LendingRates` | Bank lending rate data |
| `MoCData` | Ministry of Construction real estate data |
| `macro_research_articles` | Macro research articles (Gavekal, Goldman Sachs) |
| `macro_research_embeddings` | Vector embeddings for semantic search |

---

## 3. Unified Python Executor

### `execute_python`
Execute Python code with access to SQL and MongoDB.

```python
code: """
# SQL query
df = sql_query("SELECT * FROM Market_Data WHERE TICKER = 'VNM'")

# MongoDB query
projects = mongo_find("RealEstateProjects", {"ticker": "VHM"})

# Process and return
result = {"price_data": df.to_dict(), "projects": len(projects)}
"""
timeout: 30
maxOutputSize: 100000
```

### Available Functions in Python Sandbox

**SQL:**
- `sql_query(sql)` - Execute SQL, returns DataFrame

**MongoDB:**
- `mongo_find(collection, filter, projection, sort, limit)` - Query, returns DataFrame
- `mongo_find_one(collection, filter, projection)` - Single doc, returns dict
- `mongo_aggregate(collection, pipeline)` - Aggregation, returns DataFrame
- `mongo_count(collection, filter)` - Count documents
- `mongo_distinct(collection, field, filter)` - Distinct values

**CSV Export:**
- `save_csv(data, path, downloadable=False)` - Save DataFrame to CSV
  - `downloadable=False`: Saves to server filesystem at `path`
  - `downloadable=True`: Returns a temporary download URL (30 min expiry)

**Chart Export (Matplotlib):**
- `save_figure(fig, filename, dpi=300)` - Save matplotlib figure to PNG with download URL
  - Always returns a download URL (30 min expiry)
  - `fig`: matplotlib figure object (or None for current figure)
  - `filename`: output filename (auto-appends .png if missing)
  - `dpi`: resolution (default 300)

**Why use these exports?** For complex visualizations on large datasets, query + chart + export in one `execute_python` call is much faster than multi-step computer tool workflows.

**Pre-imported:**
- `pd` (pandas), `np` (numpy), `scipy`, `sklearn`
- `plt` (matplotlib.pyplot), `matplotlib`

**Export Limits:**
- CSV: Max 50MB per file, max 10 files per execution
- Images: Max 10MB per file, max 10 images per execution
- Download URLs expire after 30 minutes

### Example: CSV with Download URL

```python
code: """
df = sql_query("SELECT * FROM Market_Data WHERE TICKER = 'VNM'")

# Get download URL instead of saving to server path
csv_info = save_csv(df, "vnm_data.csv", downloadable=True)

result = {"rows": len(df), "download": csv_info}
"""
```

**Response includes:**
```json
{
  "csv_files_saved": [{
    "path": "vnm_data.csv",
    "rows": 100,
    "size_bytes": 5432,
    "download_url": "https://claudetrade.live/mcp-sse/download/abc123...",
    "expires_in": "30 minutes"
  }]
}
```

### Example: Matplotlib Chart with Download URL

```python
code: """
df = sql_query('''
    SELECT TRADE_DATE, PX_LAST FROM Market_Data
    WHERE TICKER = 'VNM' ORDER BY TRADE_DATE DESC
''')
df['TRADE_DATE'] = pd.to_datetime(df['TRADE_DATE'])

# Create chart
fig, ax = plt.subplots(figsize=(10, 6))
ax.plot(df['TRADE_DATE'], df['PX_LAST'], 'b-', linewidth=1.5)
ax.set_title('VNM Price History')
ax.set_xlabel('Date')
ax.set_ylabel('Price (VND)')

# Save and get download URL
img_info = save_figure(fig, "vnm_price_chart.png")

result = {"chart": img_info}
"""
```

**Response includes:**
```json
{
  "images_saved": [{
    "filename": "vnm_price_chart.png",
    "size_bytes": 45678,
    "download_url": "https://claudetrade.live/mcp-sse/download/def456...",
    "expires_in": "30 minutes"
  }]
}
```

### Example: Combined Analysis with Exports

```python
code: """
# Get price data from SQL
prices = sql_query('''
    SELECT TICKER, TRADE_DATE, PX_LAST
    FROM Market_Data
    WHERE TICKER = 'VHM' AND TRADE_DATE >= '2025-01-01'
''')
prices['TRADE_DATE'] = pd.to_datetime(prices['TRADE_DATE'])

# Get NPV data from MongoDB
npv_data = mongo_find("RealEstateProjects",
    {"ticker": "VHM"},
    {"project_name": 1, "npv_to_company": 1}
)

# Calculate
total_npv = npv_data['npv_to_company'].sum()
latest_price = prices['PX_LAST'].iloc[-1]

# Export CSV with download URL
save_csv(prices, 'vhm_prices.csv', downloadable=True)

# Create and export chart
fig, ax = plt.subplots(figsize=(10, 5))
ax.plot(prices['TRADE_DATE'], prices['PX_LAST'])
ax.set_title(f'VHM Price (Latest: {latest_price:,.0f})')
save_figure(fig, 'vhm_chart.png')

result = {
    "ticker": "VHM",
    "total_npv_bn": total_npv / 1e9,
    "latest_price": latest_price,
    "data_points": len(prices)
}
"""
```

---

## 4. PostgreSQL CVD Tools (Real-time Data)

### `postgres_sector_cvd`
Get sector/list aggregated CVD data.

```python
list_name: "VN30"  # or "Banking", "Real Estate", "Brokers", etc.
date: "2025-12-17"  # optional, defaults to today
limit: 500
```

### `postgres_stock_cvd`
Get individual stock CVD data.

```python
symbol: "VHM"
date: "2025-12-17"
limit: 500
```

**CVD (Cumulative Volume Delta):**
- Measures cumulative buy vs sell volume/value
- Positive CVD = net buying pressure
- Negative CVD = net selling pressure

---

## 5. Broker Consensus Tools

### `supabase_consensus_analyze`
Get broker consensus analysis for a ticker.

```python
ticker: "MWG"
comparisons: 4  # number of recent broker reports to compare
```

### `supabase_consensus_new_reports`
Get recent broker reports with sentiment changes.

```python
days: 2  # look back 1-7 days
```

### `supabase_consensus_comparison_get`
Get full details of a specific broker comparison.

```python
ticker: "MWG"
broker: "HSC"  # optional
report_date: "2025-12-15"  # optional
```

### `supabase_consensus_pdf_get`
Get PDF URL or download broker research report.

```python
ticker: "MWG"
broker: "HSC"  # optional - if omitted, lists all reports
report_date: "2025-12-15"  # optional
download_path: "~/Downloads/report.pdf"  # optional
```

---

## Quick Reference

| Category | Tool | Use Case |
|----------|------|----------|
| Discovery | `find_relevant_tables` | Find which table has the data you need |
| SQL | `mssql_read_data` | Query market data, financials |
| SQL | `mssql_export_data` | Export large datasets to CSV (server path) |
| MongoDB | `mongo_find` | Query company models, RE projects |
| MongoDB | `mongo_aggregate` | Complex aggregations |
| MongoDB | `mongo_vector_search` | Semantic search macro research articles |
| Python | `execute_python` | Combined SQL+MongoDB analysis |
| Python | `save_csv(..., downloadable=True)` | Export CSV with download URL |
| Python | `save_figure(fig, filename)` | Export chart with download URL |
| CVD | `postgres_sector_cvd` | Sector money flow |
| CVD | `postgres_stock_cvd` | Stock-level money flow |
| Consensus | `supabase_consensus_analyze` | Broker sentiment analysis |

---

## Connection Details

- **SQL Server**: `sqls-dclab.database.windows.net/dclab` (read-only)
- **MongoDB**: `IRIS` database on MongoDB Atlas
- **PostgreSQL**: TimescaleDB for real-time CVD
- **Supabase**: Broker consensus data

---

*Last updated: December 24, 2025*
