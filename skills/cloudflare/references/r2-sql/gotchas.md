# R2 SQL Gotchas

Verified against the live engine, June 2026. **Public docs lag the engine** — JOINs, subqueries, CTEs, set operations, and window functions all work now even where docs say "not supported."

## What Now Works (don't trust stale "unsupported" claims)

| Feature | Added | Notes |
|---------|-------|-------|
| JOINs (INNER/LEFT/RIGHT/FULL OUTER/CROSS/implicit) | May 2026 | Multi-way (3+ tables) too |
| Subqueries (IN, NOT IN, EXISTS, scalar, derived) | May 2026 | Derived tables can be joined |
| Multi-table CTEs | May 2026 | Can include JOINs |
| SELECT DISTINCT | Jun 2026 | All column types |
| UNION / UNION ALL / INTERSECT / EXCEPT | Jun 2026 | |
| **Window functions (OVER)** | verified Jun 2026 | Full set: ranking (`ROW_NUMBER`/`RANK`/`DENSE_RANK`/`PERCENT_RANK`/`NTILE`/`CUME_DIST`), navigation (`LAG`/`LEAD` w/ offset+default, `FIRST_VALUE`/`LAST_VALUE`/`NTH_VALUE`), aggregates over windows, all frame types (`ROWS`/`RANGE` incl. `INTERVAL`/`GROUPS`), `QUALIFY`. Inline `OVER(...)` only — see below. |
| JSON functions | Apr 2026 | `json_get_str/int/float/bool`, `json_contains`, `json_length` |
| EXPLAIN FORMAT JSON | Apr 2026 | Structured plan output |
| Unpartitioned tables | Apr 2026 | OK for <1000 files; partition at scale |

## What Does NOT Work

| Feature | Error / behavior | Workaround |
|---------|------------------|------------|
| `OFFSET` | `40003: OFFSET clause is not supported` | Cursor pagination (WHERE + ORDER BY) |
| Named `WINDOW w AS (...)` clause | `40003: WINDOW clause is not supported` | Inline the `OVER (...)` at each call site (the only window feature missing) |
| `func(DISTINCT ...)` on aggregates | unsupported | `approx_distinct()` for distinct counts |
| `ARRAY_AGG` / `STRING_AGG` | blocked (memory safety) | none in R2 SQL |
| `LATERAL` derived tables | not supported | restructure subqueries |
| `UNNEST` / `PIVOT` / `UNPIVOT` | not supported | flatten at write time |
| `map_entries()` on stored columns | `80001` | `map_keys` / `map_values` / `map_extract` |
| INSERT / UPDATE / DELETE | `only read-only queries` | PySpark / PyIceberg |
| CREATE / DROP / ALTER | `only read-only queries` | PySpark / PyIceberg / wrangler |
| `SELECT` without `FROM` | `query must reference at least one table` | reference a table |

> No Workers binding for R2 SQL. Query the REST endpoint via `fetch()` from a Worker (see [patterns.md](patterns.md#dashboard-worker)), or use D1 / external DB for OLTP.

## Type Safety

```sql
-- ❌ wrong                          -- ✅ right
WHERE status = '200'                 WHERE status = 200
WHERE ts > '2026-01-01'              WHERE ts > '2026-01-01T00:00:00Z'   -- need time + tz
WHERE method = GET                   WHERE method = 'GET'                -- quote strings
```

No implicit conversions. Timestamps must be RFC3339 with timezone; dates ISO 8601.

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| Column not found | Typo / case / wrong table | `DESCRIBE ns.table` |
| Type mismatch | Wrong literal type | Match column type exactly |
| Table not found | Wrong warehouse/namespace | `SHOW DATABASES`; `SHOW TABLES IN ns` |
| LIMIT exceeds maximum | >10,000 | Cursor pagination with partition filters |
| No data (unexpected) | Over-filtering | `SELECT COUNT(*)`, remove filters incrementally, `LIMIT 10` to inspect |
| Token authentication failed | Missing env var | `export WRANGLER_R2_SQL_AUTH_TOKEN=...` |

## Limits & Defaults

- **LIMIT:** default 500, max 10,000.
- **`now()` / `current_time()` quantized to 10 ms** boundaries (security measure, not a bug).
- Wrangler needs `WRANGLER_R2_SQL_AUTH_TOKEN` — it does **not** reuse `wrangler login` OAuth.
- Open beta: R2 Storage **Admin Read & Write required even for read-only** queries.

## Performance

- **File count dominates latency.** 200 small files ≈ 4–9 s/query; 10 compacted ≈ 1–3 s. Enable automatic compaction.
- **Partition + filter:** put `__ingest_ts` (or your partition key) range first in `WHERE`, narrow time windows, add predicates.
- **Multi-way JOINs on large tables** can exceed resource limits — filter heavily, join through dimension tables, avoid cross-joining large fact tables.
- **Always `LIMIT`** for early termination.
- Per-query `metrics` (`files_scanned`, `bytes_scanned`, `cache_hits`) are the primary observability signal — there is no dedicated R2 SQL GraphQL dataset. `bytes_scanned` ≈ billable data.

```sql
-- ❌ slow                                   -- ✅ fast
SELECT * FROM logs.requests LIMIT 10000;     SELECT * FROM logs.requests
                                             WHERE __ingest_ts >= '2026-01-15T00:00:00Z'
                                               AND __ingest_ts <  '2026-01-16T00:00:00Z'
                                               AND status = 404
                                             LIMIT 1000;
```

## Debug Checklist

1. `wrangler r2 bucket catalog enable <bucket>` — catalog on?
2. `echo $WRANGLER_R2_SQL_AUTH_TOKEN` — token set?
3. `SHOW DATABASES` → `SHOW TABLES IN ns` → `DESCRIBE ns.table`
4. `SELECT COUNT(*) FROM ns.table` — data present?
5. `SELECT * FROM ns.table LIMIT 10` — inspect types
6. Add filters incrementally; read `metrics` to tune

## See Also

- [api.md](api.md) — syntax & functions · [patterns.md](patterns.md) — examples
- [configuration.md](configuration.md) — setup
- [R2 SQL docs](https://developers.cloudflare.com/r2-sql/) (note: may lag the engine)
