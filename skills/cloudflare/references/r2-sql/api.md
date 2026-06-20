# R2 SQL API Reference

Read-only SQL over Iceberg tables, built on Apache DataFusion. Verified against the live engine, June 2026.

## Query Endpoint

```
POST https://api.sql.cloudflarestorage.com/api/v1/accounts/{ACCOUNT_ID}/r2-sql/query/{BUCKET}
Authorization: Bearer <token>
Content-Type: application/json
Body: {"query": "<SQL>"}
```

CLI: `npx wrangler r2 sql query "{WAREHOUSE}" "<SQL>"` (with `WRANGLER_R2_SQL_AUTH_TOKEN`).

## Response Format

```json
{
  "result": {
    "request_id": "dqe-prod-01...",
    "schema": [
      {"name": "category", "descriptor": {"type": {"name": "string"}, "nullable": false}},
      {"name": "cnt", "descriptor": {"type": {"name": "int64"}, "nullable": false}}
    ],
    "rows": [{"category": "Electronics", "cnt": 12345}],
    "metrics": {"r2_requests_count": 5, "files_scanned": 29, "bytes_scanned": 12345678, "cache_hits": 0}
  },
  "success": true, "errors": []
}
```

Error:
```json
{"result": null, "success": false,
 "errors": [{"code": 40003, "message": "invalid SQL: unsupported feature: OFFSET clause is not supported"}]}
```

## Query Structure

```sql
SELECT [DISTINCT] columns | expressions | aggregations
FROM namespace.table [alias]
[ [INNER|LEFT|RIGHT|FULL OUTER|CROSS] JOIN namespace.table2 alias2 ON ... ]
[WHERE conditions]
[GROUP BY columns]
[HAVING conditions]
[QUALIFY window_predicate]
[ORDER BY expression [ASC|DESC]]
[LIMIT n]                          -- default 500, max 10,000
```

## Schema Discovery

```sql
SHOW DATABASES;            -- list namespaces (aliases: SHOW NAMESPACES / SHOW SCHEMAS)
SHOW TABLES IN namespace;
DESCRIBE namespace.table;  -- columns, types, partition keys
EXPLAIN SELECT ...;        -- execution plan (free; no data scanned)
EXPLAIN FORMAT JSON SELECT ...;
```

## JOINs

```sql
-- All join types; multi-way (3+ tables) supported
SELECT z.domain, h.method, COUNT(*) AS cnt
FROM ns.zones z
INNER JOIN ns.http_requests h ON z.zone_id = h.zone_id
LEFT  JOIN ns.firewall_events f ON z.zone_id = f.zone_id
GROUP BY z.domain, h.method
ORDER BY cnt DESC LIMIT 20;

-- Implicit join
SELECT * FROM ns.a, ns.b WHERE a.id = b.id;
```

## Subqueries & CTEs

```sql
-- IN / EXISTS / scalar
SELECT * FROM ns.t1 WHERE id IN (SELECT id FROM ns.t2 WHERE x > 0);
SELECT * FROM ns.t1 t WHERE EXISTS (SELECT 1 FROM ns.t2 s WHERE s.id = t.id);
SELECT col, (SELECT COUNT(*) FROM ns.t2 s WHERE s.id = t.id) AS cnt FROM ns.t1 t;

-- Derived table
SELECT sub.domain, sub.total FROM (
  SELECT domain, COUNT(*) AS total FROM ns.requests GROUP BY domain
) sub WHERE sub.total > 1000;

-- Multi-table CTE with JOIN
WITH top_zones AS (
  SELECT zone_id, COUNT(*) AS req FROM ns.http_requests GROUP BY zone_id ORDER BY req DESC LIMIT 50
),
threats AS (
  SELECT zone_id, COUNT(*) AS n FROM ns.firewall_events WHERE risk_score > 0.5 GROUP BY zone_id
)
SELECT t.zone_id, t.req, COALESCE(x.n, 0) AS threats
FROM top_zones t LEFT JOIN threats x ON t.zone_id = x.zone_id
ORDER BY t.req DESC LIMIT 20;
```

## Set Operations

