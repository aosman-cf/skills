# Cloudflare R2 SQL

Serverless, distributed, **read-only** query engine (built on Apache DataFusion) for Apache Iceberg tables in R2 Data Catalog.

Your knowledge of R2 SQL's feature set is likely stale — it has expanded fast. **Prefer retrieval and live testing** over assumptions. In particular, JOINs, subqueries, CTEs, set operations, and window functions are **now supported** (older docs say otherwise).

## What It Is

- **Serverless** — no clusters; query via wrangler CLI, REST API, or from a Worker (HTTP fetch)
- **Distributed** — coordinator distributes work to DataFusion workers across Cloudflare's network
- **Zero egress** — query from anywhere without data-transfer fees
- **Read-only** — no INSERT/UPDATE/DELETE/DDL (use PySpark or PyIceberg to write)

**Status:** Open beta. Pricing announced; **billing not yet enabled** (≥30 days notice).

## Connection Values

| Value | Format |
|-------|--------|
| REST endpoint | `https://api.sql.cloudflarestorage.com/api/v1/accounts/{ACCOUNT_ID}/r2-sql/query/{BUCKET}` |
| Wrangler | `npx wrangler r2 sql query "{WAREHOUSE}" "<SQL>"` with `WRANGLER_R2_SQL_AUTH_TOKEN` set |
| Warehouse | `{ACCOUNT_ID}_{BUCKET}` |

> The REST endpoint is `api.sql.cloudflarestorage.com` — **not** `api.cloudflare.com/.../r2/sql`.

## Quick Start

```bash
npx wrangler r2 bucket catalog enable my-bucket           # 1. enable catalog
export WRANGLER_R2_SQL_AUTH_TOKEN=<r2-token>              # 2. auth (Admin R&W + R2 SQL Read)
npx wrangler r2 sql query "$ACCOUNT_ID"_my-bucket \
  "SELECT * FROM default.my_table LIMIT 10"                # 3. query
```

## What's Supported (verified June 2026)

✅ `SELECT [DISTINCT]`, `WHERE`, `GROUP BY`, `HAVING`, `ORDER BY`, `LIMIT`
✅ **JOINs** (INNER/LEFT/RIGHT/FULL OUTER/CROSS/implicit, multi-way)
✅ **Subqueries** (IN, EXISTS, scalar, derived tables)
✅ **CTEs** (`WITH`, multi-table, with JOINs)
✅ **Set ops** (UNION/UNION ALL, INTERSECT, EXCEPT)
✅ **Window functions** — full set (`ROW_NUMBER`, `RANK`, `CUME_DIST`, `LAG`/`LEAD`, `NTH_VALUE`, running/framed `SUM/AVG OVER`, `ROWS`/`RANGE`/`GROUPS` frames, `QUALIFY`); inline `OVER (...)` only
✅ 33 aggregate + 173+ scalar functions, JSON functions, complex types (struct/array/map), `EXPLAIN [FORMAT JSON]`

❌ `OFFSET`, named `WINDOW` clause, `func(DISTINCT ...)` on aggregates, `ARRAY_AGG`/`STRING_AGG`, `LATERAL`, `UNNEST`/`PIVOT`, INSERT/UPDATE/DELETE/DDL, `SELECT` without `FROM`

See [api.md](api.md) and [gotchas.md](gotchas.md) for the full list and workarounds.

## When to Use

**Use for:** SQL analytics over Iceberg (logs, BI, fraud detection, ad-hoc exploration), multi-cloud queries without egress, dashboards (query from a Worker via HTTP).

**Don't use for:** writes (use PySpark/PyIceberg), real-time OLTP (<100 ms point lookups), windowed analytics needing the named `WINDOW` clause or features below (use PySpark).

## Decision Tree

```
Query structured data in R2?
├─ In Iceberg tables
│  ├─ SQL analytics → R2 SQL (this reference)
│  └─ Python / write-back / window-clause → PyIceberg / PySpark (r2-data-catalog)
├─ Not yet Iceberg
│  ├─ Streaming → Pipelines → Data Catalog → R2 SQL
│  └─ Static files → PyIceberg/PySpark to create tables → R2 SQL
└─ Just objects → plain R2
```

## No Workers Binding

There is no `env.R2_SQL` binding. Query from a Worker via `fetch()` to the REST endpoint with the token as a secret (see [patterns.md](patterns.md#dashboard-worker)).

## Reading Order

1. [configuration.md](configuration.md) — enable catalog, tokens, env setup
2. [api.md](api.md) — full SQL syntax, functions, data types, response format
3. [patterns.md](patterns.md) — CLI/REST/Worker queries, JOINs, windows, use cases
4. [gotchas.md](gotchas.md) — what doesn't work, performance, troubleshooting

## See Also

- [r2-data-catalog](../r2-data-catalog/) — PyIceberg/PySpark, table management
- [pipelines](../pipelines/) — streaming ingest into queryable tables
- [R2 SQL docs](https://developers.cloudflare.com/r2-sql/) · [R2 SQL deep-dive blog](https://blog.cloudflare.com/r2-sql-deep-dive/)
