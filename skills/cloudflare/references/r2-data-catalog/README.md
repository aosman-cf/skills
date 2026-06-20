# Cloudflare R2 Data Catalog

Managed Apache Iceberg REST catalog built directly into R2 buckets. No catalog servers to run.

Your knowledge of catalog URIs, limits, and maintenance behavior may be stale. **Prefer retrieval** — verify against [R2 Data Catalog docs](https://developers.cloudflare.com/r2/data-catalog/) and the live API before citing specific numbers or endpoints.

## What It Provides

- **Apache Iceberg tables** — ACID transactions, schema evolution, time-travel
- **Zero-egress reads** — query from any cloud/region, no data transfer fees
- **Standard Iceberg REST API** — works with PyIceberg, PySpark, Snowflake, Trino, DuckDB
- **Automatic maintenance** — managed compaction and snapshot expiration
- **Control-plane REST API** — manage catalogs, namespaces, tables, maintenance config

**Status:** Open beta. Pricing announced; **billing not yet enabled** (Cloudflare gives ≥30 days notice). Available to all R2 subscribers.

## Connection Values (Verified Formats)

These are the exact formats R2 Data Catalog uses today. The older `https://<account-id>.r2.cloudflarestorage.com/iceberg/<bucket>` form is **wrong** — do not use it.

| Value | Format | Example |
|-------|--------|---------|
| Catalog URI | `https://catalog.cloudflarestorage.com/{ACCOUNT_ID}/{BUCKET}` | `https://catalog.cloudflarestorage.com/4482a1.../live-data` |
| Warehouse | `{ACCOUNT_ID}_{BUCKET}` (hyphens in bucket preserved) | `4482a1..._live-data` |
| Token | R2 API token (Admin R&W on Storage + R&W on Data Catalog) | `cfut_...` |

Get these from `npx wrangler r2 bucket catalog enable <bucket>`. The Iceberg `/config` route requires a `?warehouse={WAREHOUSE}` query param.

## Architecture

```
Query/Compute engines (PyIceberg, PySpark, Trino, Snowflake, DuckDB, R2 SQL)
        │  Iceberg REST API (Bearer token)
        ▼
R2 Data Catalog  ── namespace/table metadata, snapshots, transaction coordination
        │  vended S3 credentials
        ▼
R2 Bucket  ── Parquet data files + Iceberg metadata/manifest files
```

**Key concepts:**
- **Warehouse** — top-level grouping for the catalog (`{ACCOUNT_ID}_{BUCKET}`)
- **Namespace** — schema/database containing tables (e.g. `logs`, `analytics`); nested namespaces supported
- **Table** — Iceberg table with schema, partition spec, snapshots
- **Vended credentials** — temporary S3 creds the catalog hands engines for data access (`X-Iceberg-Access-Delegation: vended-credentials`)

## When to Use

**Use for:** log/analytics data lakes, BI pipelines, time-series/event data, multi-cloud or multi-engine analytics, anything needing ACID + schema evolution on object storage.

**Don't use for:** transactional/OLTP workloads (use D1 or a database), sub-second point lookups, tiny datasets (<1 GB) where setup overhead isn't justified, or unstructured blobs (store directly in R2).

## Two APIs, Don't Confuse Them

| API | Base | Use for |
|-----|------|---------|
| **Iceberg REST catalog** | `https://catalog.cloudflarestorage.com/{ACCOUNT_ID}/{BUCKET}` | Table reads/writes via PyIceberg, PySpark, Trino, etc. |
| **Control-plane REST API** | `https://api.cloudflare.com/client/v4/accounts/{ACCOUNT_ID}/r2-catalog/{BUCKET}` | Enable/disable catalog, manage maintenance config, list namespaces/tables, get-table metadata |

## Reading Order

1. [configuration.md](configuration.md) — enable catalog, tokens, compaction/snapshot expiration, client connection
2. [api.md](api.md) — control-plane REST API (incl. new get-table), PyIceberg client API, maintenance
3. [patterns.md](patterns.md) — PyIceberg + PySpark examples, partitioning, time-travel, external engines
4. [gotchas.md](gotchas.md) — auth errors, maintenance behavior, limits, troubleshooting

## See Also

- [pipelines](../pipelines/) — stream events into Iceberg tables
- [r2-sql](../r2-sql/) — serverless SQL over these tables
- [r2](../r2/) — underlying object storage
- [R2 Data Catalog docs](https://developers.cloudflare.com/r2/data-catalog/) · [Apache Iceberg](https://iceberg.apache.org/) · [PyIceberg](https://py.iceberg.apache.org/)
