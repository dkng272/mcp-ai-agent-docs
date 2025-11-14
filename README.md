# MCP AI Agent Training Documentation

**Professional documentation for training AI agents to query financial databases via Model Context Protocol (MCP)**

This repository provides comprehensive schema documentation and query best practices for building AI-powered financial analysis tools using Microsoft SQL Server MCP.

---

## 📚 Documentation Files

### [DATABASE_SCHEMA_SQL_ONLY.md](DATABASE_SCHEMA_SQL_ONLY.md)
Complete database schema reference covering Vietnamese financial market data across multiple sectors including banking, insurance, brokerage, commodities, and macroeconomic indicators.

### [MCP_SQL_QUERY_BEST_PRACTICES.md](MCP_SQL_QUERY_BEST_PRACTICES.md)
Battle-tested query patterns and optimization techniques specifically validated for Microsoft SQL MCP Server. Includes proven approaches for aggregations, window functions, joins, and time-series analysis.

### [VIETNAM_MACRO_DATA_SCHEMA_AND_BEST_PRACTICES.md](VIETNAM_MACRO_DATA_SCHEMA_AND_BEST_PRACTICES.md)
Specialized guide for Vietnam macroeconomic data including CEIC indicators (GDP, FX rates, interest rates, government fiscal data) with optimized time-series query patterns.

### [IRIS_MANUAL_DATA_GUIDE.md](IRIS_MANUAL_DATA_GUIDE.md)
Documentation for manually collected operational and market data including commodity prices, sector-specific metrics, and company operational indicators.

---

## 🎯 Use Cases

- **Financial Analysis**: Automated equity research, sector analysis, valuation screening
- **Alternative Data**: Operational metrics tracking before quarterly earnings
- **Macro Economics**: Economic indicator monitoring and correlation analysis
- **Risk Management**: Portfolio analytics and risk factor modeling
- **Trading Signals**: Quantitative strategy development and backtesting

---

## 🔑 Key Features

### Tested Query Patterns
Every query example has been validated against production MCP servers to ensure reliability.

### Performance Optimization
Includes specific guidance on indexing, aggregation strategies, and query complexity management for MCP environments.

### Specialized Agents
Documentation supports creating focused AI agents with restricted data access for security and performance benefits.

### Real-World Examples
Query patterns based on actual financial analysis workflows including earnings analysis, sector rotation, and valuation screening.

---

## 📊 Data Coverage

- **Geographic Focus**: Vietnam stock market with global commodity benchmarks
- **Time Range**: Historical data from 2004, forecasts to 2030
- **Sectors**: Banking, Insurance, Brokerage, Commodities, Energy, Agriculture, Aviation, Shipping, Retail
- **Frequency**: Daily to Quarterly depending on data type

---

## 🛠 Technical Stack

- **MCP Server**: Microsoft SQL Server MCP (Node.js or Python)
- **Database**: Azure SQL Database / SQL Server 2016+
- **AI Platforms**: Claude Desktop, Claude Code
- **Query Language**: T-SQL with MCP-specific optimizations

---

## 📖 Documentation Philosophy

1. **Accuracy First**: Only validated patterns and examples
2. **MCP-Specific**: Addresses unique MCP query limitations and capabilities
3. **Production-Ready**: Optimization tips for real-world deployment
4. **Comprehensive**: From basic schema to advanced time-series analysis

---

## 🤝 Contributing

Contributions welcome for:
- Additional query optimization techniques
- New use case examples
- Documentation improvements
- Bug reports

Please submit pull requests with tested examples and clear descriptions.

---

## 📝 Version History

- **v1.3** (2025-11-12): Added IRIS Manual Data documentation
- **v1.2** (2025-11-12): Added Vietnam Macro Data specialized guide
- **v1.1** (2025-11-11): Updated for float data types
- **v1.0** (2025-11-10): Initial release

---

## ⚖️ License

Documentation: MIT License
Data: Proprietary - Not included in repository

---

## 🌟 Acknowledgments

- **Anthropic** - Model Context Protocol
- **Microsoft** - SQL Server MCP implementation
- **CEIC** - Economic data standards

---

**Last Updated**: 2025-11-12
**Repository**: [github.com/dkng272/mcp-ai-agent-docs](https://github.com/dkng272/mcp-ai-agent-docs)