```sql
SELECT zone_id FROM ns.firewall_events WHERE action = 'block'
UNION                       -- also UNION ALL, INTERSECT, EXCEPT
SELECT zone_id FROM ns.http_requests WHERE risk_score > 0.8;
```

## Window Functions

Full DataFusion window support, with one exception: use inline `OVER (...)` — the named `WINDOW w AS (...)` clause is **not** supported.

```sql
SELECT event_id,
       ROW_NUMBER() OVER (PARTITION BY mag_type ORDER BY magnitude DESC) AS rn,
       LAG(magnitude, 2, 0.0) OVER (ORDER BY occurred_at) AS prev2,   -- offset + default
       NTH_VALUE(magnitude, 2) OVER (ORDER BY magnitude DESC) AS n2,
       SUM(magnitude) OVER (ORDER BY occurred_at) AS running,
       AVG(magnitude) OVER (ORDER BY magnitude ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg
FROM ns.earthquakes;

-- QUALIFY: filter on a window result (top row per partition)
SELECT event_id, mag_type, magnitude FROM ns.earthquakes
QUALIFY ROW_NUMBER() OVER (PARTITION BY mag_type ORDER BY magnitude DESC) = 1;
```

**Verified working (exhaustive sweep, June 2026):**
- **Ranking:** `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `PERCENT_RANK`, `NTILE`, `CUME_DIST`
- **Navigation:** `LAG`, `LEAD` (both accept offset + default args), `FIRST_VALUE`, `LAST_VALUE`, `NTH_VALUE`
- **Aggregates over windows:** running/partitioned `SUM`, `AVG`, etc.
- **Frames:** `ROWS`, `RANGE` (numeric *and* `INTERVAL` on timestamps), and `GROUPS` — all bound types (`UNBOUNDED PRECEDING`, `n PRECEDING`, `CURRENT ROW`, `n FOLLOWING`, `UNBOUNDED FOLLOWING`)
- **`QUALIFY`** (with or without `PARTITION BY`); window functions inside CTEs; multiple distinct `OVER` specs in one SELECT; window over `GROUP BY` results

**Not supported:** named `WINDOW w AS (...)` clause (inline the `OVER (...)` instead).

## Aggregate Functions (33)

- **Basic (6):** `COUNT(*)`, `COUNT(col)`, `SUM`, `AVG`/`mean`, `MIN`, `MAX`
- **Approximate (5):** `approx_percentile_cont(col, p)`, `approx_percentile_cont_with_weight`, `approx_median`, `approx_distinct` (HyperLogLog — use instead of `COUNT(DISTINCT)`), `approx_top_k(col, k)`
- **Statistical (16):** `var`/`var_samp`, `var_pop`, `stddev`/`stddev_samp`, `stddev_pop`, `covar_samp`, `covar_pop`, `corr`, `regr_slope`, `regr_intercept`, `regr_count`, `regr_r2`, `regr_avgx`, `regr_avgy`, `regr_sxx`, `regr_syy`, `regr_sxy`
- **Bitwise (3):** `bit_and`, `bit_or`, `bit_xor`
- **Boolean (2):** `bool_and`, `bool_or`
- **Positional (2):** `first_value(col ORDER BY ...)`, `last_value(col ORDER BY ...)`

> `func(DISTINCT ...)` (e.g. `COUNT(DISTINCT x)`) is **not** supported as an aggregate — use `approx_distinct(x)`.

## Scalar Functions (173+)

- **Core:** `nullif`, `coalesce`, `nvl`/`ifnull`, `nvl2`, `greatest`, `least`, `named_struct`, `get_field`, `struct`/`row`, `arrow_cast`
- **Datetime:** `now`/`current_timestamp`, `current_date`/`today`, `date_part`/`extract`, `date_trunc`, `date_bin`, `from_unixtime`, `make_date`, `to_char`/`date_format`, `to_date`, `to_timestamp`, `to_timestamp_seconds`/`_millis`/`_micros`/`_nanos`, `to_unixtime`
- **Math:** `abs`, `ceil`, `floor`, `round`, `trunc`, `sqrt`, `power`/`pow`, `exp`, `ln`, `log`, `log2`, `log10`, trig, `degrees`, `radians`, `pi`, `random`, `signum`, `gcd`, `lcm`
- **String:** `concat`, `concat_ws`, `contains`, `starts_with`, `ends_with`, `lower`, `upper`, `trim`/`btrim`/`ltrim`/`rtrim`, `replace`, `split_part`, `repeat`, `to_hex`, `levenshtein`, `ascii`, `chr`, `uuid`
- **Unicode:** `length`/`char_length`, `substr`/`substring`, `left`, `right`, `lpad`, `rpad`, `reverse`, `strpos`/`position`/`instr`, `initcap`, `translate`
- **Regex:** `regexp_count`, `regexp_instr`, `regexp_like`, `regexp_match`, `regexp_replace`
- **Crypto:** `digest`, `md5`, `sha224`, `sha256`, `sha384`, `sha512`
- **Encoding:** `encode`, `decode` (base64)
- **JSON:** `json_get_str(col, key, ...)`, `json_get_int`, `json_get_float`, `json_get_bool`, `json_contains(col, key)`, `json_length(col)` — variadic paths: `json_get_int(doc,'user','profile','level')`
- **Array (46):** `make_array`, `array_length`/`cardinality`, `array_element`, `array_append`, `array_concat`, `array_slice`, `array_has`/`array_has_all`/`array_has_any`, `array_distinct`, `array_sort`, `array_to_string`, `string_to_array`, `flatten`, `range`, `generate_series`, and more
- **Map (4):** `map`, `map_keys`, `map_values`, `map_extract`
  - **BUG:** `map_entries()` fails on stored map columns (error 80001) — use `map_keys`/`map_values`/`map_extract`.

## Expressions

```sql
SELECT amount * 1.1 AS with_tax, amount % 10 AS rem,            -- arithmetic + - * / %
       first || ' ' || last AS full_name,                       -- string concat
       CASE WHEN amount > 1000 THEN 'high' ELSE 'low' END AS t, -- searched CASE
       CASE region WHEN 'N' THEN 'North' ELSE 'Other' END,      -- simple CASE
       CAST(amount AS INT), TRY_CAST(v AS INT), amount::INT,     -- casts
       EXTRACT(YEAR FROM ts) AS yr
