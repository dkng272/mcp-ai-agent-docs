# Pylab MCP AI Agent Training Documentation

**Professional documentation for training AI agents to query financial databases via the Pylab MCP server**

This repository provides comprehensive schema documentation and query best practices for building AI-powered financial analysis tools using Pylab MCP.

---

## 📚 Documentation Files

### Reading Order for AI Agents
1. **MCP_TOOLS.md** - Start here (tool reference)
2. **DATABASE_SCHEMA.md** - When you need schema details
3. **QUERY_BEST_PRACTICES.md** - When writing queries

### [MCP_TOOLS.md](MCP_TOOLS.md)
**Entry point** for AI agents. Quick reference for Pylab MCP server tools: table discovery, SQL, MongoDB, Python executor, broker consensus, F319 forum analysis, and Zalo group signals.

### [DATABASE_SCHEMA.md](DATABASE_SCHEMA.md)
Complete database schema reference covering Azure SQL (44 tables) and MongoDB (6 collections). Includes market data, financials, sector operations, commodities, macro indicators, company models, and real estate valuations.

### [QUERY_BEST_PRACTICES.md](QUERY_BEST_PRACTICES.md)
Battle-tested query patterns for SQL and Python executor. Covers what works/fails, tool selection, JOINs, window functions, aggregations, and context reduction techniques.

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

- **MCP Server**: Pylab MCP (`pylab-mcp`) - unified access to SQL, MongoDB, Supabase
- **Databases**: Azure SQL, MongoDB Atlas, Supabase PostgreSQL
- **AI Platforms**: Claude Desktop, Claude Code
- **Query Language**: T-SQL, MongoDB queries, Python executor

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

- **v3.1** (2026-02-23): Renamed to Pylab MCP; removed PostgreSQL CVD tools (no longer on server); updated all docs
- **v3.0** (2026-02-03): Added F319 forum tools (5), Zalo group tools (5), sentiment dashboard; added CEODatabase (VEIL fund data); renamed to generic MCP_TOOLS.md
- **v2.0** (2025-12-17): Consolidated to 3 core docs (MCP_TOOLS → DATABASE_SCHEMA → QUERY_BEST_PRACTICES)
- **v1.5** (2025-12-17): Removed IRIS Manual Data guide (table shrunk); added MCP tools doc
- **v1.4** (2025-12-17): Added MongoDB collection schemas to DATABASE_SCHEMA.md
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

**Last Updated**: 2026-02-23
**Repository**: [github.com/dkng272/mcp-ai-agent-docs](https://github.com/dkng272/mcp-ai-agent-docs)
