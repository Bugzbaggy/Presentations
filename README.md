# Presentations

Talks, demo code, and written field notes on SQL Server, Azure SQL, and
database operations.

---

## Written field notes

Longer write-ups from production work. Everything is anonymized — hostnames,
database names, and accounts are placeholders — but the sequences, commands,
and failures are exactly as they happened.

| Post | What it covers |
|---|---|
| [In-Place Upgrading a SQL Server Always On Cluster to Windows Server 2025](posts/upgrading-always-on-to-windows-server-2025.md) | A rolling OS upgrade of a two-node AG with zero planned downtime — the ordering that matters most, and the eight things that went wrong |
| [A Read-Only SQL MCP, Powered by dbatools](posts/sql-server-mcp-server-for-dbas.md) | Building an MCP server that lets an AI assistant investigate a live multi-region fleet while being structurally incapable of writing |
| [Every Pull Request Gets Its Own SQL Server](posts/per-pr-database-integration-testing.md) | Per-PR database integration testing on disposable containers, and why only one check is allowed to block a PR |

Each has working code behind it:

| Project | Related post |
|---|---|
| [mssql-hadr-ops](https://github.com/Bugzbaggy/mssql-hadr-ops) | The WS2025 rolling upgrade |
| [mssql-dba-mcp](https://github.com/Bugzbaggy/mssql-dba-mcp) | The read-only SQL MCP |
| [tsqlt-integration-pipeline](https://github.com/Bugzbaggy/tsqlt-integration-pipeline) | Per-PR integration testing |
| [mssql-dacpac-cicd](https://github.com/Bugzbaggy/mssql-dacpac-cicd) | Database CI/CD |
| [mssql-data-api](https://github.com/Bugzbaggy/mssql-data-api) | Governed read-only data API |
| [nitsql](https://github.com/Bugzbaggy/nitsql) | Multi-dialect SQL analyzer |
| [schemalore](https://github.com/Bugzbaggy/schemalore) | SSDT documentation skill |

## Session material

Slide decks and demo code from conference and user-group sessions:

- Analyzing Azure Monitor Log Data for Azure SQL Database
- Azure SQL Database — Where is my SQL Agent
- Azure SQL Database — Business Continuity During Disaster
- Kusto Query Language
- New features in Management Studio
- Options and considerations for migrating SQL Server databases to Azure
- Performance Optimization with Azure SQL Database
- SQL Assessment — Microsoft's Best Practices Checker
- The magnificent seven — Intelligent Query Processing in SQL Server
- Think like the Cardinality Estimator
- What the heck is a checkpoint, and why should I care
- XeventSample

Lightning talks:

- Analyzing Azure Monitor Log data in Azure Data Studio
- NotebookJobs end-to-end demo
- Query Store hints demo

## License

See [LICENSE](LICENSE). Written posts are released under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
