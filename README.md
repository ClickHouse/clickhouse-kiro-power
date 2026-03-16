# ClickHouse Kiro Power

A [Kiro](https://kiro.dev) power that connects your IDE directly to [ClickHouse](https://clickhouse.com) databases. Query data, explore schemas, manage services, and get best-practice guidance for schema design, query optimization, and data ingestion — all through natural language in your editor.

## What It Does

This power gives Kiro the ability to talk to your ClickHouse databases via the [ClickHouse Cloud Remote MCP Server](https://github.com/ClickHouse/mcp-clickhouse). Instead of switching between your IDE and a SQL console, you can ask Kiro things like:

- "Show me the top 10 slowest queries in the last hour"
- "What tables are in my analytics database?"
- "Is my ORDER BY optimal for this query pattern?"
- "How many parts does my events table have?"
- "What's my billing for this month?"
- "Show me the ClickPipes on my service"

Kiro will run the right SQL, interpret the results, and give you actionable advice grounded in [28 best-practice rules](#whats-included) for ClickHouse.

## Quick Start

### Prerequisites

- [Kiro IDE](https://kiro.dev) installed
- A ClickHouse database (Cloud, self-hosted, or the public SQL Playground)

### 1. Enable MCP on Your Service

1. Log in to your [ClickHouse Cloud console](https://clickhouse.com/cloud)
2. Select your service → click **Connect** → enable **Remote MCP Server**

> Self-hosted ClickHouse and the public [SQL Playground](https://sql.clickhouse.com) are also supported via the open-source [`mcp-clickhouse`](https://github.com/ClickHouse/mcp-clickhouse) package. See `POWER.md` for configuration details.

### 2. Install the Power

Open Kiro and install the **ClickHouse Database** power from the Powers panel, or add it from the GitHub URL:

```
https://github.com/ClickHouse/clickhouse-kiro-power
```

Authenticate via your browser when prompted on first use — no API keys needed.

### 3. Start Querying

Once connected, the natural entry point is to discover your organization and services:

```
> "List my organizations and services"
> "What databases are on my service?"
```

Then explore and query:

```
> "Show me the schema for the events table"
> "What's the average duration by event type in the last 24 hours?"
> "Check my backup status"
> "How much has my organization spent this month?"
```

## What's Included

### MCP Tools (13 total)

**Query & Discovery**

| Tool | Description |
|------|-------------|
| `run_select_query` | Execute a read-only SELECT query on a ClickHouse service |
| `list_databases` | List all databases in a service |
| `list_tables` | List tables in a database with optional name filters and column details |

**Organization & Service Management**

| Tool | Description |
|------|-------------|
| `get_organizations` | List all accessible ClickHouse Cloud organizations |
| `get_organization_details` | Get details of a specific organization |
| `get_services_list` | List all services in an organization |
| `get_service_details` | Get details of a specific service |

**ClickPipes**

| Tool | Description |
|------|-------------|
| `list_clickpipes` | List all ClickPipes configured for a service |
| `get_clickpipe` | Get details of a specific ClickPipe |

**Backups**

| Tool | Description |
|------|-------------|
| `list_service_backups` | List all backups for a service (most recent first) |
| `get_service_backup_details` | Get details of a specific backup |
| `get_service_backup_configuration` | Get the backup configuration for a service |

**Billing**

| Tool | Description |
|------|-------------|
| `get_organization_cost` | Get billing and usage data for an organization (max 31-day window) |

Most tools require a `serviceId` and/or `organizationId`. Use `get_organizations` → `get_services_list` to discover these values.

### Steering: 28 Best-Practice Rules

The power ships with a comprehensive steering file that Kiro uses automatically when helping with ClickHouse work. It covers:

**Schema Design (Critical)**
- Plan ORDER BY before table creation (it's immutable)
- Order columns low-to-high cardinality for effective index pruning
- Use native types (UInt32, DateTime, UUID) instead of String
- Use `LowCardinality(String)` for columns with < 10K unique values
- Avoid `Nullable` unless semantically required

**Query Optimization (Critical)**
- Choose the right JOIN algorithm for your data sizes
- Filter tables before joining, not after
- Use data skipping indices (bloom_filter, set, minmax) for non-ORDER BY filters
- Use materialized views for real-time pre-aggregation

**Insert Strategy (Critical)**
- Batch inserts at 10K–100K rows per INSERT
- Use async inserts for high-frequency small batches
- Use ReplacingMergeTree instead of ALTER TABLE UPDATE
- Avoid OPTIMIZE TABLE FINAL — let background merges work

**Partitioning (High)**
- Keep partition cardinality low (100–1,000 values)
- Use partitioning for data lifecycle, not query optimization
- Start without partitioning and add it when needed

## Project Structure

```
.
├── POWER.md                              # Power definition and documentation
├── mcp.json                              # MCP server configuration (Cloud Remote MCP)
├── steering/
│   └── clickhouse-best-practices.md      # 28 best-practice rules (from ClickHouse Agent Skills)
├── submodules/
│   └── agent-skills/                     # Git submodule: ClickHouse/agent-skills
├── assets/
│   └── logo.svg                          # ClickHouse logo
├── LICENSE                               # Apache-2.0
└── README.md
```

### Agent Skills Sync

The steering content is sourced from [ClickHouse/agent-skills](https://github.com/ClickHouse/agent-skills) — the canonical best-practice rules used across all ClickHouse agent integrations (Cursor, Kiro, etc.).

The `steering/clickhouse-best-practices.md` file is a copy of `AGENTS.md` from the agent-skills repo, with Kiro-specific `inclusion: auto` frontmatter prepended. A weekly GitHub Action ([sync-agent-skills](.github/workflows/sync-agent-skills.yml)) checks the submodule for updates and opens a PR if the content has changed.

When cloning for development, use `--recurse-submodules` to populate the submodule:

```bash
git clone --recurse-submodules https://github.com/ClickHouse/clickhouse-kiro-power
```

## Troubleshooting

| Issue | Solution |
|-------|----------|
| How do I find my `serviceId`? | Call `get_organizations` (no parameters) to get your `organizationId`, then `get_services_list` to list services with their IDs |
| MCP server not connected | Reconnect from Kiro's MCP panel; verify MCP is enabled in Cloud console |
| Authentication failed | Re-authenticate via browser; verify MCP is enabled in Cloud console |
| Service is idle/sleeping | Wake the service from ClickHouse Cloud console, then retry |
| "Too many parts" error | Increase batch size to 10K+ rows per INSERT; check partition cardinality |
| "Memory limit exceeded" | Add filters to reduce data scanned; use `partial_merge` JOIN; add `LIMIT` |
| Slow queries | Verify filters align with ORDER BY; use `EXPLAIN indexes = 1` |
| "Cannot modify ORDER BY" | Create a new table with the correct ORDER BY, migrate data, then rename |

## Resources

- [ClickHouse Documentation](https://clickhouse.com/docs)
- [ClickHouse Best Practices](https://clickhouse.com/docs/best-practices)
- [ClickHouse MCP Server](https://github.com/ClickHouse/mcp-clickhouse)
- [ClickHouse Agent Skills](https://github.com/ClickHouse/agent-skills)
- [Enable Remote MCP on ClickHouse Cloud](https://clickhouse.com/docs/use-cases/AI/MCP/remote_mcp)
- [Kiro IDE](https://kiro.dev)

## Legal & Support

- **License:** [Apache-2.0](https://spdx.org/licenses/Apache-2.0.html)
- **MCP Server License:** [Apache-2.0](https://spdx.org/licenses/Apache-2.0.html) ([source](https://github.com/ClickHouse/mcp-clickhouse/blob/main/LICENSE))
- **Best Practices Content:** [Apache-2.0](https://spdx.org/licenses/Apache-2.0.html) ([source](https://github.com/ClickHouse/agent-skills/blob/main/LICENSE))
- **Privacy Policy:** [ClickHouse Privacy Policy](https://clickhouse.com/legal/privacy-policy)
- **Support:** [GitHub Issues](https://github.com/ClickHouse/clickhouse-kiro-power/issues) | [Community Slack](https://clickhouse.com/slack)
