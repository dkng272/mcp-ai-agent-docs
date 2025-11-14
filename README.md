# MCP AI Agent Training Documentation

**AI agent training materials for Microsoft SQL Server MCP and specialized database connections**

This repository contains schema documentation and query best practices for training AI agents (Claude, ChatGPT, etc.) to work with financial databases via Model Context Protocol (MCP).

---

## 📚 Documentation Files

### 1. [DATABASE_SCHEMA_SQL_ONLY.md](DATABASE_SCHEMA_SQL_ONLY.md)
**Complete database schema for Vietnamese financial market data**

- 37 tables covering stocks, financials, forecasts, market data, and sector-specific metrics
- Detailed column descriptions, data types, and relationships
- Primary keys, indexes, and query optimization tips
- Coverage: Banking, Brokerage, Insurance, Agriculture, Energy, Shipping, Aviation, and more

**Use this for**: Teaching AI agents about available tables and data structure

---

### 2. [MCP_SQL_QUERY_BEST_PRACTICES.md](MCP_SQL_QUERY_BEST_PRACTICES.md)
**Proven query patterns for Microsoft SQL MCP Server**

- Battle-tested patterns from extensive stress testing
- What works ✅ and what fails ❌ with MCP SQL queries
- Window functions, aggregations, joins, and derived tables
- Critical rules (NULLIF for division, no CTEs, float data types)
- Real-world examples for forecast analysis, sector screening, and financial metrics

**Use this for**: Teaching AI agents how to write reliable SQL queries that work with MCP

---

### 3. [VIETNAM_MACRO_DATA_SCHEMA_AND_BEST_PRACTICES.md](VIETNAM_MACRO_DATA_SCHEMA_AND_BEST_PRACTICES.md)
**Specialized documentation for Vietnam macroeconomic data**

- CEIC economic indicators (545 series, 2004-2030)
- GDP, FX rates, interest rates, government fiscal data
- Optimized query patterns for time-series analysis
- Moving averages, YoY growth, seasonal patterns

**Use this for**: Training a specialized macro economics AI agent (table-filtered MCP)

---

## 🚀 Quick Start

### For AI Agent Training

1. **Choose the right documentation** based on your use case:
   - Full financial database access → Use all three files
   - Macro data only → Use VIETNAM_MACRO_DATA_SCHEMA_AND_BEST_PRACTICES.md
   - Query optimization → Start with MCP_SQL_QUERY_BEST_PRACTICES.md

2. **Add to your MCP configuration**:
   ```json
   {
     "mcpServers": {
       "MicrosoftSQL": {
         "command": "node",
         "args": ["/path/to/MssqlMcp/Node/dist/index.js"],
         "env": {
           "SERVER_NAME": "your-server.database.windows.net",
           "DATABASE_NAME": "your-database",
           "READONLY": "true"
         }
       }
     }
   }
   ```

3. **Train your AI agent** by providing the relevant .md files as context

### For Specialized Agents (Table Filtering)

Create focused agents by restricting table access:

```json
{
  "VietnamMacroData": {
    "command": "node",
    "args": ["/path/to/MssqlMcp/Node/dist/index.js"],
    "env": {
      "SERVER_NAME": "your-server.database.windows.net",
      "DATABASE_NAME": "your-database",
      "READONLY": "true",
      "ALLOWED_TABLES": "dbo.ceic_macro_data"
    }
  }
}
```