FROM ns.t
WHERE status = 200 AND method = 'GET'                            -- =, !=, <>, <, >, <=, >=
  AND ts BETWEEN '2026-01-01T00:00:00Z' AND '2026-02-01T00:00:00Z'
  AND email IS NOT NULL
  AND ua LIKE '%Chrome%'                                         -- LIKE, ILIKE, SIMILAR TO
  AND region IN ('US', 'EU');
```

## Data Types

| Type | Literal | Example |
|------|---------|---------|
| integer | unquoted | `42`, `-10` |
| float | decimal | `3.14` |
| string | single quotes | `'GET'` |
| boolean | keyword | `true`, `false` |
| timestamp | RFC3339 (with timezone) | `'2026-01-01T00:00:00Z'` |
| date | ISO 8601 | `'2026-01-01'` |
| struct | bracket / `get_field` | `col['field']` |
| array | 1-indexed | `col[1]` |
| map | `map_keys`/`map_extract` | `map_extract(col,'k')` |

No implicit conversions: quote strings, include timezone on timestamps, don't quote integers.

## Complex Types

```sql
-- struct
SELECT pricing['price'] AS price, get_field(pricing, 'discount') AS disc FROM ns.t WHERE pricing['price'] > 50;
-- array (1-indexed)
SELECT tags[1] AS first_tag, array_length(tags) AS n, array_to_string(tags, ', ') FROM ns.t;
-- map (NOT map_entries)
SELECT map_keys(meta), map_values(meta), map_extract(meta, 'source') FROM ns.t;
-- struct in GROUP BY / aggregation
SELECT pricing['is_free'] AS free, COUNT(*), AVG(pricing['price']) FROM ns.t GROUP BY pricing['is_free'];
```

## Error Codes

| Code | Meaning |
|------|---------|
| 40003 | Unsupported SQL feature (OFFSET, named WINDOW, etc.) |
| 80001 | `map_entries()` on stored map columns |

## See Also

- [patterns.md](patterns.md) — query examples · [gotchas.md](gotchas.md) — limits & workarounds
- [configuration.md](configuration.md) — auth setup
