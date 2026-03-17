---
name: "clickhouse"
displayName: "ClickHouse Database"
description: "Query, explore, and manage ClickHouse databases with best-practice guidance for schema design, query optimization, and data ingestion"
keywords: ["clickhouse", "database", "analytics", "olap", "sql", "columnar", "data", "query", "mergetree", "clickhouse cloud", "clickpipes", "backup", "billing", "service", "organization"]
author: "ClickHouse"
---

# Onboarding

## Step 1: Enable MCP connection

The ClickHouse Cloud Remote MCP Server must be enabled on the user's ClickHouse Cloud service. They can enable it from the **Connect** view in the [ClickHouse Cloud console](https://clickhouse.com/cloud). See [Enable MCP on ClickHouse](https://clickhouse.com/docs/use-cases/AI/MCP/remote_mcp).

Authentication is handled via OAuth -- the user will be prompted to authenticate in their browser on first use. No API keys required.

For self-hosted ClickHouse, users should use the open-source MCP server (`mcp-clickhouse` via pip/uv) instead. See the [Configuration](#configuration) section below.

## Step 2: Add schema review hook

Add a hook to `.kiro/hooks/review-clickhouse-schema.kiro.hook`

```json
{
  "enabled": true,
  "name": "Review ClickHouse Schema",
  "description": "Review CREATE TABLE statements against ClickHouse best practices (ORDER BY, data types, partitioning, engine selection)",
  "version": "1",
  "when": {
    "type": "userTriggered"
  },
  "then": {
    "type": "askAgent",
    "prompt": "Load the clickhouse-best-practices steering file. Review any CREATE TABLE or ALTER TABLE statements in the current file or recent conversation against the 28 ClickHouse best-practice rules. Check: ORDER BY column order (low-to-high cardinality), data type selection (native types, LowCardinality, Nullable usage), partition strategy (cardinality, lifecycle alignment), and engine selection (ReplacingMergeTree for updates, CollapsingMergeTree for deletes). Report violations with specific rule references and suggested fixes."
  }
}
```

# ClickHouse Database Power

## Overview

Query, explore, and manage ClickHouse databases directly from your IDE. This power provides access to ClickHouse Cloud through the official MCP server and includes comprehensive best-practice guidance adapted from the [ClickHouse Agent Skills](https://github.com/ClickHouse/agent-skills).

**Key capabilities:**
- **SQL Queries**: Execute read-only SELECT queries against ClickHouse services
- **Database Discovery**: List databases and browse table schemas with column details
- **Organization & Service Management**: List organizations, services, and service details
- **ClickPipes**: Inspect data ingestion pipelines configured on services
- **Backups**: View backup history, details, and backup configuration
- **Billing**: Retrieve organization-level cost and usage data
- **Best Practices**: 28 rules covering schema design, query optimization, and insert strategy

**Perfect for:**
- Exploring and querying ClickHouse databases
- Designing new tables with proper ORDER BY, partitioning, and data types
- Optimizing slow queries (JOINs, filters, indices)
- Managing ClickHouse Cloud services and organizations
- Monitoring backup status and billing costs
- Reviewing schemas and queries against best practices

## Available Steering Files

This power has the following steering files:
- **clickhouse-best-practices** (inclusion: auto) -- Comprehensive ClickHouse best-practices guide with 28 rules for schema design, query optimization, and insert strategy. Includes SQL examples comparing incorrect vs. correct implementations, impact metrics, and references to official documentation. Automatically loaded when designing schemas, creating tables, optimizing queries, choosing ORDER BY keys, planning data ingestion, or reviewing ClickHouse configurations.

## Available MCP Servers

### clickhouse

**Endpoint:** `https://mcp.clickhouse.cloud/mcp`
**Connection:** Remote MCP server via OAuth (ClickHouse Cloud)

**Tools:**

#### Query & Discovery

1. **run_select_query** -- Execute a read-only SELECT query on a ClickHouse service via the Query API
   - Required: `query` (string) -- A valid ClickHouse SQL SELECT query
   - Required: `serviceId` (string, UUID) -- Unique identifier of the ClickHouse service

2. **list_databases** -- Retrieve all databases available in a ClickHouse service
   - Required: `serviceId` (string, UUID) -- ID of the ClickHouse service

3. **list_tables** -- Retrieve tables in a ClickHouse database, including columns
   - Required: `serviceId` (string, UUID) -- Unique identifier of the ClickHouse service
   - Required: `database` (string) -- Name of the database to list tables from
   - Optional: `like` (string) -- SQL LIKE pattern to filter tables (e.g., `"events_%"`)
   - Optional: `notLike` (string) -- SQL LIKE pattern to exclude tables

#### Organization & Service Management

4. **get_organizations** -- Retrieve all accessible ClickHouse Cloud organizations
   - No parameters required

5. **get_organization_details** -- Return details of a single organization
   - Required: `organizationId` (string, UUID) -- ID of the organization

6. **get_services_list** -- Retrieve all services in a ClickHouse Cloud organization
   - Required: `organizationId` (string, UUID) -- ID of the organization

7. **get_service_details** -- Return details of a service in an organization
   - Required: `organizationId` (string, UUID) -- ID of the organization
   - Required: `serviceId` (string, UUID) -- ID of the service

#### ClickPipes

8. **list_clickpipes** -- Retrieve all ClickPipes configured for a service
   - Required: `organizationId` (string, UUID) -- ID of the organization
   - Required: `serviceId` (string, UUID) -- ID of the service

9. **get_clickpipe** -- Return details of a specific ClickPipe
   - Required: `organizationId` (string, UUID) -- ID of the organization
   - Required: `serviceId` (string, UUID) -- ID of the service
   - Required: `clickPipeId` (string, UUID) -- ID of the ClickPipe

#### Backups

10. **list_service_backups** -- Return all backups for a service (most recent first)
    - Required: `organizationId` (string, UUID) -- ID of the organization
    - Required: `serviceId` (string, UUID) -- ID of the service

11. **get_service_backup_details** -- Return details of a single backup
    - Required: `organizationId` (string, UUID) -- ID of the organization
    - Required: `serviceId` (string, UUID) -- ID of the service
    - Required: `backupId` (string, UUID) -- ID of the backup

12. **get_service_backup_configuration** -- Return the backup configuration for a service
    - Required: `organizationId` (string, UUID) -- ID of the organization
    - Required: `serviceId` (string, UUID) -- ID of the service

#### Billing

13. **get_organization_cost** -- Retrieve billing and usage cost data for an organization (max 31-day window)
    - Required: `organizationId` (string, UUID) -- ID of the organization
    - Optional: `from_date` (string) -- Start date (YYYY-MM-DD)
    - Optional: `to_date` (string) -- End date inclusive (YYYY-MM-DD, max 30 days after from_date)

## Tool Usage Examples

### Discovering Organizations and Services

```javascript
// List all accessible organizations
usePower("clickhouse", "clickhouse", "get_organizations", {})
// Returns: Array of organizations with IDs, names, and metadata

// List services in an organization
usePower("clickhouse", "clickhouse", "get_services_list", {
  "organizationId": "org-uuid-here"
})
// Returns: Array of services with IDs, names, regions, and status
```

### Running Queries

```javascript
// Simple SELECT query
usePower("clickhouse", "clickhouse", "run_select_query", {
  "serviceId": "service-uuid-here",
  "query": "SELECT count() FROM events WHERE timestamp > now() - INTERVAL 1 HOUR"
})
// Returns: { meta, data, rows, statistics }

// Inspect table schema
usePower("clickhouse", "clickhouse", "run_select_query", {
  "serviceId": "service-uuid-here",
  "query": "DESCRIBE TABLE events"
})
// Returns: Column names, types, defaults, and comments
```

### Listing Databases and Tables

```javascript
// List all databases
usePower("clickhouse", "clickhouse", "list_databases", {
  "serviceId": "service-uuid-here"
})
// Returns: Array of database names

// List tables with filtering
usePower("clickhouse", "clickhouse", "list_tables", {
  "serviceId": "service-uuid-here",
  "database": "default",
  "like": "events_%"
})
// Returns: Tables with column details
```

### Checking Backups

```javascript
// List recent backups
usePower("clickhouse", "clickhouse", "list_service_backups", {
  "organizationId": "org-uuid-here",
  "serviceId": "service-uuid-here"
})
// Returns: Backups sorted by most recent first
```

### Checking Costs

```javascript
// Get organization billing for a date range
usePower("clickhouse", "clickhouse", "get_organization_cost", {
  "organizationId": "org-uuid-here",
  "from_date": "2025-03-01",
  "to_date": "2025-03-15"
})
// Returns: Grand total and daily per-entity usage cost records
```

## Combining Tools (Workflows)

### Workflow 1: Service Discovery

```javascript
// Step 1: Get organizations
const orgs = usePower("clickhouse", "clickhouse", "get_organizations", {})

// Step 2: List services in the organization
const services = usePower("clickhouse", "clickhouse", "get_services_list", {
  "organizationId": orgs[0].id
})

// Step 3: Get details for a specific service
const details = usePower("clickhouse", "clickhouse", "get_service_details", {
  "organizationId": orgs[0].id,
  "serviceId": services[0].id
})
```

### Workflow 2: Database Exploration

```javascript
// Step 1: List all databases
const databases = usePower("clickhouse", "clickhouse", "list_databases", {
  "serviceId": "service-uuid"
})

// Step 2: List tables in a database
const tables = usePower("clickhouse", "clickhouse", "list_tables", {
  "serviceId": "service-uuid",
  "database": "analytics"
})

// Step 3: Inspect a specific table
const schema = usePower("clickhouse", "clickhouse", "run_select_query", {
  "serviceId": "service-uuid",
  "query": "DESCRIBE TABLE analytics.events"
})

// Step 4: Check table engine, ORDER BY, and size
const tableInfo = usePower("clickhouse", "clickhouse", "run_select_query", {
  "serviceId": "service-uuid",
  "query": "SELECT engine, sorting_key, partition_key, total_rows, formatReadableSize(total_bytes) as size FROM system.tables WHERE database = 'analytics' AND name = 'events'"
})

// Step 5: Sample data
const sample = usePower("clickhouse", "clickhouse", "run_select_query", {
  "serviceId": "service-uuid",
  "query": "SELECT * FROM analytics.events LIMIT 10"
})
```

### Workflow 3: Query Performance Investigation

```javascript
// Step 1: Find slow queries in the query log
const slowQueries = usePower("clickhouse", "clickhouse", "run_select_query", {
  "serviceId": "service-uuid",
  "query": "SELECT query, query_duration_ms, read_rows, read_bytes, result_rows FROM system.query_log WHERE type = 'QueryFinish' AND query_duration_ms > 1000 AND event_time > now() - INTERVAL 1 HOUR ORDER BY query_duration_ms DESC LIMIT 10"
})

// Step 2: Check if filters align with ORDER BY
const tableKeys = usePower("clickhouse", "clickhouse", "run_select_query", {
  "serviceId": "service-uuid",
  "query": "SELECT sorting_key FROM system.tables WHERE database = 'analytics' AND name = 'events'"
})

// Step 3: Verify index usage with EXPLAIN
const explain = usePower("clickhouse", "clickhouse", "run_select_query", {
  "serviceId": "service-uuid",
  "query": "EXPLAIN indexes = 1 SELECT * FROM analytics.events WHERE user_id = 12345"
})

// Step 4: Check part health
const partHealth = usePower("clickhouse", "clickhouse", "run_select_query", {
  "serviceId": "service-uuid",
  "query": "SELECT table, count() as parts, sum(rows) as total_rows FROM system.parts WHERE active AND database = 'analytics' GROUP BY table ORDER BY parts DESC"
})
```

### Workflow 4: Schema Review

```javascript
// Step 1: Get the CREATE TABLE statement
const createTable = usePower("clickhouse", "clickhouse", "run_select_query", {
  "serviceId": "service-uuid",
  "query": "SHOW CREATE TABLE analytics.events"
})

// Step 2: Check column cardinality for ORDER BY optimization
const cardinality = usePower("clickhouse", "clickhouse", "run_select_query", {
  "serviceId": "service-uuid",
  "query": "SELECT uniq(event_type) as event_type_uniq, uniq(user_id) as user_id_uniq, uniq(toDate(timestamp)) as date_uniq FROM analytics.events"
})

// Step 3: Check partition health
const partitions = usePower("clickhouse", "clickhouse", "run_select_query", {
  "serviceId": "service-uuid",
  "query": "SELECT partition, count() as parts, sum(rows) as rows, formatReadableSize(sum(bytes_on_disk)) as size FROM system.parts WHERE table = 'events' AND database = 'analytics' AND active GROUP BY partition ORDER BY partition"
})

// Step 4: Analyze column compression
const columnSizes = usePower("clickhouse", "clickhouse", "run_select_query", {
  "serviceId": "service-uuid",
  "query": "SELECT column, type, formatReadableSize(sum(column_data_compressed_bytes)) as compressed, formatReadableSize(sum(column_data_uncompressed_bytes)) as uncompressed, round(sum(column_data_uncompressed_bytes) / sum(column_data_compressed_bytes), 2) as ratio FROM system.parts_columns WHERE table = 'events' AND database = 'analytics' AND active GROUP BY column, type ORDER BY sum(column_data_uncompressed_bytes) DESC"
})
```

## When to Load Steering Files

- Designing schemas or reviewing CREATE TABLE statements --> `clickhouse-best-practices.md`
- Optimizing queries, JOINs, or choosing ORDER BY keys --> `clickhouse-best-practices.md`
- Planning data ingestion, insert batching, or update strategy --> `clickhouse-best-practices.md`
- Choosing data types, partitioning strategy, or index types --> `clickhouse-best-practices.md`

## Best Practices

### Do:

- **Plan ORDER BY before table creation** -- it is immutable after creation
- **Order columns low-to-high cardinality** in ORDER BY for effective granule skipping
- **Use native types** (UInt32, DateTime, UUID) instead of String for everything
- **Use LowCardinality(String)** for columns with fewer than 10K unique values
- **Batch inserts 10K-100K rows** per INSERT to avoid part explosion
- **Filter on ORDER BY prefix columns** in queries for index usage
- **Use ReplacingMergeTree** instead of ALTER TABLE UPDATE for frequent updates
- **Start without partitioning** and add it later for data lifecycle needs
- **Keep partition cardinality low** (100-1,000 values)
- **Use skipping indices** (bloom_filter, set, minmax) for non-ORDER BY filter columns

### Don't:

- **Use Nullable everywhere** -- use DEFAULT values instead (Nullable adds storage overhead)
- **Use ALTER TABLE UPDATE/DELETE** for frequent changes -- use specialized MergeTree engines
- **Run OPTIMIZE TABLE FINAL regularly** -- let background merges work
- **Use String for everything** -- native types give 2-10x storage reduction
- **Put high-cardinality columns first** in ORDER BY -- prevents index pruning
- **Use very high partition cardinality** -- causes "too many parts" errors
- **Insert single rows** -- each INSERT creates a data part
- **Skip query analysis** before choosing ORDER BY -- wrong choice requires full migration

## Configuration

### ClickHouse Cloud (Default)

Authentication is handled via OAuth. No API keys needed.

1. Log in to [ClickHouse Cloud console](https://clickhouse.com/cloud)
2. Click **Connect** on your service
3. Enable **Remote MCP Server**
4. The power uses `https://mcp.clickhouse.cloud/mcp` automatically
5. Authenticate via browser when prompted on first use

### Self-Hosted ClickHouse

For self-hosted ClickHouse, replace the `mcp.json` config with the open-source MCP server:

```json
{
  "mcpServers": {
    "clickhouse": {
      "command": "uv",
      "args": [
        "run", "--with", "mcp-clickhouse",
        "--python", "3.10", "mcp-clickhouse"
      ],
      "env": {
        "CLICKHOUSE_HOST": "<your-clickhouse-host>",
        "CLICKHOUSE_PORT": "8443",
        "CLICKHOUSE_USER": "<your-user>",
        "CLICKHOUSE_PASSWORD": "<your-password>",
        "CLICKHOUSE_SECURE": "true",
        "CLICKHOUSE_VERIFY": "true"
      }
    }
  }
}
```

**Required:** `CLICKHOUSE_HOST`, `CLICKHOUSE_USER`, `CLICKHOUSE_PASSWORD`
**Optional:** `CLICKHOUSE_PORT` (default 8443), `CLICKHOUSE_SECURE` (default true), `CLICKHOUSE_DATABASE`, `CLICKHOUSE_ALLOW_WRITE_ACCESS` (default false)

### ClickHouse SQL Playground (Try It Out)

Test with the public ClickHouse SQL Playground -- no credentials needed:

```json
{
  "mcpServers": {
    "clickhouse": {
      "command": "uv",
      "args": [
        "run", "--with", "mcp-clickhouse",
        "--python", "3.10", "mcp-clickhouse"
      ],
      "env": {
        "CLICKHOUSE_HOST": "sql-clickhouse.clickhouse.com",
        "CLICKHOUSE_PORT": "8443",
        "CLICKHOUSE_USER": "demo",
        "CLICKHOUSE_PASSWORD": "",
        "CLICKHOUSE_SECURE": "true",
        "CLICKHOUSE_VERIFY": "true"
      }
    }
  }
}
```

## Troubleshooting

### How do I find my serviceId?
**Solution:** Call `get_organizations` (no parameters needed) to get your `organizationId`, then call `get_services_list` with that `organizationId` to see all services with their IDs.

### "Too many parts"
**Cause:** Too many small inserts or high partition cardinality
**Fix:** Increase batch size to 10K-100K rows per INSERT. Check partition cardinality. Enable async inserts for high-frequency small batches.

### "Memory limit exceeded"
**Cause:** Query processing too much data or wrong JOIN algorithm
**Fix:** Add filters to reduce data scanned. Use `partial_merge` or `grace_hash` JOIN algorithm. Add `LIMIT` to restrict output.

### "Query is too slow"
**Cause:** Filters not aligned with ORDER BY or missing indices
**Fix:** Verify filters use ORDER BY prefix with `EXPLAIN indexes = 1 SELECT ...`. Add skipping indices for non-ORDER BY filter columns. Consider materialized views for pre-aggregation.

### "Cannot modify ORDER BY"
**Cause:** ORDER BY is immutable after table creation
**Fix:** Create a new table with the correct ORDER BY, migrate data with `INSERT INTO new_table SELECT * FROM old_table`, then rename tables.

### "Authentication failed" (Cloud MCP)
**Cause:** OAuth session expired or MCP not enabled
**Fix:** Re-authenticate via browser when prompted. Verify MCP is enabled in ClickHouse Cloud console (Connect view). Check that the service is running.

## Tips

1. **Start with service discovery** -- call `get_organizations` then `get_services_list` to find your `serviceId`
2. **Plan ORDER BY first** -- analyze your top query patterns before creating tables
3. **Use the steering file** -- comprehensive best-practices guide with 28 rules
4. **Check cardinality** -- run `SELECT uniq(column) FROM table` before choosing ORDER BY order
5. **Use EXPLAIN** -- `EXPLAIN indexes = 1 SELECT ...` shows whether the primary index is used
6. **Monitor parts** -- query `system.parts` to catch part explosion early
7. **Leverage system tables** -- `system.query_log`, `system.parts`, `system.tables` are your friends
8. **Batch inserts** -- aim for 10K-100K rows per INSERT
9. **Use LowCardinality** -- for string columns with fewer than 10K unique values
10. **Prefer ReplacingMergeTree** -- over ALTER TABLE UPDATE for mutable data

## Legal & Support

### Power
- **License:** [Apache-2.0](https://spdx.org/licenses/Apache-2.0.html)
- **Privacy Policy:** [ClickHouse Privacy Policy](https://clickhouse.com/legal/privacy-policy)
- **Support:** [GitHub Issues](https://github.com/ClickHouse/clickhouse-kiro-power/issues) | [Community Slack](https://clickhouse.com/slack) | [Support Program](https://clickhouse.com/support/program)

### MCP Server -- ClickHouse Cloud Remote (`mcp.clickhouse.cloud`)
- **License:** [Apache-2.0](https://spdx.org/licenses/Apache-2.0.html) ([source](https://github.com/ClickHouse/mcp-clickhouse/blob/main/LICENSE))
- **Privacy Policy:** [ClickHouse Privacy Policy](https://clickhouse.com/legal/privacy-policy)
- **Support:** [GitHub Issues](https://github.com/ClickHouse/mcp-clickhouse/issues)

### MCP Server -- Self-Hosted (`mcp-clickhouse` via pip/uv)
- **License:** [Apache-2.0](https://spdx.org/licenses/Apache-2.0.html) ([source](https://github.com/ClickHouse/mcp-clickhouse/blob/main/LICENSE))

### Best Practices Content
- **Source:** [ClickHouse Agent Skills](https://github.com/ClickHouse/agent-skills)
- **License:** [Apache-2.0](https://spdx.org/licenses/Apache-2.0.html) ([source](https://github.com/ClickHouse/agent-skills/blob/main/LICENSE))