See [Table Filtering Guide](#table-filtering) below for implementation details.

---

## 💡 Use Cases

### 1. Banking Sector Analysis
```
ALLOWED_TABLES: dbo.BankingMetrics, dbo.FA_Quarterly, dbo.Bank_Rates
```
- Specialized agent for CASA, NIM, NPL analysis
- Banking-specific ratios (ROA, ROE, CIR)
- No confusion with other sector data

### 2. Valuation Screening
```
ALLOWED_TABLES: dbo.Market_Data, dbo.Valuation, dbo.FA_Quarterly
```
- P/E, P/B, EV/EBITDA analysis
- Undervalued/overvalued stock screening
- Relative valuation across sectors

### 3. Earnings Season Scanner
```
ALLOWED_TABLES: dbo.FA_Quarterly, dbo.Forecast, dbo.Market_Data
```
- Fast beat/miss analysis during earnings season
- Compare actuals vs forecasts
- Limited table scope = faster queries

### 4. Macro Economics Specialist
```
ALLOWED_TABLES: dbo.ceic_macro_data
```
- GDP, FX, interest rate analysis
- Government fiscal data tracking
- No access to company-specific data

---

## 🔧 Table Filtering Implementation

The Microsoft SQL MCP Server can be modified to restrict access to specific tables using the `ALLOWED_TABLES` environment variable.

### How It Works

1. **Modify MCP Server Code** (Node.js implementation):
   - Add table filtering logic to `ListTableTool.js`, `DescribeTableTool.js`, `ReadDataTool.js`
   - Parse `ALLOWED_TABLES` env variable (comma-separated list)
   - Validate queries against allowed tables before execution

2. **Set Environment Variable**:
   ```json
   "ALLOWED_TABLES": "dbo.ceic_macro_data,dbo.iris_macro_data"
   ```

3. **Result**: Agent can only:
   - List allowed tables
   - Describe allowed table schemas
   - Query data from allowed tables
   - JOINs between allowed tables only

### Benefits

✅ **Security**: Limit agent access to sensitive data
✅ **Performance**: Smaller data scope = faster queries
✅ **Focus**: Specialized agents stay on topic
✅ **Training**: Reduce cognitive load for AI agents
✅ **Compliance**: Separate client-facing vs internal data

### Implementation Code

Contact repository maintainer for modified MCP server code with table filtering support.

---

## 📊 Data Coverage

### Geographic Focus
- **Primary**: Vietnam stock market and macroeconomic data
- **Global**: Commodity prices, global indices, US Dollar Index

### Time Range
- **Historical**: 2004 - Present
- **Forecasts**: Up to 2030 (select series)
- **Update Frequency**: Daily to Quarterly depending on data type

### Data Sources
- **CEIC**: Official economic statistics
- **FIIN**: Vietnamese financial statements
- **SSI/TCBS**: Market data and prices
- **Proprietary**: Internal forecasts and analysis

---

## 🛠 Technical Requirements

### MCP Server
- **Microsoft SQL MCP Server** (Node.js or Python version)
- Azure SQL Database or SQL Server connection
- Read-only recommended for agent access

### AI Platforms Tested
- ✅ Claude Desktop (Anthropic)
- ✅ Claude Code
- ⚠️ ChatGPT (via MCP support when available)

### Database Requirements
- Azure SQL Database or SQL Server 2016+
- Interactive browser authentication or SQL authentication
- Read permissions on target tables

---

## 📖 Documentation Standards

All documentation follows these principles:

1. **Tested Patterns Only**: Every query example has been validated
2. **What Works & What Fails**: Explicit guidance on MCP limitations
3. **Real-World Examples**: Based on actual financial analysis use cases
4. **Performance Tips**: Query optimization for production use
5. **Clear Structure**: Easy navigation and searchability

---

## 🤝 Contributing

This is a living documentation repository. Contributions welcome for:

- New query patterns and examples
- Additional table schemas
- Performance optimization tips
- Use case examples
- Bug fixes in documentation

**How to contribute**:
1. Fork the repository
2. Update documentation with tested examples
3. Submit pull request with clear description
4. Ensure all query examples are validated

---

## 📝 Version History

- **v1.2** (2025-11-12): Added Vietnam Macro Data specialized documentation
- **v1.1** (2025-11-11): Updated to reflect float data types (removed TRY_CAST)
- **v1.0** (2025-11-10): Initial release with full schema and best practices

---

## 📧 Contact & Support

For questions, issues, or access to modified MCP server code:
- **GitHub Issues**: [Create an issue](../../issues)
- **Discussions**: [Start a discussion](../../discussions)

---

## ⚖️ License

Documentation: MIT License
Data: Proprietary - Not included in this repository

---

## 🌟 Acknowledgments

- **Anthropic** - Claude AI and Model Context Protocol
- **Microsoft** - SQL Server MCP implementation
- **CEIC** - Economic data provider
- **FIIN** - Vietnamese financial data

---

**Last Updated**: 2025-11-12
**Maintained by**: Duy Nguyen
