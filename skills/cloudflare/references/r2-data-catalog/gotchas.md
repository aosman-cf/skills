# R2 Data Catalog Gotchas

Problem → cause → fix. Plus current limits and maintenance behavior.

## Connection / Auth

### Wrong catalog URI or warehouse (most common setup error)

The current formats are:
- Catalog URI: `https://catalog.cloudflarestorage.com/{ACCOUNT_ID}/{BUCKET}`
- Warehouse: `{ACCOUNT_ID}_{BUCKET}`

The old `https://<account-id>.r2.cloudflarestorage.com/iceberg/<bucket>` form and "warehouse = bucket name" are **wrong** and will fail. Always take both values from `wrangler r2 bucket catalog enable`.

### 401 Unauthorized

Token lacks Data Catalog permission. Use a token with R2 Data Catalog R&W. Test with `catalog.list_namespaces()`.

### 403 Forbidden on data files

Token lacks R2 Storage permission. **Admin Read & Write on R2 Storage is required even for read-only data access** during open beta.

### `/config` returns "Warehouse name missing in query param"

The Iceberg `/v1/config` route needs `?warehouse={ACCOUNT_ID}_{BUCKET}`. PyIceberg/PySpark add this automatically when you set `warehouse=`.

## Maintenance Behavior (Updated)

- **No throughput cap on compaction.** The former 2 GB/hour/table limit has been **lifted** — compaction simply triggers hourly and processes the backlog with no hard cap. Large small-file backlogs still take multiple hourly cycles to fully compact.
- **Snapshot expiration now deletes data files** (since April 2026), not just metadata. Expiring a snapshot removes data files no longer referenced by retained snapshots. Manual `remove_orphan_files` is rarely needed.
- **Compaction requires a stored credential.** `wrangler r2 bucket catalog compaction enable` and the dashboard wizard store it automatically; pure-API setups must POST to `/credential`.
- Compaction is **Parquet-only**; target size 64–512 MB (default 128).

## Tables & Schema

| Error | Cause | Fix |
|-------|-------|-----|
| `TableAlreadyExistsError` / `NamespaceAlreadyExistsError` | Exists | Use `create_namespace_if_not_exists` / load existing |
| `NoSuchTableError` | Wrong name or not created | Check namespace+table; create first |
| `422 Validation` on schema update | Incompatible change (required field, type shrink) | Add nullable columns only; widen types (int→long, float→double) |
| `TypeError: Cannot cast` on append | PyArrow type ≠ Iceberg schema | Cast to int64 (Iceberg default); check `table.schema()` |

## Concurrency

- **`CommitFailedException`** — optimistic-locking conflict from simultaneous commits. Add retry with backoff (see [patterns.md](patterns.md#pattern-concurrent-writes-with-retry)).
- **Stale metadata** after external writes — reload: `table = catalog.load_table(("ns","tbl"))`.

## PySpark / Iceberg

| Issue | Fix |
|-------|-----|
| Catalog auth fails | Add header `X-Iceberg-Access-Delegation: vended-credentials` |
| `NoAuthWithAWSException` on orphan removal | Vended creds don't work for orphan removal — supply S3 access/secret keys |
| Version mismatch errors | Use Iceberg `1.6.1` |
| Slow first run (~30–60s) | JAR download; cached afterward |
| Remote signing errors | Set `s3.remote-signing-enabled=false` |

## Nested Namespaces

URL separator for nested namespaces in control-plane API paths is **`%1F`** (Unit Separator), not `/` or `.`: `/namespaces/parent%1Fchild/tables`.

## Query Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| Empty scan results | Wrong filter / partition column | Test `table.scan().to_pandas()` with no filter first |
| Slow scans over time | Too many small files | Enable automatic compaction |

## Current Limits (Open Beta)

| Resource | Recommendation |
|----------|---------------|
| Tables per namespace | <10,000 |
| Files per table | <100,000 (compact regularly) |
| Partitions per table | 100–1,000 optimal |
| `get-table` snapshots returned | Max 10 (use `total_snapshots` for full count) |
| Target file size | 64–512 MB |

## Common Error Codes (Control Plane)

| Code | Cause | Fix |
|------|-------|-----|
| 401 | Missing/invalid token | Token needs catalog + storage permissions |
| 403 | Lacks storage permission | Add Admin R&W on R2 Storage |
| 404 | Catalog not enabled / wrong path | `wrangler r2 bucket catalog enable <bucket>` |
| 409 | Already enabled/exists | Check status first |

## Debug Checklist

1. `npx wrangler r2 bucket catalog status <bucket>` — enabled?
2. Token has R2 Storage (Admin R&W) + R2 Data Catalog (R&W)?
3. `catalog.list_namespaces()` succeeds?
4. Catalog URI = `catalog.cloudflarestorage.com/{ACCOUNT_ID}/{BUCKET}`, warehouse = `{ACCOUNT_ID}_{BUCKET}`?
5. Namespace created before `create_table`?
6. `pip install --upgrade pyiceberg` (recent version)?
7. Compaction enabled + `credential_status: present`?

## See Also

- [configuration.md](configuration.md) · [api.md](api.md) · [patterns.md](patterns.md)
- [R2 Data Catalog docs](https://developers.cloudflare.com/r2/data-catalog/)
